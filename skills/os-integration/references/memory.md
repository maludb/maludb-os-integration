# Memory — how an application stores what the kernel expects to find

Three stores, deliberately kept apart. The application writes the first two itself; the third
belongs to the kernel, and the application only ever *reads* it through the kernel.

| Memory | Where | Written by | Read by |
|---|---|---|---|
| **Record memory** | The application's own PostgreSQL 17 database, base tables | The application's PHP, as its read-write role | Its `mcp_*` views, through a read-only role, by its records MCP server |
| **Activity memory** | The application's `activity_log` (append-only), shipped to the tenant's MaluDB as `activity` episodes | `log_activity()` only | Its `mcp_activity_*` views (activity MCP server); MaluDB recall |
| **Agent memory + skills** | The tenant's MaluDB memory database (`<tenant>_memory`) | The kernel's PHP handlers and its agent runner | The kernel's Memory MCP server (8814) |

An application never opens a connection to the kernel's database, and the kernel never
opens one to the application's. Everything crosses as MCP calls, API calls, or MaluDB episodes.

## 1. Record memory

- One PostgreSQL 17 database per application per tenant. Schema in numbered, additive migrations
  (`db/NNN_*.sql`), run in order as `postgres`.
- **Three roles**, created by the first migration: `<app>_rw` (PHP — full DML, the only writer),
  `<app>_records_ro` (records MCP) and `<app>_activity_ro` (activity MCP). The two read roles get
  `USAGE` on `public` and **no grant on any base table** — only on the views below. `BYPASSRLS` is
  never granted.
- **Session context.** Every connection carries the acting member:
  ```sql
  SELECT set_config('app.member_id', '42', false);   -- PHP: connection-scoped, from the session
  SELECT set_config('app.member_id', '42', true);    -- MCP: transaction-scoped, per query, from the verified token
  ```
  with `app_current_member_id()` = `NULLIF(current_setting('app.member_id', true), '')::bigint`.
  Unset means anonymous and every rule denies. The application looks up role, departments and grants
  from that one id — never trusts a role passed in.
