# The adapter, step by step

The worked example is ZozoCal-Restaurant: a multi-tenant restaurant reservation system on
PHP/PostgreSQL 17 with `users`, `restaurants` and `user_restaurants(user_id, restaurant_id, role)`.
Its login function is `login_user()`, and its guard is `requireAuth()` (98 callers). Substitute
your application's names.

## 1. Migration (`docs/sql/os_adoption.sql` or the next numbered migration)

```sql
ALTER TABLE users ADD COLUMN os_member_id bigint UNIQUE;          -- the kernel's member id
ALTER TABLE restaurants ADD COLUMN os_scope_id bigint UNIQUE;     -- the kernel's scope (a site)
ALTER TABLE user_restaurants ADD COLUMN source text NOT NULL DEFAULT 'local'
    CHECK (source IN ('local', 'os'));                            -- 'os' rows are the kernel's; the feed replaces them
-- + sso_nonces, member_sessions, directory_sync_state (php-sign-on-kit.md §2)
-- + os_scopes and the application's roles map, if it wants its own copy (scoped-applications.md §3)
```

- An adopted application does not need a separate `members` table. `users` plus `os_member_id` is
  the mirror.
- It does not need `member_scope_roles` either. The tenant membership table, holding the rows with
  `source = 'os'`, is the holding.
- Keep `department_members` only if the application uses departments.

## 2. Verifiers

Copy `php-sign-on-kit.md` §1 into `helpers/os_tokens.php`, or wherever the application keeps helpers.
Read `ACTION_TOKEN_KEY`, `APP_KEY`, `OS_LAUNCHER_URL`, `OS_INTERNAL_URL` and `OS_APPLICATION_TOKEN`
through the application's own config function (`app_config('ACTION_TOKEN_KEY')`).

## 3. `/sso` and `/sso/logout`

This is the receiver of `php-sign-on-kit.md` §4, with the mirror step replaced by the link, all in
one transaction:

1. **Find the user by `os_member_id`.** If none is found, match an existing user by email, once,
   and only if that user has no `os_member_id` yet. Record the link and log it. If there is still
   none, create the user with an unusable password (`'!OS'`) and the claims' name and email.
2. **Update the name and email** from the claims.
3. **Replace the rows with `source = 'os'`** in the membership table from `claims.scopes`. For each
   scope, find the tenant by `os_scope_id` and write the application's role (the claims' `role`).
   A scope with no tenant yet is created first, the way the sync does (§6).
4. **Apply the platform super-admin mapping.** If the owner decided that `claims.business_role ===
   'super_admin'` makes the user the application's platform admin, set it. Otherwise clear it.

Then call the application's own login function, for example `login_user($user, false)`, and select
the tenant for `claims.scope` (`switchRestaurant(<the restaurant for that scope>)`). Then call
`session_open()` and redirect to the application's home.

`/sso/logout` is the kit's §5 as it stands. It needs the session list, because a PHP file session
cannot be found by user id.

## 4. The guard

```php
function requireAuth(): void {
    if (os_enabled()) {
        $u = $_SESSION['user'] ?? null;
        if ($u === null || !session_is_listed(db(), session_id()) || !os_user_still_admitted((int) $u['id'])) {
            os_to_launcher();              // HX-Redirect for HTMX, Location otherwise
        }
    }
    // … the existing body, unchanged
}
```

- `os_user_still_admitted()` checks that the user is active, linked, and still holds the current
  tenant (a membership row with `source = 'os'`, whose tenant has not been closed).
- The tenant switcher offers only the tenants held, and verifies CSRF.
- Keep the check to one indexed query. It runs on every request.

## 5. The local account screens, under the flag

While `os_enabled()`:
- `login.php`, `register.php`, the forgot and reset pages, and the Google callback all redirect to
  the launcher.
- Staff and user management, and tenant creation, render read-only with "Managed in the operating
  system" and a link to `OS_LAUNCHER_URL`. Their save handlers answer 403 with the same sentence.
- Logout ends the application's session, marks the session list, and redirects to the launcher.
- API keys keep working for users who are linked and active. The API's auth adds the same
  `os_user_still_admitted()` check.

## 6. The directory sync (`bin/os_directory_sync.php` + a systemd timer, every minute)

