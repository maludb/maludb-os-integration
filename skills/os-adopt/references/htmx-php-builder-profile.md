# Adopting an application built with `htmx-php-builder`

*Written from the Cidery adoption (2026-10-04, `github.com/maludb/maludb-os-cidery`, `docs/os-adoption.md` there).* An
application built with the `htmx-php-builder` plugin before it learned the kernel (≤ 0.4.x) comes out in one shape, so
its adoption is one list of deltas, each with the fix that worked. Survey it as `survey.md` says, then expect these.

| What the builder produced | What the contract wants | The fix (Cidery's files) |
|---|---|---|
| `config/application.php` defaults + `config/local.php` (PHP array) for PHP; `config/services.env` (`CIDERY_*` keys) for Python | `config/.env` with the contract's keys (`DB_NAME`, `MCP_RECORDS_PORT`, `ACTION_TOKEN_KEY`, `OS_*`), the one file the installer writes | Keep both files; add `env()` and an env→config map to `application.php` so `.env` overrides; make `services/common/config.py` read `.env` too, with `CIDERY_*` → contract aliases. Nothing else in the code changes. |
| The deployment root is `/var/www` (units, Apache include, README) | `/srv/apps/<key>`, with `{{APP_DIR}}` in every template | New `deploy/apache-<key>.conf` and `deploy/<key>-*.service` templates with the installer's placeholders; the old files stay for the standalone product. |
| `php-session-auth`: password + TOTP + Google, invitations, reset; `complete_login()` fills the session | The hand-off token is the only way in | `/sso` ends in `complete_login()`; `os_close_local_signin()` at the top of login, 2FA, reset, invite and Google; `require_login()` → launcher; `logout` → launcher. The TOTP challenge is skipped on a hand-off (the kernel authenticated the person). |
| `users.role` is ONE value; `require_role(...)` and `user_can()` test it | The kernel grants a SET of roles | `users.os_roles text[]` + `users.os_capability`; `user_can()` also tests the set; `users.role` keeps the highest for screens that show one. |
| A users admin (`html/users/*`) invites, changes roles, disables | People are managed in the kernel | The list and form show `os_managed_notice()`; every handler calls `os_refuse_if_managed()` after `require_post()`. |
| One database per client + an operator registry (`<app>_host`), `provision-client.sh` (roles per file, `enable_memory_schema`, grants last, a seed row) | One database per tenant, created by the installer | `deploy/os-provision.sh` (idempotent, `app.schema_migrations`) named in `maludb-os.json` → `database.provision`; the registry database is not used under the OS. |
| Python services in `services/` with `services/.venv` | `mcp/` + `mcp/venv` by convention | `maludb-os.json` → `runtime.python` `{dir, venv, requirements}`; the installer builds that venv. |
| `activity_log.action` like `receipt_posted`; `source` ∈ screen/command_bar/ama/mcp/system; shipped into the in-database MaluDB memory schema | `entity.verb`, `source` incl. `agent`; one episode per row to the TENANT's MaluDB with `application` | Widen the `source` check, add `agent_run_id`; `bin/activity_ingest.php` does both (the local schema for the app's own activity server, the tenant's MaluDB through the API, checkpoint `activity_ingest_state`). The action names stay; the registry carries `log_event` in `entity.verb` form for approval policies. |
| The builder's own actions MCP server (localhost) with a 2-part action token; handlers answer HTMX (HX-Location, 422 forms) | The KERNEL's actions server POSTs the handlers in JSON mode with the tenant's 3/4-part tokens and the relay | `verify_action_token()` learns the 3/4-part shapes and the relay/replay rules; `app/json_mode.php` translates the HTMX answer at shutdown (HX-Location → `location`, invalid-feedback → `errors`, `#error-message` → `message`) — no handler rewritten. |
| `docs/05-action-manifest.md` (6 columns) → `config/manifest.json` for the builder's server; endpoints like `POST /vessels/{id}/status` and rows like `/kegs/{id}/state (event=clean)` | `mcp/action_registry.json` in the kernel's shape, flat endpoints, typed params, `log_event`, `who`, `confirm`, `approval`, `built` | `bin/build_action_registry.php` derives it: `{id}` → a named entity parameter the kernel resolves and substitutes into the path (`/vessels/{vessel}/status`), `(event=clean)` → `fixed`; `deploy/kernel-registry-<key>.json` names a `find_<kind>` resolver per entity. The kernel's tool factory supports both since 2026-10-04. |
| The builder's assistant service (Claude Agent SDK, holds `ANTHROPIC_API_KEY`) behind the command bar and Ask me anything | The command bar runs the app's expert IN the kernel; the application holds no model key | `html/assistant/message.php` branches on `os_enabled()` to the kernel's chat endpoint (`app/os_assistant.php`); the standalone service is simply not installed beside a kernel (not in `services[]`). |
| MCP servers on `mcp` 2.x (`MCPServer`), bearer tokens named, not owned, per scope; no member identity | The kernel's token → `app_roles` only; an agent's run token → the run-facts gate; a person's action token | The same middleware contract on the 2.x API (`services/common/auth.py`); `app_roles` from the catalogue tables; `find_<kind>` resolver tools answering `{"rows": [{<kind>_id, label}]}`. |
| `.htaccess` pretty-URL router | — | Keep it; the vhost template says `AllowOverride All`. |

## The order that worked
1. `config/.env` support (PHP, Python) — everything after reads it.
2. The migration (link columns, kit tables, sources, the roles catalogue) and the provision script; prove both on a scratch database, twice.
3. `app/os.php`, `/sso`, `/sso/logout`, `/api/v1/health`, the guard, the local screens under the flag; prove with `bin/dev_handoff.php`.
4. JSON mode, the kernel tokens in `verify_action_token()`, the registry builder.
5. The MCP servers' kernel contract and the resolvers.
6. The command bar through the chat endpoint.
7. `maludb-os.json`, `os/expert.md`, skills, `deploy/`; `bin/app_install.php plan`.

## Two traps
- **The guard inside the login.** If `current_user()` re-checks the session list on every request, make `complete_login()`
  list the session BEFORE anything in it asks `current_user()` (a `log_activity()` call will), or the new session is
  cleared as it is born.
- **Column lists.** A `USER_COLUMNS` constant in the queries file hides the new `os_*` columns from every lookup; add them
  there, not in one query.
