---
name: os-install
description: Install an application from its repository onto a server that runs the MaluDB Business OS kernel, and integrate it — database, config, Apache virtual host, services, registration in the kernel (catalog entry, application, roles, endpoints, application token), scopes and per-scope grants for a multi-site or multi-department application, the action registry, skills, the proposed expert — then prove sign-on end to end. Use when the user says "install <app> on the OS", "deploy <repo> into the business OS", "register this application with the kernel", or gives a repository URL to put beside a kernel. The repository must carry a maludb-os.json (built with os-integration, or adopted with os-adopt).
---

# Installing an application beside the kernel

This skill runs **on the kernel's server**. An application from us lives at
`/srv/apps/<catalog_key>`, with its own database on the tenant's PostgreSQL 17, its own Apache virtual
host at `<label>.<domain>`, and its own services. It reaches the kernel only through the kernel's
loopback port (`http://127.0.0.1:8080`) and MaluDB. The kernel reaches it through MCP, its internal
port and the sign-on paths. Everything the installer does is logged in the kernel as the member
performing it.

Read `../os-integration/references/registration.md` (the manifest, field by field) before starting.
For a scoped application, also read `../os-integration/references/scoped-applications.md`. The
step-by-step commands are in [references/runbook.md](references/runbook.md).

## Before anything

1. **Confirm the target.** Check that `/var/www` (or the kernel root the user names) is a kernel.
   It has `app/features/applications/sso.php` and `bin/import_skill.php`, and
   `systemctl is-active certstudy-actions-mcp` answers `active`. Also confirm who is installing: a
   super-admin's member id.
2. **Read the repository before installing anything.** Read `maludb-os.json` in full, check its
   `schema` is `maludb-os.application/1`, and refuse the install without `sso`. Read every migration
   it will run, the systemd units it will install, and the vhost. You install what you have read.
3. **Collect what only the owner knows.** Present these as one list and wait:
   - the business domain (DNS for `<label>.<domain>` is the owner's to create);
   - for a scoped application, the sites or departments it serves and whether any sites are new;
   - who gets which role where;
   - approval to proceed.

   Never guess a grant.
4. **Ask before anything destructive.** That covers an existing `/srv/apps/<key>`, an existing
   database of that name, an enabled vhost of that name, or a port already taken. Stop and ask.

## The installer does the order below (2026-09-28)

The kernel carries `bin/app_install.php plan|apply <repository>` (runbook §4): `plan` is read-only and
says, step by step, what is done and what would be done; `apply` (root) does what is not yet done, in
this order, idempotently. Run `plan`, read it with the owner, then `apply`; run `plan` again as the
check. The steps below remain the reference for what each one means and for a kernel too old to carry
the installer.

## The order (each step ends with its check; do not continue past a failed check)

1. Clone to `/srv/apps/<catalog_key>` at the release tag or commit the owner names. Install its dependencies.
2. Database, the three roles and the migrations, in order, as `postgres`, on an empty database.
3. `config/.env` from `env.required`:
   - `ACTION_TOKEN_KEY` and `ACTIONS_RELAY_KEY` copied from the kernel's `config/.env`, **never printed**;
   - `OS_INTERNAL_URL`, `OS_LAUNCHER_URL`, `APP_KEY`, and the tenant's `MALUDB_*`;
   - `OS_ENABLED=1` for an adopted application;
   - the file is readable by group `www-data` only.
4. Ports:
   - pick free ones (`ss -ltn`) for `APP_INTERNAL_PORT` and each MCP port;
   - the kernel owns 8080 and 8811–8816;
   - record what other applications hold.
5. The Apache vhost from `deploy/`, with the domain filled in:
   - `apachectl configtest`, then reload;
   - `/etc/hosts` gets `127.0.0.1 <label>.<domain>` only if the owner's DNS does not exist yet, and say so.
6. Services and timers from `services`. Enable, start, and wait until each MCP server answers
   `initialize` on its port. `/api/v1/health` answers 200.
7. **Register in the kernel** (runbook §4):
   1. the catalog entry, kind `ours`;
   2. the application with `url`, the sign-on paths, `directory_writes` and `scope_kind`;
   3. its roles;
   4. its endpoints, only those that now answer;
   5. status `active`.
8. **The application token:** mint it and write it into the application's `config/.env` in one step,
   so it never reaches the screen or the transcript (runbook §5). Restart the services.
9. **Scoped:**
   1. create any new site;
   2. add the scopes;
   3. make the owner's grants, each naming a scope and a role;
   4. run the application's directory sync once with `--full`;
   5. check each scope exists inside the application.
10. Copy the action registry to `mcp/registries/<app_key>.json` in the kernel and restart
    `certstudy-actions-mcp`. Import and assign each skill at application scope.
11. **Propose the agents.** Write each `agents[]` entry up as a proposal for the super-admin: the
    job description, the tool grants and, on a scoped application, the scopes it should be granted.
    Do not hire. Hiring is the owner's one click in Agent HR.
12. **Prove it** (runbook §7):
    - the health endpoint;
    - a real launch as a granted person, which lands in their scope;
    - a scope they do not hold, refused;
    - the replay of their token, refused;
    - a revocation, reflected in the application within one sync.
13. **Report:**
    - the URL and what was installed where;
    - the ports;
    - the kernel ids (application, endpoints, scopes, grants);
    - each proof and its result;
    - the owner's remaining steps (DNS, TLS, hiring the expert, third-party consoles);
    - where the secrets are, without their values.

## Non-negotiables

- Never print, log or commit a secret. That means the tenant's keys, the application token, and
  database passwords. Generate with `openssl rand`, write straight into the file, and name the file.
- Never register an endpoint before its server answers. The grant picker offers whatever is registered.
- Never connect the application to the kernel's database, or the kernel to the application's.
- Never make a grant the owner did not name. Application users are not OS users: nobody is granted by default.
- Never run the kernel's own deploy (`web/scripts/deploy.sh`) or change the kernel's code from this skill.
- Stop and hand over anything only the owner can do: DNS, certificates for a real domain, and
  third-party consoles (voice, SMS and mail providers pointing at the new name).