This is the kit's §6 order, with the adopted mapping:
- **`scopes[]`:** upsert the tenant by `os_scope_id`. A new scope creates a tenant seeded exactly as
  the application's registration seeds one: default settings, a first section, opening hours. Use
  the scope's name and time zone, and a slug from the name. A rename renames it. A removal marks the
  tenant inactive and keeps its data.
- **`access[]`:** for each linked user, replace the rows with `source = 'os'` from `scopes`. When
  `capability` is null, end their sessions. An unlinked member is ignored.
- **`members[]`:** for each linked user, update the name and email. When `status` is not `active`,
  deactivate the user and end their sessions.
- On the first run, and on `--full`, call `scopes.php` first.

## 7. `maludb-os.json`

Follow `../os-integration/references/registration.md`. For an adopted, scoped application:

```json
"sso": { "path": "/sso", "logout_path": "/sso/logout" },
"directory": { "reads": true, "writes": false },
"scopes": { "kind": "location" },
"identity": { "profile": "adopted", "enabled_env": "OS_ENABLED" }
```

Plus `catalog_key`, `vhost` (`document_root` as the application has it), `database`, `env`,
`services` (the sync timer), `endpoints`, and `agents` when an expert is written.

**Roles (0.4.0).** The application's roles are not declared in `maludb-os.json` any more. Map the roles the
application already has (`user_restaurants.role`: admin, manager, staff) to a catalogue with the rights
each gives, publish it through `app_roles` on the records MCP server, and admit the kernel's token to that
tool — `../os-integration/references/roles-and-rights.md`. The kernel then grants any set of them per
scope, and the adapter writes the tenant membership row's role from `scopes[].role` (the highest) or
`scopes[].roles`.

An application with no MCP servers yet declares the endpoints it has. For example, ZozoCal's admin
MCP at `/api/mcp/admin.php` is `auth_kind: bearer`. Record the memory and MCP parts of the contract
as owed.

## 7b. The kernel's actions server reaching HTMX handlers (2026-10-04)

The kernel's actions server POSTs a handler with `Accept: application/json`, `X-Action-Token` (the tenant's 3-part
person token or 4-part run token) and, for a run token, `X-Action-Relay`. An existing application's handlers answer
HTMX. Do not rewrite them: buffer the whole answer when JSON mode is on and translate it at shutdown — `HX-Location`
/ `Location` → `location` (the kernel takes `record_id` from its tail), `invalid-feedback` divs and `alert-danger`
lists → a 422 with `errors` and `fields`, the error partial's `#error-message` → the message, 401/launcher redirect →
401. A new handler may still call `emit_action_status()` to say more. The reference implementation is Cidery's
`app/json_mode.php`. Endpoints with path parameters (`/vessels/{id}/status`) are declared in the registry with the
entity's name in the path (`/vessels/{vessel}/status`); the kernel resolves and substitutes it (registration.md, "Path
parameters").

## 8. `/api/v1/health` and `deploy/`

- **Health:** an unauthenticated GET answering `{"ok": true, "application": "<key>", "version": "…", "database": "ok"}`.
- **`deploy/<key>.conf`:** the Apache vhost, with `ServerName <label>.<domain>`, the document root,
  the `/sso` rewrites, and a `Listen 127.0.0.1:<port>` vhost for the kernel's actions server.
- **Systemd units** for the sync timer. Existing cron lines move under `deploy/` with paths taken
  from the install path, not hard-coded.

## 9. Adoption proofs (besides `testing-without-a-kernel.md`)

- **Standalone:** with `OS_ENABLED` off, the existing test or check scripts pass, and password
  login works as before.
- **On:** every path into a session other than `/sso` goes to the launcher. That covers the login
  form, Google, registration, invitation acceptance, reset and remember-me.
- **First sign-on of an existing user:** the email match links them once. A second person with the
  same email is not linked to the same account, because the link is unique.
- **Scope held:** the development hand-off with a scope opens that tenant. The switcher lists only
  held tenants, and a direct request for another tenant's page is refused.
- **Revocation:** the fixture feed revokes and the next request lands on the launcher. The API key
  of a deactivated user answers 401.
- **Scope added:** the fixture feed adds a scope and the tenant appears, seeded. A removed scope
  leaves the tenant inactive with its data intact.
