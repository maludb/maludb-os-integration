# MCP servers and APIs — how an application must cover its functionality

The platform's rule: **a module is not done until an agent can query it and act on it.** For an
application from us that means two read MCP servers it runs itself, an action manifest the
platform's actions server executes, and, when an outside system needs it, a token API.
Everything an agent can do in the application goes through these — never a database connection,
never a scraped screen.

| Surface | Purpose | Backing | Who runs it | Reachable |
|---|---|---|---|---|
| `<app>_records_mcp` | Record questions | `mcp_*` views, role `<app>_records_ro` | The application | Under the application's name, `/mcp/records` |
| `<app>_activity_mcp` | Activity questions | `mcp_activity_*` views, role `<app>_activity_ro` | The application | Under the application's name, `/mcp/activity` |
| Action tools | Every action a button can do | The application's **action manifest → registry**; the tools POST to its PHP handlers on the internal port | **The platform's actions server** (one per tenant, localhost only) | Never proxied |

Why the application does not ship an actions server: the platform's one enforces tool grants at
the MCP boundary, refuses writes under an evaluation run, holds the relay key the agent must not
have, and turns a handler's `location` into `record_id`. Those are controls, and a control that
each application re-implements is not one. The application's job is to make its writes
*executable* by that server (§3) and *safe* when they arrive (§3, PHP side).

Ports are **configuration, not constants**: the platform owns 8811–8816, 8765 and 8080. Read
`MCP_RECORDS_PORT`, `MCP_ACTIVITY_PORT` and `APP_INTERNAL_PORT` from `config/.env`; the
installation agent assigns them per application and writes them into the registry
(`registration.md`).

## 1. The shape every server shares

Python 3, the MCP SDK's `FastMCP` (`mcp>=1.2,<2`), **streamable HTTP** at `/mcp`, bound to
`127.0.0.1` with uvicorn, one systemd unit per server (`User=www-data`,
`WorkingDirectory=<app>/mcp`, `ExecStart=<app>/mcp/venv/bin/python …`, `NoNewPrivileges`,
`PrivateTmp`, `ProtectSystem=full`). The SDK's DNS-rebinding guard is on because the host is
loopback, so Apache must proxy with `ProxyPreserveHost Off` for `/mcp/*`.

