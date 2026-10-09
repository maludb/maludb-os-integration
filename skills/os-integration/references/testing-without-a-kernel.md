# Building on a server with no kernel

An application from us is **installed** on the kernel's server, because the directory, ledger, chat and
run-facts endpoints answer on the kernel's loopback port only. It can be **built** anywhere. This page
shows how to prove the sign-on and scope handling on a development server that has no kernel, so that
nothing needs changing when the installer puts the application beside one.

## 1. A development key

Put a development `ACTION_TOKEN_KEY` in the application's `config/.env`: any 64 hex characters,
`openssl rand -hex 32`. It is **not** the tenant's key. At install, the installer overwrites it with
the kernel's key, so a development token can never open a production installation.

## 2. Mint a hand-off the way the kernel does (`bin/dev_handoff.php`, never deployed to `html/`)

```php
<?php
// php bin/dev_handoff.php <member_id> [scope_id] — prints a /sso URL signed exactly as the kernel signs one.
// Development only: the kernel's own mint is mint_sso_token()/sign_sso_claims() in its app/auth.php.
require dirname(__DIR__) . '/app/bootstrap.php';
if (PHP_SAPI !== 'cli') { exit(1); }
$member = (int) ($argv[1] ?? 1);
$scope = isset($argv[2]) ? (int) $argv[2] : null;
$key = action_token_key();
$app = (string) env('APP_KEY');
$payload = $member . '.' . (time() + 60) . '.' . $app . '.' . bin2hex(random_bytes(16));
$token = $payload . '.' . hash_hmac('sha256', 'sso:' . $payload, $key);
$fixture = json_decode((string) file_get_contents(__DIR__ . '/dev_directory.json'), true);   // §3
$claims = $fixture['claims'][(string) $member] + ['member_id' => $member, 'scope' => $scope];
$text = rtrim(strtr(base64_encode(json_encode($claims)), '+/', '-_'), '=');
echo rtrim((string) env('APP_URL'), '/') . '/sso?' . http_build_query(['token' => $token, 'claims' => $text . '.' . hash_hmac('sha256', $text, $key)]) . "\n";
```

The sign-out notice is the same shape: `"{member}.{time()}.{app}"`, then `.` and `hash_hmac('sha256', 'sso-logout:' . that, key)`. POST it as `notice=` to `/sso/logout`.

## 3. A directory fixture (`bin/dev_directory.json`)

The fixture holds the claims per member and a feed document in the kernel's exact schema.

```json
{
  "claims": {
    "1":  {"display_name": "Owner", "email": "owner@example.invalid", "business_role": "super_admin", "is_external": false,
           "status": "active", "departments": [], "capability": "admin", "role": null,
           "scopes": [{"scope_id": 1, "kind": "location", "id": 10, "name": "Downtown", "role": "admin", "capability": "admin"},
                      {"scope_id": 2, "kind": "location", "id": 11, "name": "Airport",  "role": "admin", "capability": "admin"}]},
    "26": {"display_name": "Priya", "email": "priya@example.invalid", "business_role": "user", "is_external": false,
           "status": "active", "departments": [], "capability": "write", "role": null,
           "scopes": [{"scope_id": 2, "kind": "location", "id": 11, "name": "Airport", "role": "user", "capability": "write"}]}
  },
  "feed": {
    "schema": "os.directory-changes/1", "since": null, "next": "2026-01-01T00:00:00.000000Z", "full": true,
    "members": [], "departments": [], "memberships": [], "deleted_departments": [],
    "scopes": [
      {"scope_id": 1, "kind": "location", "location_id": 10, "department_id": null, "name": "Downtown",
       "address": "1 Main St", "timezone": "America/New_York", "removed_at": null, "updated_at": "2026-01-01T00:00:00Z"},
      {"scope_id": 2, "kind": "location", "location_id": 11, "department_id": null, "name": "Airport",
       "address": "Terminal B", "timezone": "America/Chicago", "removed_at": null, "updated_at": "2026-01-01T00:00:00Z"}],
    "access": [
      {"member_id": 1,  "role": null, "capability": "admin", "scopes": [{"scope_id": 1, "role": "admin", "capability": "admin"}, {"scope_id": 2, "role": "admin", "capability": "admin"}]},
      {"member_id": 26, "role": null, "capability": "write", "scopes": [{"scope_id": 2, "role": "user", "capability": "write"}]}]
  }
}
```

Make `directory_sync.php` accept `--from-file bin/dev_directory.json` and apply `feed` with the same functions it uses for the kernel's answer. To test a revocation, edit the fixture so Priya holds `"scopes": []` and `"capability": null`, then run the sync again.

## 3b. The application switcher without a kernel

`os_my_applications()` asks the kernel as the person; without one it returns null and the header shows nothing. The exemplar's
proof (`prove-switcher.sh`) asks the REAL kernel of the host with the install's own token — a read — because the feed is the
kernel's judgement of who may open what; on a server with no kernel, prove the partial from a fixture instead: call
`view('shared/app-switcher.php', ['feed' => <a decoded os.my-applications/1 document>])` and check the button, the rows, the
marked current application and the foot, as the exemplar's in-process checks do.

## 4. The proofs to run before calling it ready

1. The `dev_handoff.php 26` URL opens a session at Airport. The same URL a second time is refused (replay), and so is the same URL after 61 s.
2. A token minted with `APP_KEY=other` is refused (audience).
3. `dev_handoff.php 26 1` names Downtown, which Priya does not hold. The session opens at Airport, the one scope she holds, and Downtown's data is unreachable.
4. After the revocation sync, Priya's next request lands on the launcher, and `member_sessions` shows `ended_by = 'directory'`.
5. The sign-out notice ends every session of the member and answers 204. A bad notice also answers 204 and changes nothing.

When the application is installed beside a kernel, the same proofs run with the kernel's real tokens (`os-install`).
