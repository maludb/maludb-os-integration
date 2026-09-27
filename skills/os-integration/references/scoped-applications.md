# Scoped applications — one installation, several sites or departments

*Built on the kernel 2026-09-25 (C1, db/141; kernel spec `docs/build-specs/kernel-scoped-applications.md`).*

Many applications keep separate data for separate parts of the business. A reservation system keeps
each restaurant's tables and bookings apart, and a project tool keeps each department's plans apart.
One installation serves them all. The kernel owns **which** parts exist and **who** may reach each of
them. The application owns what happens inside.

| Word | Meaning |
|---|---|
| **Scope kind** | What one installation is partitioned by: `none` (one set of data — HR), `location` (each **site**), `department` (each department). Declared in `maludb-os.json` → `scopes.kind`. |
| **Site** | A kernel location of kind `site`: a place the business trades from (a restaurant, a shop, a branch). It is not a machine. It has a `name`, an `address` and an IANA `timezone`, and people and agents *reside* at it. |
| **Scope** | One site or one department that this installation serves, identified by the kernel's `scope_id`. A super-admin adds scopes on the application's **Scopes** tab (`application_scope_add`). |
| **Role** | The application's own word for what someone may do (`admin`, `manager`, `user`), **published by the application** through `app_roles` since 0.4.0 (`roles-and-rights.md`), with the rights it gives. Each role **amounts to** a kernel capability (`read`/`write`/`admin`). Exactly one role is `is_admin`: a super-admin holds it in every scope. |
| **Grant** | Kernel-side, one per scope. It names a scope and a **set** of roles (0.4.0; one before), and goes to a member, a department (its live members), or **everyone residing at a site**. Nobody holds anything by default: **application users are not OS users**. |
| **Holding** | Everything one member holds on this application: on a scoped application, `[{scope_id, role, roles, rights, capability}]`, one entry per scope; `role` is the highest of `roles`. |

## 1. What the application declares

```json
"scopes": { "kind": "location" },
"roles": [
  { "key": "admin",   "name": "Admin",   "capability": "admin", "is_admin": true },
  { "key": "manager", "name": "Manager", "capability": "write" },
  { "key": "user",    "name": "Staff",   "capability": "write" }
]
```

- `scopes.kind` is `location` or `department`. Leave the block out for an unscoped application.
- **Since 0.4.0 the roles are not declared here**: the application publishes them through `app_roles` and the kernel reads them (`application_roles_refresh`) — `roles-and-rights.md`. The `roles` block above is what 0.3.0 wrote with `application_roles_set`; it still works for an application that cannot publish.
- Roles work without `scopes`: an unscoped application with its own roles. The claims then carry `role`, `roles` and `rights`.
- Role keys must match `^[a-z][a-z0-9_]{0,39}$`. The list is ordered; `role` names the highest of a holding's roles (capability first, then this order).
- The installer writes `scope_kind` with `application_save`, then — after the endpoints — reads the roles with `application_roles_refresh` (super-admin).

## 2. What the application receives

**At sign-on.** The claims beside the hand-off token gain three fields. They are additive, so an old receiver still works.

```json
"capability": "write",                         // the highest capability held (unchanged meaning)
"role": null,                                  // unscoped application with roles: the member's role
"roles": ["manager", "user"], "rights": ["tables.book", "floor.manage"],   // 0.4.0: every role held, and what they give
"scopes": [ {"scope_id": 9, "kind": "location", "id": 21, "name": "Airport",
             "role": "manager", "roles": ["manager", "user"], "rights": ["tables.book", "floor.manage"], "capability": "write"} ],
"scope": 9                                     // the scope chosen on the launcher; null when several and none chosen
```

- `id` is the kernel's location or department id.
- **Key your data by `scope_id`, not by `id`.** A scope removed and added again is a new scope.
- The launcher shows one button per scope a person holds (`/launch/<app>?scope=<scope_id>`). A person with one scope gets it chosen automatically. A scope the person does not hold is refused before any token is minted.

**From the change feed** (`GET {OS_INTERNAL_URL}/api/v1/directory/changes.php?since=`, schema `os.directory-changes/1`). Two lists concern **this application only**:

| List | Row | Apply it |
|---|---|---|
| `scopes[]` | `scope_id, kind, location_id, department_id, name, address, timezone, removed_at, updated_at` | Upsert your tenant row (the restaurant) by `scope_id`. Its name, address and time zone follow the kernel's. When `removed_at` is set, **close** it: no one reaches it any more. Keep its data, because the kernel removing a scope is not a request to delete bookings. |
| `access[]` | `member_id, role, roles, rights, capability, scopes[{scope_id, role, roles, rights, capability}]` | **Replace** that member's holding. With `scopes: []` and `capability: null` they hold nothing now: remove their access and end their sessions. A row for a member your mirror does not know **and** who holds nothing is ignored. |

