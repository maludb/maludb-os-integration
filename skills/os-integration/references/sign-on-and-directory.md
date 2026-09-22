# Sign-on and the directory — how a person gets in, and how the application knows who exists

Design: *Business OS — Kernel and Integration Design* (2026-09-22; repo copy
`docs/business-os-integration.md` on the platform). The platform is a **kernel**: it manages the
agents, the super-admins who administer them, and single sign-on. It runs no business
application. People sign in once at the kernel's person face (`app.<domain>`, the launcher) and
are handed to an application by a signed token. The application keeps **no password and no
account**; it keeps a **mirror** of the kernel's directory, with the kernel's ids.

## 1. Receiving a person — the hand-off token

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

The kernel serves it on its internal port, `OS_INTERNAL_URL` (`http://127.0.0.1:8080`),
under `/api/v1/directory/`. It is never on a public name. Every call carries
`Authorization: Bearer ${OS_APPLICATION_TOKEN}` — the **application token** the installation
agent minted at registration and wrote into `config/.env` — and a write also carries
`X-Acting-Member: <member id>` of the signed-on person, whose kernel rights decide.

| Call | Use it for | Who may |
|---|---|---|
| `GET members`, `GET departments` | A full refresh of the mirror (first run, or after a gap) | Every active application |
| `GET changes?since=<cursor>` | The mirror's timer: every minute, apply changes, store the cursor (`directory_sync_state`, like the ingest checkpoint, under an advisory lock) | Every active application that declares `directory.reads` |
| `POST members` | Create a human; the kernel sends the invitation | `directory.writes` declared, and the acting member passes the kernel's people rule |
| `PATCH members/{id}` | Name, contact details, external flag, status, role `user` ↔ `dept_admin` | Same |
| `POST` / `DELETE members/{id}/departments/{dept}` | Membership and the admin flag | Same |
| `POST departments`, `PATCH departments/{id}` | Create, rename, re-parent, name the manager | Same; admin of the parent |

The feed document carries `"schema": "os.directory-changes/1"`; it is additive within a major
version. What an application can **never** do through it: grant a module or an application,
mint or revoke a token, touch an agent, or set `super_admin`. HR proposes; a super-admin grants
in the kernel.

**Writes are attributed to the person**, not to the application: the kernel logs
`source = 'application'`, the application key and the acting member. Log the same event in the
application's own `activity_log` with the kernel's `request_id` (returned in the response
header) so the two trails join.

## 5. The command bar — the application's assistant runs in the kernel

Every application on our stack ships the voice-first command bar (`chat-actions` in
`htmx-php-builder`). It stays the application's feature, but the model behind it is **not the
application's**: the application holds no model key, and every model call must reach the
kernel's prompt ledger. So the bar posts the utterance to the kernel:

```
POST {OS_INTERNAL_URL}/api/v1/agents/{agent_key}/chat
Authorization: Bearer ${OS_APPLICATION_TOKEN}
X-Acting-Member: <member id>
{"utterance": "...", "screen": "<manifest screen key>", "context": {"record_id": ...}, "conversation_id": "..."}
```

`agent_key` is the shipped agent the registration names for the bar (`assistant.agent`,
normally the expert). The kernel runs one turn of that agent with the application's tools,
through its ledger proxy, as a run whose requester is the acting person; it answers
`{"reply": "...", "actions": [{"tool","status","record_id"|"approval_request_id"}], "navigate": "<screen>"|null, "run_id"}`.
An action the agent is not granted is refused by the kernel; an approval-category action pauses
in the kernel's queue and the reply says so. The application renders the reply and follows
`navigate`; it never calls a model and never ships ledger rows.

Until the chat endpoint exists on the kernel (owed — `agents.md`), the bar's router may run
**only on a person's own MCP token against the application's read servers** for navigation and
questions, with no model call the kernel cannot see: in practice, ship the bar wired to the
endpoint above and let it answer *"The assistant is not connected yet"* when
`OS_APPLICATION_TOKEN` is absent.

## 6. Environment the kernel gives an application

| Key | Meaning |
|---|---|
| `ACTION_TOKEN_KEY`, `ACTIONS_RELAY_KEY` | The tenant's; verify run tokens, hand-off tokens, sign-out notices, relayed actions |
| `OS_APPLICATION_TOKEN` | This application's bearer for the directory, ledger and chat endpoints; shown once, stored hashed on the kernel; revoked when the application is retired |
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
