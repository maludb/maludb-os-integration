# Install runbook — the commands

Placeholders used throughout:

| Placeholder | Meaning |
|---|---|
| `K` | The kernel root (`/var/www`) |
| `A` | `/srv/apps/<catalog_key>` |
| `KEY` | The catalog key |
| `SA` | The installing super-admin's member id |
| `DOM` | The business domain |

Commands that write secrets are written so that no value reaches the terminal.

## 1. Code and database

```bash
sudo mkdir -p /srv/apps && sudo git clone <repo> "$A" && sudo git -C "$A" checkout <tag>
sudo chown -R "$USER":www-data "$A"
[ -f "$A/composer.json" ] && (cd "$A" && composer install --no-dev --optimize-autoloader)
# database <tenant>_<suffix> and the three roles from maludb-os.json → database.roles
for r in rw records_ro activity_ro; do openssl rand -hex 24 > "/tmp/$KEY.$r.pw"; chmod 600 "/tmp/$KEY.$r.pw"; done
# CREATE ROLE … LOGIN PASSWORD from those files; CREATE DATABASE … OWNER <rw role>; then, in order:
for f in "$A"/db/*.sql; do sudo -u postgres psql -v ON_ERROR_STOP=1 -d "<db>" -f "$f"; done
```

- The passwords go into `config/.env` in §2, and then `/tmp/$KEY.*.pw` is deleted.
- An adopted application's schema may live elsewhere (`docs/sql/…`). Use the files the manifest names.

**Check:** every migration applied on an empty database, and the `mcp_*` views are granted only to the read roles.

## 2. `config/.env`

```bash
getk() { sudo grep -E "^$1=" "$K/config/.env" | head -1 | cut -d= -f2-; }   # never echo the result
{
  echo "APP_KEY=$KEY"
  echo "APP_URL=https://<label>.$DOM"
  echo "OS_INTERNAL_URL=http://127.0.0.1:8080"
  echo "OS_LAUNCHER_URL=https://app.$DOM/"
  echo "ACTION_TOKEN_KEY=$(getk ACTION_TOKEN_KEY)"
  echo "ACTIONS_RELAY_KEY=$(getk ACTIONS_RELAY_KEY)"
  echo "MALUDB_API_URL=$(getk MALUDB_API_URL)"
  echo "OS_ENABLED=1"                  # adopted applications
  # DB_* and MCP_*_DB_* from §1, ports from §3, the rest of env.required
} | sudo tee "$A/config/.env" >/dev/null
sudo chgrp www-data "$A/config/.env" && sudo chmod 640 "$A/config/.env"
```

- An application that reads `config/local.php` rather than `.env` gets the same keys in its own
  format (`os-adopt` records which).
- The tenant's MaluDB token (`MALUDB_API_TOKEN`) is minted for the application, never copied from
  the kernel's. See `memory.md` in `os-integration`.

## 3. Ports, vhost, services

```bash
ss -ltn | awk 'NR>1{print $4}' | sed 's/.*://' | sort -n | uniq      # taken ports
# the kernel: 8080, 8811-8816; HR: 8180, 8821, 8822 — choose the next free block
sudo cp "$A/deploy/<key>.conf" /etc/apache2/sites-available/<key>.conf   # fill ServerName, ports, paths
sudo a2ensite <key> && sudo apachectl configtest && sudo systemctl reload apache2
for u in <services from maludb-os.json>; do sudo cp "$A/$u" /etc/systemd/system/; done
sudo systemctl daemon-reload && sudo systemctl enable --now <units>
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:<APP_INTERNAL_PORT>/api/v1/health    # 200
```

For each MCP port, send an `initialize` to `http://127.0.0.1:<port>/mcp` with a person's MCP token.
It must answer before the endpoint is registered.

## 4. Registration in the kernel — the actions, as the super-admin

