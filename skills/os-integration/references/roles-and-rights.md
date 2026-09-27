# Roles and rights — the application publishes them, the kernel grants them

*Built on the kernel 2026-09-27 (db/145; kernel spec `docs/build-specs/kernel-application-roles.md`).
First application: HR (`/srv/apps/hr`, db/012, `docs/build-specs/roles-and-rights.md`).*

The kernel assigns rights **inside** an application, and the application decides what those rights are.
The application publishes its roles through its own MCP server, and the super-admin grants a person any
**set** of them. The application enforces what the roles give.

| Word | Meaning |
|---|---|
| **Right** | One thing a holder may do inside the application, in the application's words: `pay.write` — "Record pay changes and run pay runs". Keys match `^[a-z][a-z0-9_.]{0,59}$`. |
| **Role** | A named bundle of rights (`payroll` — "Payroll"), with a description and the kernel **capability** it amounts to (`read`/`write`/`admin`). Keys match `^[a-z][a-z0-9_]{0,39}$`. Exactly one role is `is_admin`; a super-admin holds it everywhere. |
| **Grant** | Kernel-side. It gives a **set** of roles to a member, a department or everyone at a site (per scope on a scoped application). Its capability is the highest of its roles'. Only the super-admin grants, changes and revokes. |

## 1. Publish: the `app_roles` tool

Add one read-only tool, **`app_roles`**, that takes **no arguments**, to the application's records MCP
server. It answers this document:

```json
{
  "schema": "os.app-roles/1",
  "rights": [
    {"key": "directory.read", "description": "See the directory, departments, leave types and holidays"},
    {"key": "pay.read",       "description": "See everyone's pay"},
    {"key": "pay.write",      "description": "Record pay changes and prepare, approve and pay pay runs"},
    {"key": "hr.staff",       "description": "Run HR — everything"}
  ],
  "roles": [
    {"key": "employee", "name": "Employee", "description": "Anyone employed here.", "capability": "read",
     "is_admin": false, "rights": ["directory.read"]},
    {"key": "payroll",  "name": "Payroll",  "description": "Keeps pay.", "capability": "write",
     "is_admin": false, "rights": ["directory.read", "pay.read", "pay.write"]},
    {"key": "hr_admin", "name": "HR Admin", "description": "Runs HR.", "capability": "admin",
     "is_admin": true,  "rights": ["directory.read", "pay.read", "pay.write", "hr.staff"]}
  ]
}
```

The kernel refuses the whole document, and changes nothing, in any of these cases:
- the `schema` is wrong;
- a key has the wrong shape, or a role is published twice;
- a role names a right that is not listed in `rights`;
- a role has no name, or its capability is not read, write or admin;
- the admin role does not amount to `admin`;
- anything other than exactly one role is `is_admin`;
- the list of roles is empty.

Keep the catalogue in the application's own database, so that its SQL rules can use it (HR: `hr_rights`,
`hr_roles`, `hr_role_rights`, and a view `mcp_app_roles` to read them in one go). The order of `roles` is
the order the kernel shows them in.

```python
@mcp.tool(name="app_roles", annotations={"title": "Roles and rights", "readOnlyHint": True, "openWorldHint": False})
async def app_roles() -> str:
    """The roles this application offers and the rights each gives (os.app-roles/1)."""
    async with (await _pool()).acquire() as con:
        rights = await con.fetch("SELECT right_key, description FROM app_rights ORDER BY sort_order")
        roles = await con.fetch("SELECT role_key, name, description, capability, is_admin, rights FROM mcp_app_roles ORDER BY sort_order")
    return json.dumps({"schema": "os.app-roles/1",
        "rights": [{"key": r["right_key"], "description": r["description"]} for r in rights],
        "roles": [{"key": r["role_key"], "name": r["name"], "description": r["description"], "capability": r["capability"],
                   "is_admin": r["is_admin"], "rights": list(r["rights"])} for r in roles]})
```

## 2. Admit the kernel's own token to that tool only

The kernel calls `app_roles` as itself, not as a person or an agent. It sends a 60-second token signed with
the tenant's `ACTION_TOKEN_KEY` and bound to your `app_key`:

```
kernel.{expires}.{app_key}.{nonce 32 hex}.{hmac-sha256 over "kernel:{expires}.{app_key}.{nonce}"}
```

In the MCP server's bearer middleware, check for this token **before** resolving a member. When it
verifies, serve the request with no member, no rows and one tool. `tools/list` shows `app_roles` alone,
and any other `tools/call` is refused.

```python
def verify_kernel_token(token: str) -> bool:
    parts = token.split(".")
    if len(parts) != 5 or parts[0] != "kernel":
        return False
    _, exp, app, nonce, sig = parts
    key = ENV.get("ACTION_TOKEN_KEY", "").encode()
    if not key or not exp.isdigit() or app != ENV.get("APP_KEY") or len(nonce) != 32:
        return False
    expected = hmac.new(key, f"kernel:{exp}.{app}.{nonce}".encode(), hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, sig) and time.time() <= int(exp)

KERNEL_TOOLS = {"app_roles"}
# middleware:  if token and verify_kernel_token(token): request_is_kernel.set(True); return await inner(...)
# list_tools:  if request_is_kernel.get(): return [t for t in tools if t.name in KERNEL_TOOLS]
# call_tool:   if request_is_kernel.get() and name not in KERNEL_TOOLS: raise ToolError(...)
```