Two files carry the whole pattern and are reused verbatim by every server (copy them from the
platform's `mcp/db.py` and `mcp/server_common.py`, about 260 lines together):

- `db.py` — env loader for `config/.env`; one cached asyncpg pool per role (`min_size=1,
  max_size=8, command_timeout=15`) with json/jsonb codecs registered so tools never return
  JSON-inside-a-string; `hash_token`, `resolve_token`, `verify_action_token`, `to_json`,
  `fetch_scoped`, `run_search`.
- `server_common.py` — a raw ASGI middleware that reads `Authorization: Bearer …`, resolves it,
  answers `401 {"error":"unauthorized: provide a valid MCP access token as a Bearer token"}` with
  `WWW-Authenticate: Bearer` on failure, and on success sets the contextvars
  `request_member_id`, `request_role`, `request_run_id`; then `uvicorn.run(app, host="127.0.0.1", port=…)`.

```python
if __name__ == "__main__":
    import server_common
    server_common.run(mcp, "MCP_RECORDS_DB_USER", "MCP_RECORDS_DB_PASSWORD",
                      port=int(os.environ["MCP_RECORDS_PORT"]), endpoint_name="Records MCP")
```

### Identity on every query

```python
async def fetch_scoped(pool, sql, *params, member_id=None, role=None) -> list[dict]:
    async with con.transaction():
        await con.execute(
            "SELECT set_config('app.member_id', $1, true), set_config('app.role', $2, true)",
            "" if member_id is None else str(member_id), role or "anon")
        rows = await con.fetch(sql, *params)
```

Transaction-local, so a pooled connection never leaks an identity. The views do the scoping; the
server never filters rows in Python by who is asking.

### Who may call — two token shapes, one key

1. **The application's own bearer tokens** — `mcp_access_tokens(id, member_id, label,
   token_hash sha256 UNIQUE, scope CHECK ('mcp','api'), last_used_at, expires_at, revoked_at)`.
   Raw form `mcp_` + 48 hex chars, shown once, only the hash stored. Resolved in SQL by a
   `SECURITY DEFINER` function `mcp_resolve_token(p_hash) RETURNS TABLE(member_id, member_role)`
   (stamps `last_used_at`; requires not revoked, not expired, member active) — the read roles
   have no grant on the table, only `EXECUTE` on this function. Minted by
   `php bin/mint_mcp_token.php --email … --label … [--scope mcp|api]`. This is how a person's
   Claude Desktop or Claude Code connects.
2. **The tenant's signed action token** — what the platform's assistant and agents present.
   Two forms, HMAC-SHA256 over the payload with the tenant's `ACTION_TOKEN_KEY`:
   - `{member_id}.{expires}.{hmac}` signed over `"mid.exp"` — acts for the person at the keyboard (TTL 600 s).
   - `{member_id}.{expires}.{run_id}.{hmac}` signed over `"run:mid.exp.run"` — an **agent run
     token**, minted only by the platform's agent runner, alive for the run.

   **The integration contract: every application from us on a tenant's server verifies with the
   tenant's `ACTION_TOKEN_KEY` and `ACTIONS_RELAY_KEY`** — the installation agent writes the same
   two keys into each application's `config/.env`. That is what lets a platform agent present the
   token it already holds to any application, with its own member id and run id intact, and
   nobody minting per-application credentials. Member ids are the platform's (see §5).

   `resolve_token()` tries the hash first, then the signed shape. A run token is what a
   platform agent presents to the read servers (as Bearer) and what arrives at the PHP handlers
   (as `X-Action-Token`, relayed by the platform's actions server).

   **Grants, fail closed.** The platform decides which tools an agent holds on an endpoint
   (`agent_tool_grants`); a person's token is never filtered. The application's read servers
   install the same `agent_grants` hook as the platform's, but they cannot query the platform's
   database. Until the platform exposes a run-facts call (`agents.md`, "What the platform still
   owes"), **a run token on an application's read server lists and calls no tools** — a person's
   token works, an agent's is refused with the platform's own sentence: *"'x' is not among the
   tools this agent was granted on <endpoint>."* Never fail open here.

### What a tool returns

A **JSON string** (`-> str`), serialised by `db.to_json` (date/datetime → ISO 8601, Decimal →
float, timedelta → days). Rows come from `mcp_*` views only. Composite answers are a dict of
sub-results: `{"application": […], "endpoints": […]}`. Every list tool takes
`limit: int = Field(25, ge=1, le=100)` as its last positional SQL argument; detail tools cap
children (`interactions_limit`); unparameterised lists hard-cap at 200.

## 2. Designing the tool surface (before any code)

Derive it from the application's **question inventory** — every recurring question a screen
answers, written down with its gate. Then:

- **One named tool per recurring question.** `find_<things>` (search/list), `get_<thing>` (one
  record with its children), `my_<things>` (the caller's own), plain nouns for reports
  (`pipeline`, `occupancy`, `revenue_by_period`). snake_case; descriptions written for an LLM
  with **positive and negative triggers**: *"Call for 'who is booked Tuesday' … Not for prices
  (see `find_services`)."*
- **One guarded search tool per read server** for the long tail — `records_search` /
  `activity_search`: a single `SELECT`/`WITH`, no `;`, keyword blacklist, `SET LOCAL
  statement_timeout='5s'`, wrapped as `SELECT * FROM (…) _q LIMIT 200`, run under the read role
  with `app.member_id` set. Safe by construction because the role sees only the views. Its
  docstring **enumerates every readable view by name**.
- **Gates** use the platform's vocabulary and are enforced in SQL, not Python: `insider` (anyone
  who works here — excludes external members), `mod:<grant>`, `admin` (super-admin, or dept-admin
  within their departments), `super`. The view's `WHERE` carries it (`WHERE app_is_insider()`,
  `app_has_module('bookings') OR app_is_super_admin()`). A tool only trims what the view already
  allows.
- **Every action a button can do gets an action tool** (§3) — no write path exists that an agent
  cannot reach, and none that bypasses the handler's own authorization.
- **Read-only annotations** on the two read servers: `{"readOnlyHint": True, "openWorldHint": False}`.

Module layout: one `mcp/business_<slice>.py` per slice exposing
`def register(mcp, q) -> None` where `q(sql, *args)` runs a member-scoped read; Pydantic input
models at module scope (`ConfigDict(str_strip_whitespace=True, extra="forbid")`, a `Field(...,
description=…)` on every field); the server file only imports and registers. A module never
opens its own connection. Positional `$n` placeholders built as `args.append(v);
where.append(f"col = ${len(args)}")`.

## 3. Actions — the manifest is the source of truth

Writes never happen in Python. The platform's actions server turns a **markdown action manifest**
into tools that POST to the application's own PHP handlers, form-encoded, on the internal port.
The application ships the manifest, the builder and the registry it produces; the installation
agent hands the registry (with the application's base URL) to the platform's actions server,
which loads every installed application's registry beside its own.

**Manifest** (`docs/<app>-action-manifest.md`), one table per section, eight columns:

```
| Action | File | Params | Undo | Confirm | Agent approval | Log | Who |
| `booking_create` | `/bookings/save.php` | **contact**, **starts_at**, service, notes | cancel it | | | `booking.create` | mod:bookings |
| `booking_update` | `/bookings/save.php` | **booking**, any field of booking_create | restore prior | | | `booking.update` | mod:bookings |
| `booking_cancel` | `/bookings/cancel.php` | **booking**, reason | reinstate | ✔ | external send | `booking.cancel` | mod:bookings |
```

Bold = required, `[]` = repeated, parentheses = a hint for the description (no comma or semicolon
inside them, and no note after the last parameter — the builder would read either as a phantom
parameter), `✔` in Confirm = destructive (the tool gains a `confirmed: bool` field). "any field of
X" expands to X's fields, all optional, and marks the action **partial** (§4). Screens get their
own table so `find_screen` and `navigate` work.

**Builder**: a `bin/build_action_registry.php` that parses the manifest into
`mcp/action_registry.json`; `built` is literally `is_file(webroot . endpoint)`, so an unbuilt
action is findable but never callable; `--check` exits non-zero when the registry is stale.

**What the platform's tool factory does with it** (so the manifest is written to fit): for every
built action it builds the input model, registers with `readOnlyHint: False` and `destructiveHint
= confirm`, and writes the description from the manifest's own words — the log event, *"Allowed
for: {who} — the endpoint enforces it"*, the confirm instruction, the approval category, the undo
sentence, and *"After success, say what you did in ONE short sentence and END THE TURN."* Entity
parameters resolve through a table of `(view, id column, label column)`: digits → id lookup,
otherwise `ILIKE … LIMIT 6`; several matches return the candidates so the assistant asks one
question. **The application's registry must therefore name, for each entity parameter, the
`mcp_*` view and columns that resolve it** — the platform cannot reach the application's
database, so the actions server resolves entities by calling the application's own records MCP
`find_*` tool named in the registry (`"resolve": {"tool": "find_contacts", "id": "contact_id",
"label": "display_name"}`).

**The POST** (`app_post`): `http://127.0.0.1:{APP_INTERNAL_PORT}{path}`, `Accept:
application/json`, `X-Action-Token: <token>`, and for a run token `X-Action-Relay:
hmac(ACTIONS_RELAY_KEY, token)` — the agent holds the token, never the relay key, so it cannot
POST to a handler itself and step around its grants. Every application handler verifies both
with the tenant's shared keys. Answer mapping:

```python
if body.get("ok") is True:
    out = {k: v for k, v in body.items() if k != "ok"}
    found = re.search(r"/(\d+)/?(?:[?#].*)?$", str(body.get("location", "")))
    if found: out["record_id"] = int(found.group(1))      # every create returns record_id
    return {"status": "success", **out}
