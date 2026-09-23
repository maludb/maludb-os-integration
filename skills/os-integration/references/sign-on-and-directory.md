# Sign-on and the directory — how a person gets in, and how the application knows who exists

Design: *Business OS — Kernel and Integration Design* (2026-09-22; repo copy
`docs/business-os-integration.md` on the platform). The platform is a **kernel**: it manages the
agents, the super-admins who administer them, and single sign-on. It runs no business
application. People sign in once at the kernel's person face (`app.<domain>`, the launcher) and
are handed to an application by a signed token. The application keeps **no password and no
account**; it keeps a **mirror** of the kernel's directory, with the kernel's ids.

## 1. Receiving a person — the hand-off token

*Minting is built on the kernel (2026-09-22, A3): `app/auth.php` — `mint_sso_token()`, `sign_sso_claims()`, and the verifiers `verify_sso_token()`, `verify_sso_claims()`, `verify_sso_logout_notice()`, which an application in PHP may copy verbatim.*

The launcher shows a person the applications they hold a live access grant on. Clicking one
mints a token and redirects the browser to the application's `sso.path` (declared in
`maludb-os.json`, `registration.md`):

```
GET https://hr.subello.com/sso?token={member_id}.{expires}.{app_key}.{nonce}.{hmac}&claims={base64url json}.{hmac}
```

| Part | Meaning |
|---|---|
| `member_id` | The kernel's member id — the same id the application mirrors |
| `expires` | Unix seconds; **TTL 60 s** |
| `app_key` | The audience: the application's `catalog_key`. A token for HR opens nothing else |
| `nonce` | 16 random bytes, hex. **Single use**: store it until `expires`; refuse a second presentation |
| `hmac` | HMAC-SHA256 with the tenant's `ACTION_TOKEN_KEY` over `"sso:member.exp.app.nonce"` |
| `claims` | A signed JSON body beside the token (same key, over the base64url text): `display_name`, `email`, `business_role`, `is_external`, `status`, `departments: [{id, name, is_admin}]`, `capability` (the grant: `read`/`write`/`admin`) |

The receiver, in order: verify both signatures with constant-time comparison; check `expires`;
check `app_key` is this application; check the nonce is unused and record it; **upsert the
mirror row** from the claims (§3); open the application's own hardened session for that member
(the `php-session-auth` rules: regenerate the id, `SameSite=Lax`, `Secure`, `HttpOnly`, the
cookie for this exact host); log `member.sign_on` with `source = 'web'` and the kernel's
request id if `X-Request-Id` came; redirect to `/`. Any failure answers one page — *"This
sign-on link has expired. Open the application from app.<domain> again."* — and logs
`member.sign_on.refused` with the reason. Never distinguish the reasons to the visitor.

**There is no login form.** `/login` on an application redirects to the launcher
(`OS_LAUNCHER_URL` from `config/.env`). A visitor with no session on any other page is sent to
the launcher with `?app={catalog_key}` so it can come straight back.

## 2. Signing out

- The application's own sign-out ends **its** session only and returns to the launcher.
- The kernel's sign-out posts to `sso.logout_path` a signed notice
  `{member_id}.{issued}.{app_key}.{hmac}` (over `"sso-logout:member.issued.app"`, 120 s). The
  application ends every session it holds for that member and answers 204. Fail quietly: a
  kernel that cannot reach the application still ends its own session, and the mirror's status
  (§3) catches up within a minute.

## 3. The mirror — the application's `members` and `department_members`

The application's identity tables hold the **kernel's ids** and only these columns:

```sql
CREATE TABLE members (
    id              bigint PRIMARY KEY,                      -- the kernel's member id, never generated here
    member_kind     text NOT NULL CHECK (member_kind IN ('human','agent')),
    display_name    text NOT NULL,
    email           text,
    business_role   text NOT NULL CHECK (business_role IN ('super_admin','dept_admin','user')),
    is_external     boolean NOT NULL DEFAULT false,
    status          text NOT NULL CHECK (status IN ('active','inactive')),
    capability      text CHECK (capability IN ('read','write','admin')),   -- this application's grant, NULL = none
    synced_at       timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE department_members (
    member_id       bigint NOT NULL REFERENCES members(id) ON DELETE CASCADE,
    department_id   bigint NOT NULL,                          -- the kernel's department id
    department_name text NOT NULL,
    is_admin        boolean NOT NULL DEFAULT false,
    PRIMARY KEY (member_id, department_id)
);
```