The kernel tries each **active `mcp` endpoint** registered for the application, in order, until one answers
`app_roles`. Register the records server before you ask the kernel to read the roles.

## 3. Register: read the roles

After the endpoints are registered, a super-admin presses **Read roles from the application** on the
application's Access tab. The installer does the same with the action **`application_roles_refresh`**
(`{"application": id}`). The refresh makes the kernel's copy match what you publish:
- a new role is added;
- a changed role is updated (name, description, capability, rights, order, admin flag);
- a role you stop publishing is **withdrawn**, never deleted.

Grants that still hold a withdrawn role are listed on the Access tab until someone changes them, and a
withdrawn role is never granted anew. After every refresh the change feed sends every holder again, so
changed rights reach your mirror within a minute. Publish a new version of a role under the **same key**.
Retire a role by dropping it from `roles`, then move its holders to another role in the kernel.

`maludb-os.json` → `roles` (0.3.0) is **no longer read** for an application from us. `application_roles_set`
stays for an application that cannot publish its roles (someone else's, reached only by grants). A refresh
turns that hand-set list off: after one, the kernel keeps only what the application publishes.

## 4. Receive: roles and rights beside the capability

Every place that carried `capability` and `role` now also carries **`roles`** (every role key held) and
**`rights`** (the union of the rights those roles give, as your catalogue says). The new fields are
additive, so an old receiver keeps working. `role` stays, set to the highest of the roles.

| Where | Unscoped application | Scoped application |
|---|---|---|
| Hand-off claims | `roles`, `rights` | `roles`, `rights` (the union over scopes), and each `scopes[]` entry has its own `roles`, `rights` |
| Change feed `access[]` | `member_id, role, roles, rights, capability, scopes[]` | the same, and each `scopes[]` entry has `roles`, `rights` |
| Run facts | `role, roles, rights, capability, scopes[]` | the same |

**Trust your own catalogue over `rights`.** Store the **role keys** and derive rights from your own tables.
The `rights` field is there for an application that keeps no catalogue of its own, and for reading by eye.
Keep only the roles you publish, and ignore any other key.

```php
/** Claims or an access[] row → the mirror. Only roles this application offers are kept. */
function mirror_apply_roles(PDO $pdo, int $memberId, array $roles): void
{
    $known = $pdo->query('SELECT role_key FROM app_roles')->fetchAll(PDO::FETCH_COLUMN);
    $keep = array_values(array_intersect(array_unique(array_map('strval', $roles)), $known));
    $pdo->prepare('UPDATE members SET roles = CAST(:r AS text[]), synced_at = now() WHERE id = :id')
        ->execute(['r' => '{' . implode(',', $keep) . '}', 'id' => $memberId]);
}
// sign-on: if (array_key_exists('roles', $claims)) mirror_apply_roles($pdo, $memberId, (array) $claims['roles']);
// feed:    foreach ($doc['access'] ?? [] as $a) { capability ← $a['capability']; roles ← $a['roles'] (none when capability is null);
//                                                 capability null → end the member's sessions }
```

On a scoped application, give `member_scope_roles` a `roles text[]` column and fill it from each
`scopes[].roles` entry. Keep `role_key` holding the highest role, for code that reads one.

**Apply `access[]`.** It is what makes a revocation or a role change reach you within a minute. Without it,
a person keeps their old rights until they next sign in.

## 5. Enforce: ask for rights, not capabilities

Write one SQL function that answers from the mirror and the catalogue, and make the rules call it:

```sql
CREATE FUNCTION app_has_right(p_right text) RETURNS boolean LANGUAGE sql STABLE SECURITY DEFINER
SET search_path = public AS $$
    SELECT app_is_super_admin()
        OR EXISTS (SELECT 1 FROM members m JOIN app_role_rights rr ON rr.role_key = ANY (m.roles)
                    WHERE m.id = app_current_member_id() AND m.status = 'active' AND m.capability IS NOT NULL
                      AND rr.right_key = p_right);
$$;
```

- PHP gates (`require_right('pay.write')`), `mcp_*` view filters and the menu should all use the same function.
- A super-admin holds every right.
- **Moving from capabilities:** until the kernel has sent roles (`roles = '{}'`), read the old capability as before. HR reads capability `admin` as the HR Admin role. A person's rights then do not change until a grant names their roles.
- A right the application decides from its own records works alongside a role. HR's Manager is both: the manager named on a person's employment record, **or** a holder of the Manager role over the departments they administer. Say which in the right's description.

## 6. Checklist

- [ ] `app_roles` on the records MCP server: no arguments, `os.app-roles/1`, from the application's own catalogue tables.
- [ ] The kernel's token (`kernel.…`) verified in the MCP middleware and admitted to `app_roles` only.
- [ ] `members.roles` (and `member_scope_roles.roles` when scoped), written only from the claims and from the feed's `access[]`.
- [ ] `access[]` applied: capability and roles replaced; nothing held → no access and every session ended.
- [ ] Rules call `app_has_right()`; the gates, views, menu and buttons agree; super-admin holds every right.
- [ ] Registered: endpoints first, then `application_roles_refresh` — the Access tab lists every role with its rights.
- [ ] Proven: grant a role → sign-on and the feed both deliver it → the right is on; change the role → it is off; revoke → no access.
