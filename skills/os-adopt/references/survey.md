# The adoption survey

Answer every question with evidence (`file:line`, a grep count, the `CREATE TABLE`). Write the answers into `docs/os-adoption.md`. Where the code disagrees with the documentation, the code wins, and say so.

## 1. Stack
- The PHP version, the web root and the routing: page-per-file, a front controller, or HTMX partials.
- The database (PostgreSQL? which version?) and where the schema lives. Several schema files may exist: which one is current?
- Where configuration comes from (`.env`, `config/local.php`, environment), and what is hard-coded that differs per server.

## 2. Every way into a session
- The **login function**: the one place that fills the session. List every session key it sets (`$_SESSION['user']`, `user_id`, `current_tenant_id`, `role`, …).
- Every caller of it: the password form, Google/OAuth callbacks, registration, invitation acceptance, password reset, "remember me".
- Other session writers: impersonation, tenant switching, anything that reads a cookie and logs someone in.
- The **guard**: the function pages call (`requireAuth()`). Count its callers. Note what it does for an HTMX request (`HX-Redirect`) and for a normal one.
- Pages with no guard. Decide for each: public on purpose (a booking page, a webhook), or a gap.

## 3. People and tenants
- The users table: its columns, and how the password, provider and role are held.
- The **tenant table** (restaurants, organisations, accounts) and the **membership table** (`user_restaurants(user_id, restaurant_id, role)`): the roles in use, and whether a user can belong to several tenants.
- The platform-level role, if any (a super-admin across tenants).
- The screens that create users, invite them, change roles, deactivate them, and create tenants. These become read-only under `OS_ENABLED`.
- How a tenant is chosen for a request (a session key, a switcher, a subdomain).

## 4. Machine access
- API keys or bearer tokens: where they are checked, and whether they depend on the user being active.
- MCP servers the application already runs: how they authenticate, what they expose.
- Webhooks and third-party callbacks: which verify a signature and which do not.
- Cron and CLI scripts, and whether any are reachable from the web.

## 5. What it holds that the kernel should
- Model keys (OpenAI, Anthropic, voice providers) and where they are stored. The contract says an application never holds a model key. Record them, and leave the move to the owner's decision.
- An activity or audit log: its shape, and whether one function writes it.
- Outbound mail and SMS.

## 6. Security gaps
List anything that would be exposed once the application sits behind the business's name:
- signature checks disabled;
- unsigned webhooks;
- public diagnostic pages (`phpinfo`, database tests);
- web-reachable cron;
- one tenant's secret used as another's fallback;
- state changes without CSRF;
- request bodies with personal data written to logs.

## 7. The decisions to ask for
From the survey, propose each of these with a recommendation, and let the owner decide:
- scope kind;
- roles and capabilities;
- the admin role;
- the platform super-admin mapping;
- which catalog key and DNS label (`reservations` → `reservations.<domain>`);
- the business area.