Rules: a row is written only from a hand-off's claims or from the change feed (§4); a request
naming a member id with no row is **refused, never auto-created** (a run token included); every
request re-checks `status = 'active'` and, for people, `capability IS NOT NULL`; the
application adds its own columns (preferences, an HR employee record) in **its own tables keyed
by `member_id`**, never on the mirror. `app.member_id` therefore means the same person in every
application, on the kernel, and in the MaluDB namespaces `member:<id>`, `agent:<id>`, `dept:<id>`.

**Roles are directory facts.** `business_role` and `is_admin` travel with the token and the
feed; what a dept-admin may do *inside* the application is the application's rule (HR lets one
approve leave in their departments). Who may sign in to the kernel's admin face is the kernel's
rule (super-admins only) and never the application's concern.

## 4. The directory API — reading the directory, and (HR only) changing it

*Built on the kernel 2026-09-22 (A4).* The kernel serves it on its internal port, `OS_INTERNAL_URL`
(`http://127.0.0.1:8080`), under `/api/v1/directory/`. It is never on a public name. Every call
carries `Authorization: Bearer ${OS_APPLICATION_TOKEN}` — the **application token** a super-admin
mints on the application's Overview in the kernel (one live token; minting again replaces it;
retiring the application revokes it) and writes into `config/.env` — and a write also carries
`X-Acting-Member: <member id>` of the signed-on person, whose kernel rights decide. Bodies are JSON
(`Content-Type: application/json`) or a form. One 401 sentence covers every token failure.

| Call | Use it for | Who may |
|---|---|---|
| `GET members.php` | Every member (every kind and status) with their live departments — the mirror's full refresh | Every active application with a token |
| `GET departments.php` | Every department (archived ones dated) and every live membership | Same |
| `GET changes.php?since=<next>` | The mirror's timer: every minute, apply the rows, store `next` (`directory_sync_state`, like the ingest checkpoint, under an advisory lock). No `since` = the whole directory. Rows are current state, not events — a member carries its `status`, a membership its `left_at`, a department its `archived_at`; upsert, remove what has left. A deleted department has no row: `deleted_departments[]` (id, name, deleted_at) lists them since the cursor — every one ever on a full answer — and the mirror deletes that row by id. `next` is taken 10 s back, so a row may arrive twice | Same |
| `POST members.php` | Invite a human: `email`, `business_role` (`user`/`dept_admin`), `department_id`, `message` → 202 `{status:"invited", invitation_id, email, expires_at}`; the kernel sends the invitation; the member appears in the feed once they register | `directory.writes` declared; the acting member is the super-admin or an admin of that department |
| `PATCH members.php?id=<member>` | `display_name`, `job_title`, `phone`, `timezone`, `is_external`, `status` (`active`/`suspended`), `business_role` (`user` ↔ `dept_admin`) → `{member}` | `directory.writes`; the people rule (`app_can_admin_member`); never a super-admin, never an agent |
| `POST memberships.php` | Add or update: `member_id`, `department_id`, `is_admin`, `is_primary` | `directory.writes`; an admin of that department |
| `DELETE memberships.php` | Remove: `member_id`, `department_id` (dated with `left_at`, never erased) | Same |
| `POST departments.php` | Create: `name`, `description`, `parent_id`, `manager_member_id` → 201 `{department}` | `directory.writes`; the super-admin, or an admin of the parent |
| `PATCH departments.php?id=<department>` | Rename, re-parent (a cycle is refused), name the manager, describe | `directory.writes`; an admin of that department |

The feed document is `"schema": "os.directory-changes/1"` with `since`, `next`, `full`, `members[]`,
`departments[]`, `memberships[]`, `deleted_departments[]` (id, name, deleted_at); the lists are `os.directory/1`; both additive within a major
version. A member row: `id, display_name, email, member_kind, business_role, is_external, status,
job_title, phone, timezone, departments[{id,name,is_admin,is_primary}], updated_at`. A membership:
`member_id, department_id, is_admin, is_primary, joined_at, left_at`. A department: `id, name,
description, parent_id, manager_member_id, is_system, system_key, archived_at, updated_at`.

What an application can **never** do through it: grant a module or an application, mint or revoke
a token, touch an agent, or set `super_admin`. HR proposes; a super-admin grants in the kernel.

**Writes are attributed to the person**, not to the application: the kernel logs
`source = 'application'`, the acting member as actor, and `via_application` (the app key) in
the row. Log the same event in the application's own `activity_log` with the kernel's
`request_id` (returned in the response header) so the two trails join.

## 5. The command bar — the application's assistant runs in the kernel

*Built on the kernel 2026-09-22 (A6).* Every application on our stack ships the voice-first
command bar (`chat-actions` in `htmx-php-builder`). It stays the application's feature, but the
model behind it is **not the application's**: the application holds no model key, and every model
call must reach the kernel's prompt ledger. So the bar posts the utterance to the kernel:

