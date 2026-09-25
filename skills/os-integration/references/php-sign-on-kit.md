# The PHP sign-on kit — code to copy

*Proven code, copied from the kernel's `app/auth.php` (the minting side) and from HR (`/srv/apps/hr`, the
first application from us, the receiving side), with the scope handling of 2026-09-25 added.* Use it as
written. It is not a sketch. Names follow the `htmx-php-builder` conventions (`db()`, `env()`,
`log_activity()`, `view()`). An adopted application renames what it must and keeps the order of the checks.

## 1. The verifiers (`app/os_tokens.php`)

```php
<?php
declare(strict_types=1);

/** The tenant's ACTION_TOKEN_KEY — the same value as the kernel's config/.env. */
function action_token_key(): string
{
    $k = (string) env('ACTION_TOKEN_KEY', '');
    if (strlen($k) < 32) {
        throw new RuntimeException('ACTION_TOKEN_KEY is not configured.');
    }
    return $k;
}

function base64url_decode(string $text): string|false
{
    return base64_decode(strtr($text, '-_', '+/') . str_repeat('=', (4 - strlen($text) % 4) % 4), true);
}

/** [member_id, nonce] for a valid hand-off token bound to THIS application, else null. */
function verify_sso_token(string $token, string $expectedAppKey): ?array
{
    $parts = explode('.', $token);
    if (count($parts) !== 5) {
        return null;
    }
    [$mid, $exp, $app, $nonce, $mac] = $parts;
    if (!ctype_digit($mid) || !ctype_digit($exp) || (int) $exp < time() || $app !== $expectedAppKey
        || !preg_match('/^[a-f0-9]{32}$/', $nonce)) {
        return null;
    }
    $expected = hash_hmac('sha256', 'sso:' . $mid . '.' . $exp . '.' . $app . '.' . $nonce, action_token_key());
    return hash_equals($expected, $mac) ? [(int) $mid, $nonce] : null;
}

/** The signed claims beside the token: base64url(json) . '.' . hmac(base64url text). */
function verify_sso_claims(string $signed): ?array
{
    $dot = strrpos($signed, '.');
    if ($dot === false) {
        return null;
    }
    $text = substr($signed, 0, $dot);
    $mac = substr($signed, $dot + 1);
    if (!hash_equals(hash_hmac('sha256', $text, action_token_key()), $mac)) {
        return null;
    }
    $json = base64url_decode($text);
    $claims = $json === false ? null : json_decode($json, true);
    return is_array($claims) ? $claims : null;
}

/** The kernel's sign-out notice: {member}.{issued}.{app}.{hmac}, 120 s. Member id or null. */
function verify_sso_logout_notice(string $notice, string $expectedAppKey, int $ttlSeconds = 120): ?int
{
    $parts = explode('.', $notice);
    if (count($parts) !== 4) {
        return null;
    }
    [$mid, $issued, $app, $mac] = $parts;
    if (!ctype_digit($mid) || !ctype_digit($issued) || $app !== $expectedAppKey || abs(time() - (int) $issued) > $ttlSeconds) {
        return null;
    }
    $expected = hash_hmac('sha256', 'sso-logout:' . $mid . '.' . $issued . '.' . $app, action_token_key());
    return hash_equals($expected, $mac) ? (int) $mid : null;
}
```

The minting side is **not** copied into an application. Only the kernel mints. For tests, see `testing-without-a-kernel.md`.

## 2. The tables (`db/0NN_os_identity.sql`)

A new application uses these as its identity. An adopted application adds them beside its own users table, and links that table with `os_member_id` (`os-adopt`).

```sql
CREATE TABLE sso_nonces (                       -- single use: kept until the token would have expired
    nonce       text PRIMARY KEY,
    member_id   bigint NOT NULL,
    expires_at  timestamptz NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX sso_nonces_expires_idx ON sso_nonces (expires_at);

CREATE TABLE member_sessions (                  -- every session opened, so a sign-out notice ends them all
    session_hash  text PRIMARY KEY,             -- sha256 of the PHP session id
    member_id     bigint NOT NULL,
    created_at    timestamptz NOT NULL DEFAULT now(),
    last_seen_at  timestamptz NOT NULL DEFAULT now(),
    ended_at      timestamptz,
    ended_by      text CHECK (ended_by IN ('member', 'kernel', 'expired', 'directory'))
);
CREATE INDEX member_sessions_member_idx ON member_sessions (member_id) WHERE ended_at IS NULL;

CREATE TABLE directory_sync_state (             -- the change feed's cursor, one row
    id           smallint PRIMARY KEY DEFAULT 1 CHECK (id = 1),
    next_cursor  text,
    full_at      timestamptz,
    last_run_at  timestamptz,
    last_error   text,
    updated_at   timestamptz NOT NULL DEFAULT now()
);
INSERT INTO directory_sync_state (id) VALUES (1) ON CONFLICT DO NOTHING;
```