if body.get("status") == "pending_approval": return {"status": "pending_approval", **body}
return {"status": "error", "http_status": r.status_code, "message": …, "errors": […]}
```

### PHP side — JSON mode, so handlers are not rewritten

- A valid action token **acts as that member for one request only** (`header_remove('Set-Cookie')`,
  session destroyed at shutdown), sets `app.member_id`, and **stands in for CSRF**.
- `json_mode_begin()` at bootstrap when `Accept: application/json` and not HTMX; `json_mode_finish()`
  at shutdown answers from what the handler reported: **200** `{ok:true, …data, location?}`;
  **202** `{ok:false, status:'pending_approval', message, approval_request_id}`; **422**
  `{error:{code:'invalid', message, errors, fields}}`; 4xx mapped (`400 bad_request … 429
  rate_limited`); ≥500 `server_error` with generic words; a handler that reported nothing → **501
  not_converted**.
- Every write handler reports: `emit_action_status(true, ['id' => $id, 'location' => "/bookings/$id"])`
  on success, `emit_action_status(false, ['errors' => $errors]); respond_invalid($errors);` on
  refusal. The first failure stands. **`location` must end in the new record's id** — that is
  where `record_id` comes from.
- Every write handler still does `require_post()` + `verify_csrf()` (satisfied by the token) +
  the gate mirror (`require_module_grant('bookings')`, `require_insider()`, …) + `log_activity()`.
  The endpoint enforces "Who"; the tool description only states it.

## 4. Partial updates

An agent's `*_update` sends the record and what changes. When the token is an action token and
`_partial=1` is posted, `partial_update_prefill()` checks visibility through the `mcp_*` view,
reads the row from the **base table** (a view hides columns), and fills only the absent `$_POST`
keys — booleans as `'1'/'0'` (not sent ≠ false), jsonb addresses expanded to `address_*` fields,
arrays as pg literals. A table `PARTIAL_UPDATE_TARGETS = [endpoint => [base table, id field,
view, view id column]]` names what may be prefilled. Only under an action token, never for a form.

## 5. Identity: the platform's member ids, everywhere

The platform signs people in to an application from us, so the application does **not** own
identities. Its `members` table mirrors the platform's: **the same `id` values**, `member_kind`
(`human`|`agent`), `business_role`, `is_external`, `status`, plus department membership, kept in
step at sign-on and by the installation agent. Consequences that must hold:

- `app.member_id` means the same person in every application and in the platform.
- MaluDB namespaces (`member:<id>`, `agent:<id>`, `dept:<id>`) line up across applications.
- An agent's run token names a member the application already knows; an unknown id is refused,
  not auto-created.
- The application keeps **no password**; the sign-on hand-off (a signed token on the action
  token's pattern is the recommendation — the mechanism is still open on the platform side) is
  the only way in besides a bearer token.

## 6. A token API for things that are not MCP clients

When an outside system (a website, a partner) must read the application, add `GET`-only
`/api/v1/*.php` endpoints on the pattern:

```php
require_once dirname(__DIR__, 3) . '/app/api/bootstrap.php';
api_cors(); api_require_get();
$member = api_authenticate();               // session first, then Bearer with scope='api'
$data   = booking_feed(db(), (int) $member['id']);
log_activity(db(), 'api.bookings.read', 'member', (int) $member['id'], ['source' => 'api']);
api_json($data);
```

Rules: one 401 body for every failure (`{"error":{"code":"unauthorized","message":"A valid API
token is required."}}` — no enumeration); CORS from an allow-list with **no
`Allow-Credentials`**; `api` and `mcp` token scopes are not interchangeable; additive-only
within a major version, and a feed document carries its own schema id (`"<app>.bookings/1"`).
Each public endpoint is named in the vhost's allow-list; everything else on the public name is
the application's UI.

## 7. Apache, under the application's own name

```
# reservations.subello.com — port assigned by the installation agent (name-based when there is no proxy in front)
<VirtualHost *:81>
    DocumentRoot /srv/apps/reservations/html
    ProxyPreserveHost Off
    ProxyPass        /mcp/records  http://127.0.0.1:${MCP_RECORDS_PORT}/mcp
    ProxyPassReverse /mcp/records  http://127.0.0.1:${MCP_RECORDS_PORT}/mcp
    ProxyPass        /mcp/activity http://127.0.0.1:${MCP_ACTIVITY_PORT}/mcp
    ProxyPassReverse /mcp/activity http://127.0.0.1:${MCP_ACTIVITY_PORT}/mcp
</VirtualHost>
# PHP on the internal port, for the platform's actions server and approval replays only
<VirtualHost 127.0.0.1:${APP_INTERNAL_PORT}>
    DocumentRoot /srv/apps/reservations/html
</VirtualHost>
```

The platform's registry knows the two proxied servers by URL under the application's name
(`registration.md`); the internal port is what its actions server posts to.

## Known gaps to design around (true of the platform today)

- No `offset`/`truncated` on list tools — raise `limit` or filter harder.
- MCP tool calls are not themselves logged to `activity_log` (the token's `last_used_at` is the
  only trace). Log yours if the question inventory needs "what did the agent ask".
- No rate limiting on MCP or API — add it before an endpoint is reachable beyond a known origin.
- The platform's actions server loads only its own registry today; loading an application's
  registry with a base URL, and resolving entities through the application's `find_*` tools,
  are platform changes owed under phase 7 (`agents.md`). Ship the manifest and registry now so
  nothing waits on the application when they land.

## Checklist

- [ ] Question inventory with gates → named tools + one guarded search per read server.
- [ ] `db.py` + `server_common.py` copied; ports from `.env`; loopback only; systemd units.
- [ ] `mcp_access_tokens` + `mcp_resolve_token()`; `ACTION_TOKEN_KEY`/`ACTIONS_RELAY_KEY` from the tenant; run tokens fail closed on tools.
- [ ] Action manifest → builder → `mcp/action_registry.json` with entity `resolve` entries; `record_id` on every create; partial updates.
- [ ] Handlers report through `emit_action_status()`; JSON mode; token + relay verified; token replaces CSRF; every write logged.
- [ ] Members mirror the platform's ids; no passwords.
- [ ] Apache: `/mcp/records` and `/mcp/activity` proxied with `ProxyPreserveHost Off`; PHP on a loopback internal port.