```
POST {OS_INTERNAL_URL}/api/v1/agents/chat.php?agent=expert        (or ?agent=<agent member id>)
Authorization: Bearer ${OS_APPLICATION_TOKEN}
X-Acting-Member: <member id>
Content-Type: application/json
{"utterance": "...", "screen": "<manifest screen key>", "context": {"record_id": 12}, "conversation_id": "<your id>", "wait": 60}
```

`expert` is the agent the kernel names on the application's Expertise tab; any other agent must
hold access to the application. The kernel runs ONE turn of that agent as an agent run (trigger
`chat`) — its own grants, every model call through the ledger proxy, approvals paused in the
kernel's queue, the acting person as requester, the application stamped on the run and its ledger
rows — and answers:

```
200 {"run_id", "request_id", "status": "succeeded|failed|awaiting_approval|cancelled", "finished": true,
     "reply": "...", "error": null, "approval_request_id": null,
     "actions": [{"tool": "booking_create", "status": "ok", "duration_ms": 412, "record_id": 88}],
     "navigate": null, "conversation_id": "...", "cost": "0.0392", "currency": "USD"}
202 {... "status": "running", "finished": false ...}   → GET /api/v1/agents/chat.php?run=<run_id> until finished
```

`wait` (seconds, up to 110) is how long the kernel holds the request; a run that outlasts it is
fetched with GET. `conversation_id` is yours: the kernel carries the last three turns of it into
the next prompt. `actions` are the tool calls the run made, in order; a paused one shows as
`awaiting_approval` with the request id — the reply says so in the agent's words. `navigate` is
reserved (the kernel does not know your screens). Refusals: 400 no acting member, 403 no grant or
an agent without access, 404 no expert named, 409 the agent is busy or inactive, 422 an empty
utterance. Render the reply, list the actions, follow nothing blindly.

## 6. The ledger feed — for the accounting application

*Built on the kernel 2026-09-22 (A5).* The kernel keeps token accounting only and hands the
accountant a period statement; it never posts a journal. An accounting application from us pulls
the statement on its own timer with the same application token:

```
GET {OS_INTERNAL_URL}/api/v1/ledger/periods.php                 → {"schema":"os.ledger-periods/1","periods":[{period,status,closed_at,statement_lines,…}]}
GET {OS_INTERNAL_URL}/api/v1/ledger/periods.php?period=2026-08  → the document os.ledger-period/1
```

The document: `period, period_start, period_end, status (open|closed), closed_at, currencies[],
exchange, amount_scale, lines[{provider, model, department, agent, application, calls, late_calls,
input_tokens, output_tokens, cache_read_tokens, cache_write_tokens, amount, currency, note}],
totals[{currency, calls, late_calls, input_tokens, output_tokens, amount}]`. Book a **closed**
month once; an open month's lines still move. A late call — dated in a closed month, arrived after
— is counted in the open month and flagged, never in the closed one. The same document is what a
person downloads from the kernel's Statements screen, and what the `ledger_period` MCP tool answers.

## 7. Environment the kernel gives an application

| Key | Meaning |
|---|---|
| `ACTION_TOKEN_KEY`, `ACTIONS_RELAY_KEY` | The tenant's; verify run tokens, hand-off tokens, sign-out notices, relayed actions |
| `OS_APPLICATION_TOKEN` | This application's bearer for the directory, ledger and chat endpoints; minted on the application's Overview in the kernel, shown once, stored hashed there; one live token, rotated by minting again; revoked when the application is retired |
| `OS_INTERNAL_URL` | `http://127.0.0.1:8080` |
| `OS_LAUNCHER_URL` | `https://app.<domain>/` |
| `APP_KEY` | This application's `catalog_key` — the audience of every token it accepts |
| `MALUDB_API_URL`, `MALUDB_API_TOKEN` | The tenant's one memory (`memory.md`) |

## Checklist

- [ ] `sso.path` receiver: two signatures, TTL, audience, single-use nonce, mirror upsert, own session, one refusal page, both events logged.
- [ ] `sso.logout_path` receiver ends the member's sessions, answers 204.
- [ ] No login form; `/login` and every unauthenticated page go to the launcher.
- [ ] `members` / `department_members` mirror with the kernel's ids; unknown id refused; status and capability re-checked per request; application data in its own tables.
- [ ] Directory timer on `GET changes` with a checkpoint and an advisory lock; full refresh on first run.
- [ ] Writes (HR only): `X-Acting-Member`, `directory.writes` declared, the kernel's `request_id` copied into the local log.
- [ ] Command bar posts to the kernel's chat endpoint; no model key anywhere in the application.