Add `members`/`department_members` (`sign-on-and-directory.md` §3) and, when scoped, `os_scopes`/`member_scope_roles` (`scoped-applications.md` §3).

## 3. The session list (`app/os_sessions.php`)

```php
function session_hash(string $sessionId): string { return hash('sha256', $sessionId); }

function session_open(PDO $pdo, int $memberId, string $sessionId): void
{
    $pdo->prepare('INSERT INTO member_sessions (session_hash, member_id) VALUES (:h, :m)
                   ON CONFLICT (session_hash) DO UPDATE SET member_id = EXCLUDED.member_id, ended_at = NULL, ended_by = NULL, last_seen_at = now()')
        ->execute(['h' => session_hash($sessionId), 'm' => $memberId]);
}

/** Call on every request after session_start(): a session the kernel ended is over here too. */
function session_is_listed(PDO $pdo, string $sessionId): bool
{
    $st = $pdo->prepare('SELECT 1 FROM member_sessions WHERE session_hash = :h AND ended_at IS NULL');
    $st->execute(['h' => session_hash($sessionId)]);
    return $st->fetchColumn() !== false;
}

function end_member_sessions(PDO $pdo, int $memberId, string $by): int
{
    $st = $pdo->prepare('UPDATE member_sessions SET ended_at = now(), ended_by = :by WHERE member_id = :m AND ended_at IS NULL');
    $st->execute(['by' => $by, 'm' => $memberId]);
    return $st->rowCount();
}
```

## 4. The receiver (`html/sso.php`)

```php
<?php
declare(strict_types=1);
/** /sso — the kernel's hand-off. Checks in order; any failure is ONE page that never says which. */
require_once dirname(__DIR__) . '/app/bootstrap.php';

$pdo = db();
$token = (string) ($_GET['token'] ?? '');
$claimsRaw = (string) ($_GET['claims'] ?? '');
$refuse = static function (string $reason, ?int $memberId = null) use ($pdo): never {
    log_activity($pdo, 'member.sign_on.refused', 'member', $memberId, ['actor_member_id' => null, 'after' => ['reason' => $reason]]);
    http_response_code(403);
    header('Cache-Control: no-store');
    echo view('sso/refused.php', ['launcher' => env('OS_LAUNCHER_URL')]);   // "This sign-on link has expired. Open the application from the launcher again."
    exit;
};

$verified = $token === '' ? null : verify_sso_token($token, (string) env('APP_KEY'));
if ($verified === null) { $refuse('token'); }
[$memberId, $nonce] = $verified;
$claims = $claimsRaw === '' ? null : verify_sso_claims($claimsRaw);
if ($claims === null || (int) ($claims['member_id'] ?? 0) !== $memberId) { $refuse('claims', $memberId); }
if (($claims['status'] ?? 'active') !== 'active') { $refuse('status', $memberId); }
if (!in_array($claims['capability'] ?? null, ['read', 'write', 'admin'], true)) { $refuse('capability', $memberId); }

$claimed = $pdo->prepare('INSERT INTO sso_nonces (nonce, member_id, expires_at) VALUES (:n, :m, to_timestamp(:e)) ON CONFLICT (nonce) DO NOTHING');
$claimed->execute(['n' => $nonce, 'm' => $memberId, 'e' => (int) explode('.', $token)[1]]);
if ($claimed->rowCount() !== 1) { $refuse('replay', $memberId); }
$pdo->exec("DELETE FROM sso_nonces WHERE expires_at < now() - interval '1 hour'");

$pdo->beginTransaction();
try {
    mirror_apply_member_from_claims($pdo, $memberId, $claims);        // §6: members, departments, capability, role
    mirror_apply_holding($pdo, $memberId, $claims['scopes'] ?? []);   // §6: scoped applications only
    $pdo->commit();
} catch (Throwable $e) {
    $pdo->rollBack();
    error_log('sso mirror: ' . $e->getMessage());
    $refuse('mirror', $memberId);
}

session_regenerate_id(true);
$_SESSION = ['member_id' => $memberId, 'csrf_token' => bin2hex(random_bytes(32)), 'signed_on_at' => time(),
             'scope_id' => isset($claims['scope']) ? (int) $claims['scope'] : null];   // scoped: the chosen scope
session_open($pdo, $memberId, session_id());
log_activity($pdo, 'member.sign_on', 'member', $memberId, ['after' => ['capability' => $claims['capability'], 'scope_id' => $_SESSION['scope_id']]]);
header('Cache-Control: no-store');
header('Location: /', true, 302);
exit;
```