- **Visibility views.** Every table an agent may read gets a `mcp_<table>` view declared
  `WITH (security_barrier = true)` whose `WHERE` clause *is* the row-level rule, calling one
  visibility function (the kernel's is `app_can_see(module, owner_member_id, department_id,
  entity_type, entity_id, organization_id)` — owner, super-admin, dept-admin of that department,
  else module grant + department membership). The primary key is re-aliased `<entity>_id`; raw
  columns that hide secrets or internals are simply not selected. `GRANT SELECT` on the view to the
  read role, and nothing else. Append columns last when altering a view; re-check grants afterwards.
- **Never serialize a base row** to an agent or a screen: reads go through the view or a
  whitelist presenter.

## 2. Activity memory — the one funnel

Activity memory cannot be backfilled, so logging exists from the first deploy.

### The table (copy it)

```sql
CREATE TABLE activity_log (
    id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    occurred_at     timestamptz NOT NULL DEFAULT now(),
    actor_member_id bigint REFERENCES members(id) ON DELETE SET NULL,
    source          text NOT NULL DEFAULT 'web'
                    CHECK (source IN ('web','assistant','mcp','cron','agent','desk','webhook','portal','api')),
    action          text NOT NULL,           -- entity.verb
    screen          text, route text,        -- route = "METHOD /path"
    entity_type     text, entity_id bigint,  -- singular table name + its PK; both null for non-record events
    before          jsonb, after jsonb,      -- the changed fields only
    request_id      text, session_id text, ip_address inet,
    agent_run_id    bigint,                  -- the kernel's run id when an agent acted (no FK)
    department_id   bigint, location_id bigint,
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON activity_log (actor_member_id, occurred_at DESC);
CREATE INDEX ON activity_log (entity_type, entity_id, occurred_at DESC);
CREATE INDEX ON activity_log (action, occurred_at DESC);
CREATE INDEX ON activity_log (occurred_at DESC);
CREATE INDEX ON activity_log (request_id);
GRANT INSERT, SELECT ON activity_log TO <app>_rw;
REVOKE ALL ON activity_log FROM <app>_records_ro, <app>_activity_ro;
```

### The function (one signature, everywhere)

```php
function log_activity(PDO $pdo, string $action, ?string $entityType = null,
                      ?int $entityId = null, array $opts = []): void
// $opts: actor_member_id, source, screen, route, before, after, request_id, agent_run_id
function log_screen_view(PDO $pdo, string $screen): void   // action 'screen.view'
```

Rules the kernel relies on:

- **`action` is `entity.verb`**, lower snake_case, dot-separated: `booking.save`, `booking.cancel`,
  `ticket.set_status`, `auth.login_failed`, `cron.run`, `api.bookings.read` (three parts for API
  reads). The dot is structural — approval policies match `'refund.*'` and `'*.delete'`.
- **`source`**: the UI is `'web'` (there is no `'ui'`); a token API read is `'api'`; an action
  taken by a kernel agent through an action token is `'agent'` when the token carries a run id,
  `'assistant'` otherwise; `'mcp'` for the actions MCP server; `'cron'` under CLI. Default it from
  the request, never from a parameter an outside caller controls.
- **`before` / `after`** are the changed fields only. Money actions must write `after.amount` and
  `after.currency`. Anything sensitive (a memory text, a credential, a document body) goes in as a
  **200-character excerpt at most** — never the value.
- **`request_id`**: honour an inbound `X-Request-Id`; when an agent's action token carries a run id,
  use the run's own request id for every row of that run, so the kernel's prompt ledger links to
  the activity the call produced. Otherwise 8 random bytes, hex.
- `log_screen_view()` on a JSON request logs **only** when the header `X-Screen-View: 1` is present
  (a real render, not a prefetch).
- `log_activity()` never throws — a failure goes to `error_log`, the request completes.
- Every state-changing handler: `require_post()` + `verify_csrf()` (or the action token) +
  authorization + `log_activity()`. A write without a log row is a bug.

### Shipping to MaluDB

A bridge (the kernel's is `mcp/activity_ingest.py`, run by a systemd timer every minute as the
web user) ships new rows as episodes:

1. `SELECT pg_try_advisory_lock(hashtext('activity_ingest'))` — not held ⇒ exit 0.
2. Read the singleton checkpoint:
   ```sql
   CREATE TABLE activity_ingest_state (
       id smallint PRIMARY KEY DEFAULT 1 CHECK (id = 1),
       last_id bigint NOT NULL DEFAULT 0,
       updated_at timestamptz NOT NULL DEFAULT now());
   INSERT INTO activity_ingest_state (id, last_id) VALUES (1, 0) ON CONFLICT DO NOTHING;
   GRANT SELECT, UPDATE ON activity_ingest_state TO <app>_rw;
   ```
3. `SELECT … FROM activity_log WHERE id > $last ORDER BY id LIMIT 500`, and for each row
   `POST {MALUDB_API_URL}/v1/episodes` with `Authorization: Bearer {MALUDB_API_TOKEN}`, 30 s
   timeout, **one row per request**:
   ```json
   {
     "kind": "activity",
     "title": "booking.save by member #12",
     "summary": null,
     "occurred_at": "2026-09-21T14:03:11+00:00",
     "sensitivity": "internal",
     "provenance": "provided",
     "payload": {
       "activity_log_id": 123, "actor_member_id": 12, "source": "web",
       "action": "booking.save", "screen": "booking-edit", "route": "POST /bookings/save.php",
       "entity_type": "booking", "entity_id": 45,
       "before": {"status": "held"}, "after": {"status": "confirmed"},
       "request_id": "…", "session_id": "…"
     }
   }
   ```
   Null-valued payload keys are dropped. `title` says "by system" when there is no actor.
4. Accept 200 or 201; on anything else stop the loop (do not advance past a rejected row).
5. `UPDATE activity_ingest_state SET last_id = GREATEST(last_id, $id)` — the checkpoint only moves
   forward; a duplicate episode is preferable to a lost one.

**Add one payload key the kernel's bridge does not send: `"application": "<catalog_key>"`** — the
tenant's memory holds episodes from every application, and recall needs to know which one an
episode came from. Put it first in `payload`.

### Environment

`MALUDB_API_URL` and `MALUDB_API_TOKEN`, in the application's `config/.env` (gitignored; the web
group keeps read access). Missing ⇒ the bridge is a clean no-op (exit 0), never a crash. The token
is minted once, out of band, by proving the tenant's Postgres login (`POST /v1/tokens`); the
installation agent will do that at install and write the two keys. **Every application on the
tenant's server shares the tenant's one memory database** — do not create a memory database per
application.

## 3. Agent memory and skills — read-only from the application's side

The kernel owns agent memory. Its namespaces are `member:<id>`, `agent:<id>`, `dept:<id>` and
`org`; reads are through the kernel's Memory MCP server (`recall`, `core_memory`,
`session_search`); writes are kernel PHP actions that are gated, approved and logged
(`memory_remember`, `core_memory_set`). An application does **not** call
`/v1/memory/remember`, does **not** write principal profiles, and does **not** ingest skills. What
it contributes is:

- **Episodes** (§2) — that is how the application's history becomes recallable.
- **Skills it ships** (`skills/<name>/SKILL.md` + reference files in its repository) — see
  `registration.md`: the installation agent ingests them and assigns them at application scope, so
  the application's expert and every agent allowed to use it get them. Bundle rules: a folder with
  `SKILL.md` whose frontmatter has `name:` (`/^[a-z0-9][a-z0-9\-]{1,63}$/`) and a mandatory
  `description:`; at most 20 files and 256 KB; text files only (`md markdown txt json yaml yml csv`);
  no URLs that look like instructions, no shell-looking lines, no "ignore previous" phrasing — the
  scanner records those as findings for a reviewer, and refuses executables outright.

## Checklist

- [ ] Own database, three roles, read roles with no base-table grants.
- [ ] `app.member_id` set on every connection; one visibility function; every agent-readable table has a `security_barrier` `mcp_*` view.
- [ ] `activity_log` with the columns and `source` enum above; `log_activity()` with the signature above; `entity.verb` names; excerpts not values.
- [ ] Every write handler logs; screen views logged behind `X-Screen-View: 1`.
- [ ] The ingest bridge with its checkpoint, advisory lock, one episode per row, `application` in the payload, and a systemd timer.
- [ ] `MALUDB_API_URL` / `MALUDB_API_TOKEN` read from `config/.env`, no-op when absent.
- [ ] Skills the application ships live in `skills/` and pass the bundle rules.