- A full answer (no `since`) lists every live scope and every member who holds anything.
- A `since` answer lists every scope that changed and every member whose holding **may** have changed. That covers grants, revocations, expiries, department moves, residency changes, status changes and scope removals. Apply the list in order; rows may repeat because of the 10-second overlap.

**On demand.** `GET {OS_INTERNAL_URL}/api/v1/directory/scopes.php` returns `{"schema":"os.directory-scopes/1", scope_kind, roles[], scopes[]}` with the application token. A fresh installation calls it at install time, so every restaurant exists before anyone signs in.

**For agents.** Run facts (`POST /api/v1/runs/facts.php`) answer `capability`, `role` and `scopes[]` for the caller, in the same shape as the claims. An agent is granted per scope, like a person, and an agent with `scopes: []` may act in no scope.

## 3. The mirror tables to add

These go beside `members` and `department_members` from `sign-on-and-directory.md` §3.

```sql
-- One row per scope this installation serves, keyed by the kernel's scope id.
CREATE TABLE os_scopes (
    scope_id       bigint PRIMARY KEY,                     -- the kernel's application_scopes.id
    kind           text NOT NULL CHECK (kind IN ('location', 'department')),
    location_id    bigint,                                 -- the kernel's location (a site)
    department_id  bigint,                                 -- the kernel's department
    name           text NOT NULL,
    address        text,
    timezone       text,
    removed_at     timestamptz,                            -- closed: nobody reaches it
    synced_at      timestamptz NOT NULL DEFAULT now()
);

-- What each member holds, per scope. Replaced whole from the claims or an access[] row.
CREATE TABLE member_scope_roles (
    member_id   bigint NOT NULL REFERENCES members(id) ON DELETE CASCADE,
    scope_id    bigint NOT NULL REFERENCES os_scopes(scope_id) ON DELETE CASCADE,
    role_key    text,                                      -- the application's role; NULL when it declares none
    capability  text NOT NULL CHECK (capability IN ('read', 'write', 'admin')),
    synced_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (member_id, scope_id)
);
```

Your own tenant table (`restaurants`, `project_spaces`) carries `scope_id bigint UNIQUE REFERENCES os_scopes`. A new application may use `os_scopes` itself as its tenant table. An adopted application links its existing table (`os-adopt`).

## 4. The rules inside the application

1. **Every scoped row carries the scope.** Every table holding a restaurant's data has the tenant key. Every query, `mcp_*` view and MCP tool filters by the scopes the caller holds, in SQL: `scope_id IN (SELECT scope_id FROM member_scope_roles WHERE member_id = app.member_id)`. A super-admin is in `member_scope_roles` like anyone else, because the claims and the feed list their admin role in every scope. Never special-case them in the application.
2. **The session has a current scope.** Set it from the claims' `scope`. When the claims leave it null, use the person's first scope and offer a switcher over the scopes they hold. Re-check it on **every request** against `member_scope_roles`. If the scope is no longer held, or it is closed, pick another held scope or end the session.
3. **Roles decide inside a scope.** Map your guards to `role_key`, e.g. `require_role('manager')` against the current scope's row. The capability is the kernel's summary, for the kernel's own screens and for agents.
4. **Structure is the kernel's.** The application never creates or renames a scope, and never grants a role. A "new restaurant" button in the application sends the admin to the kernel's Scopes tab. A "staff" screen becomes read-only, lists who holds what here, and links to the kernel.
5. **Agents obey scopes too.** An MCP tool called with a run token filters by the scopes that run facts returned. An action names its scope, and the handler refuses a scope the caller does not hold.
6. **Nothing leaks across scopes.** Search, reports, exports, emails and the activity log shipped to MaluDB all carry the scope. An activity row's `after` includes `scope_id`, so the memory can answer "what happened at Airport".

## 5. Proof (add to the application's proofs)

- The claims for a member holding one scope open that scope. With two scopes, `scope` opens the chosen one and the switcher lists exactly two.
- A request for a scope the member does not hold is refused. That covers the switcher, a direct URL, an MCP tool and an action.
- The feed: add a scope in the kernel and the next sync creates the tenant. Revoke a grant and the next sync removes the holding and ends the member's sessions. Remove a scope and the tenant closes with its data kept.
- A run token whose facts return `scopes: []` can read nothing scoped.