The session cookie follows `php-session-auth`: `HttpOnly`, `Secure` and `SameSite=Lax`, for this exact host only.

## 5. The sign-out receiver (`html/sso/logout.php`)

```php
<?php
declare(strict_types=1);
/** /sso/logout — end every session of the member; 204 whether or not the notice verified. */
require_once dirname(__DIR__, 2) . '/app/bootstrap.php';
if (($_SERVER['REQUEST_METHOD'] ?? 'GET') !== 'POST') { header('Allow: POST'); http_response_code(405); exit; }
$pdo = db();
$notice = (string) ($_POST['notice'] ?? '');
if ($notice === '' && is_array($body = json_decode((string) file_get_contents('php://input'), true))) {
    $notice = (string) ($body['notice'] ?? '');
}
$memberId = $notice === '' ? null : verify_sso_logout_notice($notice, (string) env('APP_KEY'));
if ($memberId !== null) {
    $ended = end_member_sessions($pdo, $memberId, 'kernel');
    log_activity($pdo, 'member.sign_out', 'member', $memberId, ['after' => ['by' => 'kernel', 'sessions' => $ended]]);
}
header_remove('Set-Cookie');
http_response_code(204);
```

The Apache vhost maps `/sso` and `/sso/logout` to these files. The logout path is a POST from the kernel's host, and needs no CSRF token because the signed notice is the authority.

## 6. Applying claims and the feed (`app/os_directory.php`)

```php
/** A GET against the kernel's internal port with the application token. Null when unreachable. */
function kernel_get(string $path): ?array
{
    $ch = curl_init(rtrim((string) env('OS_INTERNAL_URL'), '/') . $path);
    curl_setopt_array($ch, [CURLOPT_RETURNTRANSFER => true, CURLOPT_CONNECTTIMEOUT => 3, CURLOPT_TIMEOUT => 15,
        CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . env('OS_APPLICATION_TOKEN'), 'Accept: application/json']]);
    $body = curl_exec($ch);
    $code = (int) curl_getinfo($ch, CURLINFO_HTTP_CODE);
    return $body !== false && $code === 200 ? (json_decode((string) $body, true) ?: null) : null;
}

/**
 * The mirror row from a hand-off's claims — the only place, with the feed's access[], that a capability
 * is written. The claims carry no job title, phone or time zone; the feed's members[] fills those, so an
 * existing row keeps what it had.
 */
function mirror_apply_member_from_claims(PDO $pdo, int $memberId, array $claims): void
{
    $pdo->prepare(<<<'SQL'
        INSERT INTO members (id, member_kind, display_name, email, business_role, is_external, status, capability, synced_at)
        VALUES (:id, 'human', :name, :email, :role, :ext, 'active', :cap, now())
        ON CONFLICT (id) DO UPDATE SET display_name = EXCLUDED.display_name, email = EXCLUDED.email,
            business_role = EXCLUDED.business_role, is_external = EXCLUDED.is_external, status = 'active',
            capability = EXCLUDED.capability, synced_at = now()
    SQL)->execute([
        'id' => $memberId,
        'name' => (string) ($claims['display_name'] ?? ('Member #' . $memberId)),
        'email' => ($claims['email'] ?? '') !== '' ? (string) $claims['email'] : null,
        'role' => in_array($claims['business_role'] ?? 'user', ['super_admin', 'dept_admin', 'user'], true) ? $claims['business_role'] : 'user',
        'ext' => !empty($claims['is_external']) ? 't' : 'f',
        'cap' => (string) $claims['capability'],
    ]);
    // Departments: replace the live memberships with the claims' list (the kernel's ids; name kept for display).
    $pdo->prepare('UPDATE department_members SET left_at = now() WHERE member_id = :m AND left_at IS NULL')->execute(['m' => $memberId]);
    foreach (is_array($claims['departments'] ?? null) ? $claims['departments'] : [] as $d) {
        $pdo->prepare('INSERT INTO departments (id, name) VALUES (:id, :n) ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name')
            ->execute(['id' => (int) $d['id'], 'n' => (string) $d['name']]);
        $pdo->prepare('INSERT INTO department_members (member_id, department_id, is_admin, left_at) VALUES (:m, :d, :a, NULL)
                       ON CONFLICT (member_id, department_id) DO UPDATE SET is_admin = EXCLUDED.is_admin, left_at = NULL')
            ->execute(['m' => $memberId, 'd' => (int) $d['id'], 'a' => !empty($d['is_admin']) ? 't' : 'f']);
    }
    // An unscoped application with its own roles: keep $claims['role'] in your own member-role column.
}

/** Replace a member's holding on a scoped application: [{scope_id, role, capability}, …]. */
function mirror_apply_holding(PDO $pdo, int $memberId, array $scopes): void
{
    $pdo->prepare('DELETE FROM member_scope_roles WHERE member_id = :m')->execute(['m' => $memberId]);
    $ins = $pdo->prepare('INSERT INTO member_scope_roles (member_id, scope_id, role_key, capability)
                          SELECT :m, :s, :r, :c WHERE EXISTS (SELECT 1 FROM os_scopes WHERE scope_id = :s AND removed_at IS NULL)');
    foreach ($scopes as $s) {
        $ins->execute(['m' => $memberId, 's' => (int) $s['scope_id'], 'r' => $s['role'] ?? null, 'c' => (string) $s['capability']]);
    }
}

/** One scopes[] row: upsert the scope, then create or rename the application's own tenant for it. */
function mirror_apply_scope(PDO $pdo, array $s): void
{
    $pdo->prepare('INSERT INTO os_scopes (scope_id, kind, location_id, department_id, name, address, timezone, removed_at, synced_at)
                   VALUES (:id, :k, :l, :d, :n, :a, :t, :r, now())
                   ON CONFLICT (scope_id) DO UPDATE SET name = EXCLUDED.name, address = EXCLUDED.address, timezone = EXCLUDED.timezone,
                                                       removed_at = EXCLUDED.removed_at, synced_at = now()')
        ->execute(['id' => $s['scope_id'], 'k' => $s['kind'], 'l' => $s['location_id'], 'd' => $s['department_id'],
                   'n' => $s['name'], 'a' => $s['address'] ?? null, 't' => $s['timezone'] ?? null, 'r' => $s['removed_at'] ?? null]);
    if (($s['removed_at'] ?? null) !== null) {
        $pdo->prepare('DELETE FROM member_scope_roles WHERE scope_id = :s')->execute(['s' => $s['scope_id']]);
    }
    app_materialise_scope($pdo, $s);          // YOUR function: create/rename the restaurant (or project space) for this scope
}

/** The feed's access[] rows: each replaces the member's holding. Nothing held → sessions end. */
function mirror_apply_access(PDO $pdo, array $rows): void
{
    foreach ($rows as $a) {
        $memberId = (int) $a['member_id'];
        $known = $pdo->prepare('SELECT 1 FROM members WHERE id = :m');
        $known->execute(['m' => $memberId]);
        if ($known->fetchColumn() === false) {
            continue;                         // never auto-create: the first hand-off (or members[]) makes the row
        }
        $pdo->prepare('UPDATE members SET capability = :c, synced_at = now() WHERE id = :m')
            ->execute(['c' => $a['capability'] ?? null, 'm' => $memberId]);
        mirror_apply_holding($pdo, $memberId, $a['scopes'] ?? []);
        if (($a['capability'] ?? null) === null) {
            end_member_sessions($pdo, $memberId, 'directory');
        }
    }
}
```