Call the kernel's Actions MCP as the super-admin. It is the same door the kernel's screens and
agents use, so every call is authorised, approval-checked and logged. On the kernel host today, the
harness that does this is `K/mcp/smoke_actions.py`. It takes a scenario file and the acting member,
mints an action token for that member, calls each tool, and runs `lookup` SQL between calls. (When
the kernel's `bin/app_install.php plan|apply` exists, build plan C4, use it instead.)

`register.json` — fill in from `maludb-os.json`. It uses no `{{run}}`, because these are real names.

```json
[
 {"tool": "application_catalog_save", "args": {"catalog_key": "KEY", "name": "Name", "kind": "ours",
   "business_area": "Sales & Service", "description": "…", "icon": "feather-calendar", "category": "calendar"}},
 {"tool": "application_save", "args": {"name": "Name", "app_key": "KEY", "catalog_key": "KEY", "category": "calendar",
   "url": "https://<label>.DOM", "sso_path": "/sso", "sso_logout_path": "/sso/logout",
   "scope_kind": "location", "location": "<the office id it runs on>", "owner_department": "<id>", "version": "1.0.0"}},
 {"lookup": {"app": "SELECT id FROM applications WHERE app_key = 'KEY'"}},
 {"tool": "application_endpoint_save", "args": {"application": "{{app}}", "name": "Records MCP", "kind": "mcp",
   "url": "https://<label>.DOM/mcp/records", "auth_kind": "bearer", "agent_reachable": "1", "mcp_surface_version": "1.0"}},
 {"tool": "application_roles_refresh", "args": {"application": "{{app}}"}},
 {"tool": "application_set_status", "args": {"application": "{{app}}", "status": "active"}}
]
```

```bash
cd "$K/mcp" && venv/bin/python smoke_actions.py run /path/to/register.json "$SA"
```

- **Check:** the application page in the kernel shows the sign-on paths, *Serves*, the endpoints, and the roles **with their rights**, read from the application (0.4.0: the roles come from its `app_roles` tool, which needs the records server running and its endpoint registered first; an application that cannot publish them gets `application_roles_set` instead).
- **Every step must answer `ok`.** A refusal carries the kernel's own sentence. Fix the cause and
  re-run only what is left. `application_save` with the same key refuses, which is idempotency by design.

## 5. The application token — minted straight into the application's config

The token is shown once. Mint it on the kernel host through the kernel's own function, and write it
into the application's `config/.env` in the same command, so it never appears.

```bash
sudo -E php -r '
  require "'"$K"'/app/bootstrap.php";
  require "'"$K"'/app/features/applications/queries.php";
  $pdo = db(); $_SESSION["member_id"] = (int) getenv("SA"); db_apply_context($pdo);
  $app = (int) getenv("APP"); $t = mint_application_token($pdo, $app, (int) getenv("SA"), getenv("KEY") . " (application token)");
  log_activity($pdo, "application.token_mint", "application", $app, ["after" => ["token_id" => $t["id"], "by" => "installer"]]);
  $f = getenv("A") . "/config/.env"; $env = preg_replace("/^OS_APPLICATION_TOKEN=.*\n?/m", "", (string) file_get_contents($f));
  if (file_put_contents($f, rtrim($env) . "\nOS_APPLICATION_TOKEN=" . $t["raw"] . "\n") === false) {
      fwrite(STDERR, "could not write $f — revoke token {$t["id"]} on the application page and run again\n"); exit(1);
  }
  echo "token ", $t["id"], " written\n";'
sudo systemctl restart <the application's units>
```

Run it as root (`sudo -E`, with `SA`, `APP`, `KEY` and `A` exported): the application's `config/.env` is group-readable by www-data and writable by no one else.

- **Check:** `curl -s -H "Authorization: Bearer $(sudo grep ^OS_APPLICATION_TOKEN= "$A/config/.env" | cut -d= -f2-)" http://127.0.0.1:8080/api/v1/directory/scopes.php | head -c 200` answers `os.directory-scopes/1`.
- The same token works with the application's own sync script, so prefer checking through that
  script.

## 6. Scopes and grants (a scoped application)

```json
[
 {"tool": "location_save", "args": {"name": "Airport", "kind": "site", "address": "Terminal B", "timezone": "America/Chicago"}},
 {"lookup": {"app": "SELECT id FROM applications WHERE app_key = 'KEY'",
             "ap": "SELECT id FROM locations WHERE kind = 'site' AND name = 'Airport'"}},
 {"tool": "application_scope_add", "args": {"application": "{{app}}", "location": "{{ap}}"}},
 {"lookup": {"sap": "SELECT id FROM application_scopes WHERE application_id = {{app}} AND location_id = {{ap}} AND removed_at IS NULL"}},
 {"tool": "application_access_grant", "args": {"application": "{{app}}", "residents": "{{ap}}", "scope": "{{sap}}", "roles": "user"}},
 {"tool": "application_access_grant", "args": {"application": "{{app}}", "member": "<the owner-named manager>", "scope": "{{sap}}", "roles": "manager,user"}}
]
```

- Only sites and grants the owner named.
- A site that already exists is looked up, not saved again.
- The residents grant goes with residency: add people to the site on Work Locations
  (`location_add_resident`) and they hold the role there.

Then run the application's sync with `--full`, and check that each scope exists inside the
application with the scope's name and time zone.

## 7. Proofs

1. `GET /api/v1/health` on the application's name: 200, database `ok`.
2. **A real launch.** Mint a person's action token on the kernel
   (`php -r 'require "K/app/bootstrap.php"; echo mint_action_token(<member>, 300);'`) and
   `GET http://127.0.0.1:8080/launch.php?application=<app>` with `X-Action-Token` and
   `Accept: application/json`.
   - The answer's `location` is `https://<label>.DOM/sso?token=…&claims=…`.
   - Follow it with a cookie jar to the application's name. The session opens in the person's scope.
3. **Refusals:**
   - The same `location` a second time is refused (replay).
   - `&scope=<one they do not hold>` gives 404 at the kernel.
   - A member with no grant gives 404.
4. **Revocation:**
   1. Revoke a test grant (`application_access_revoke`).
   2. Wait for one sync.
   3. That person's next request lands on the launcher.
   4. Grant it back if it was real.
5. **Sign-out:** sign out in the kernel. The application's session list shows `ended_by = 'kernel'`.

Record every result in the report, with the kernel ids.
