---
name: os-adopt
description: Fit an EXISTING PHP application (one not built for the MaluDB Business OS) into the kernel — single sign-on from the kernel's launcher, users managed in the kernel's directory, and, for a multi-tenant application, each tenant (a restaurant, a branch, a department's space) mapped to a kernel scope with the application's own roles granted per scope. Works in the application's own repository and leaves it installable by `os-install`. Use when the user says "integrate this existing app with the OS", "adopt <app>", "add OS sign-on to <app>", "make <app> use the kernel's users", or points at a repository that has its own login, users table or tenants.
---

# Adopting an existing application

An **adopted** application keeps its code, its schema and its users table. It gains the kernel's
sign-on beside them and hands the management of people to the kernel. Nothing is rewritten that does
not have to be. The contract it must meet is the `os-integration` skill; this skill is the route an
existing codebase takes to get there. Read `os-integration`'s SKILL.md first, then
`../os-integration/references/sign-on-and-directory.md` and, when the application has tenants,
`../os-integration/references/scoped-applications.md`.

## The shape of an adoption

```
kernel launcher ──hand-off token──▶ /sso (new) ──▶ link users.os_member_id ──▶ the app's OWN login function
kernel change feed ──every minute──▶ bin/os_directory_sync.php (new) ──▶ users / tenant table / memberships
the app's guard ──no session──▶ kernel launcher        (instead of /login.php, while OS_ENABLED is on)
```

- **One flag.** `OS_ENABLED` in the application's config. Off: the application runs exactly as it
  does today, standalone. On: the kernel's sign-on is the only way in, and the local account screens
  are read-only. The flag is also the way back.
- **Link, don't replace.** Add `os_member_id bigint UNIQUE` to the users table. When the tenant table
  maps to a scope, add `os_scope_id bigint UNIQUE` to it. The application's own ids stay the keys of
  its own data.
- **End in the application's own login function.** Nearly every PHP application has one place that
  fills the session (`login_user()`, `do_login()`, `Auth::login()`). The `/sso` receiver verifies,
  links, then calls it, so every session key the application expects is filled the way it always was.
- **The kernel owns people and structure.** Users, their roles per tenant and the tenants themselves
  arrive from the claims and the feed. The application stops creating them while `OS_ENABLED` is on.

## How to work

1. **Honour the repository's own rules first.** Read its `CLAUDE.md`, `README` and contribution notes.
   If they require a written plan and approval before changes (a `tasks/todo.md`, an activity log,
   pushes after each change), follow them. Work on a branch named `os-adoption`, never directly on a
   server.
2. **Survey.** Follow [references/survey.md](references/survey.md) and write the answers into
   `docs/os-adoption.md` in the application's repository: stack and database, the login function and
   every path into a session, the guard and how many files call it, the users table, the tenant table
   and the per-tenant role table, the local account screens, API and MCP authentication, webhooks,
   cron, the model keys it holds, and the security gaps found. Cite each answer as `file:line`.
3. **Decide the profile, and stop for the owner** on anything the survey cannot decide:
   - the scope kind: `location` when the tenant is a place (a restaurant, a shop), `department` when
     it is a team's space, `none` when there is no tenant;
   - the application's roles and the kernel capability each amounts to;
   - which role is the admin role;
   - what happens to the application's platform super-admin, if it has one. The recommendation: the
     kernel's super-admin is the application's platform admin while `OS_ENABLED` is on.

   Record the decisions in `docs/os-adoption.md`.
4. **Fix the security gaps that would be exposed behind the kernel's name** before adding sign-on.
   Examples: unsigned webhooks, signature checks commented out, public diagnostic pages, cron
   reachable from the web, one tenant's secrets used as another's fallback, state changes without
   CSRF. Each fix is its own commit.
5. **Build the adapter** in the order of [references/adapter.md](references/adapter.md):
   1. migration;
   2. token verifiers;
   3. `/sso` and `/sso/logout`;
   4. the guard;
   5. the local account screens under the flag;
   6. the directory sync and scope materialisation;
   7. `maludb-os.json`;
   8. `/api/v1/health`;
   9. `deploy/`.

   The code comes from `../os-integration/references/php-sign-on-kit.md`, renamed to the
   application's conventions. **An application built with `htmx-php-builder` ≤ 0.4.x** has one known
   shape; its deltas and the fixes that worked are [references/htmx-php-builder-profile.md](references/htmx-php-builder-profile.md) —
   read it before the survey, and expect the survey to confirm it.
6. **Prove it without a kernel** (`../os-integration/references/testing-without-a-kernel.md`), plus
   the adoption proofs in `adapter.md` §9. The most important of those is the standalone proof:
   **with `OS_ENABLED` off, the application behaves exactly as before.**
7. **Report.** Record what changed, what is proven, what the owner decided, and what the application
   still owes the rest of the contract. The rest of the contract means the activity log shipped to
   MaluDB, the `mcp_*` views and the MCP servers, and the model keys that should move to the kernel.
   Hand these over by name as the next phase, and do not build substitutes. Commit on the branch and
   leave the merge to the owner.

## Non-negotiables

- Never drop, rename or re-key the application's users table or its tenant table. Link them.
- Never auto-create a user or a tenant from anything but the verified claims or the kernel's feed.
- Never leave a second way in while `OS_ENABLED` is on. Password login, Google sign-in, invitation
  acceptance, password reset and "remember me" all redirect to the launcher. API keys stay, but a key
  whose user the kernel deactivated stops working, because the guard and the API auth both check the link.
- Never let the application create tenants or grant roles while `OS_ENABLED` is on. Its screens say
  "Managed in the operating system" and link to it.
- Never hold the kernel's keys in the repository. `ACTION_TOKEN_KEY` and `OS_APPLICATION_TOKEN` come
  from the installer, in `config`, git-ignored.
- Never change a production server from this skill. The output is a branch; installing is `os-install`.
- Never rewrite the application's handlers to answer JSON. A JSON-mode shim that reads the HTMX answer at shutdown
  (`htmx-php-builder-profile.md`) keeps one code path for people and agents.