In `bin/directory_sync.php`, which a systemd timer runs every minute, the apply order is:
1. `members[]`, then `departments[]` twice, then `memberships[]` (as HR does);
2. `scopes[]`, then `access[]`;
3. store `next` in `directory_sync_state`, all in one transaction, under `pg_try_advisory_lock(hashtext('directory_sync'))`.

`--full` drops the cursor. On the first run, call `scopes.php` too, so every scope exists before anyone signs in.

## 7. The guard every page calls

```php
function to_launcher(): never
{
    $_SESSION = [];
    header('Location: ' . env('OS_LAUNCHER_URL') . '?app=' . rawurlencode((string) env('APP_KEY')));
    exit;
}

function require_member(PDO $pdo): array
{
    if (empty($_SESSION['member_id']) || !session_is_listed($pdo, session_id())) {
        to_launcher();
    }
    $memberId = (int) $_SESSION['member_id'];
    $st = $pdo->prepare("SELECT * FROM members WHERE id = :m AND status = 'active' AND capability IS NOT NULL");
    $st->execute(['m' => $memberId]);
    $member = $st->fetch() ?: null;
    if ($member === null) {
        end_member_sessions($pdo, $memberId, 'directory');
        to_launcher();
    }
    // Scoped: the current scope must still be held and open; else the first one that is, else out.
    $held = $pdo->prepare('SELECT r.scope_id FROM member_scope_roles r JOIN os_scopes s USING (scope_id)
                            WHERE r.member_id = :m AND s.removed_at IS NULL ORDER BY s.name');
    $held->execute(['m' => $memberId]);
    $scopes = array_map('intval', $held->fetchAll(PDO::FETCH_COLUMN));
    if (!in_array((int) ($_SESSION['scope_id'] ?? 0), $scopes, true)) {
        if ($scopes === []) {
            end_member_sessions($pdo, $memberId, 'directory');
            to_launcher();
        }
        $_SESSION['scope_id'] = $scopes[0];
    }
    return $member;
}
```
