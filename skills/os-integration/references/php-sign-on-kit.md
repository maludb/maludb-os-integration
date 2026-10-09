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
    // Roles (0.4.0, roles-and-rights.md): every role held, filtered to the ones this application publishes.
    if (array_key_exists('roles', $claims)) {
        mirror_apply_roles($pdo, $memberId, is_array($claims['roles']) ? $claims['roles'] : []);
    }
}

/** members.roles from the claims or an access[] row — only the roles this application offers (app_roles' catalogue). */
function mirror_apply_roles(PDO $pdo, int $memberId, array $roles): void
{
    $known = $pdo->query('SELECT role_key FROM app_roles')->fetchAll(PDO::FETCH_COLUMN);
    $keep = array_values(array_intersect(array_unique(array_map('strval', $roles)), $known));
    $pdo->prepare('UPDATE members SET roles = CAST(:r AS text[]), synced_at = now() WHERE id = :m')
        ->execute(['r' => '{' . implode(',', $keep) . '}', 'm' => $memberId]);
}

/** Replace a member's holding on a scoped application: [{scope_id, role, roles, capability}, …] (roles: 0.4.0). */
function mirror_apply_holding(PDO $pdo, int $memberId, array $scopes): void
{
    $pdo->prepare('DELETE FROM member_scope_roles WHERE member_id = :m')->execute(['m' => $memberId]);
    $ins = $pdo->prepare('INSERT INTO member_scope_roles (member_id, scope_id, role_key, roles, capability)
                          SELECT :m, :s, :r, CAST(:rs AS text[]), :c WHERE EXISTS (SELECT 1 FROM os_scopes WHERE scope_id = :s AND removed_at IS NULL)');
    foreach ($scopes as $s) {
        $roles = is_array($s['roles'] ?? null) ? $s['roles'] : (($s['role'] ?? null) !== null ? [$s['role']] : []);
        $ins->execute(['m' => $memberId, 's' => (int) $s['scope_id'], 'r' => $s['role'] ?? null,
                       'rs' => '{' . implode(',', array_map('strval', $roles)) . '}', 'c' => (string) $s['capability']]);
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
        mirror_apply_roles($pdo, $memberId, ($a['capability'] ?? null) === null ? [] : (array) ($a['roles'] ?? []));   // 0.4.0
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

## 8. The application switcher (`app/switcher.php` + `app/views/shared/app-switcher.php`, 2026-10-09)

`sign-on-and-directory.md` §8 says what it is. One file serves both shapes of application — a kit one (this kit: `env()`,
`current_member_id()`, `kernel_call()`) and an adopted one (`os_enabled()`, `os_launcher_url()`, `current_user()['os_member_id']`,
`kernel_call()` from `os-adopt/adapter.md`). Require it from the bootstrap after the OS helpers; render the partial in the layout
before `div.nxl-h-item.dark-light-theme`: `<?= view('shared/app-switcher.php') ?>`. The CSS is the design system's APPLICATION
SWITCHER section. Proof: the exemplar's `scripts/prove-switcher.sh` (design-system `examples/php/app-switcher/`).

```php
<?php
declare(strict_types=1);

/**
 * The application switcher — the Helpdesk button and the dropdown of the person's applications in every application's
 * header (K31, 2026-10-09; the kernel's docs/build-specs/kernel-app-switcher.md; the integration plugin's
 * sign-on-and-directory.md §8). One file for both shapes of application: an adopted one (`app/os.php`: os_enabled(),
 * os_launcher_url(), kernel_call(), current_user()['os_member_id']) and a kit one (env('OS_APPLICATION_TOKEN'),
 * env('OS_LAUNCHER_URL'), kernel_call(), current_member_id() — the kernel's member id IS the member id). Required by the
 * bootstrap after the OS helpers; rendered by app/views/shared/app-switcher.php.
 */

/** The kernel's id of the signed-in person, or 0 when there is none (standalone, anonymous, an agent's token). */
function os_switcher_member_id(): int
{
    if (function_exists('current_user')) {
        return (int) (current_user()['os_member_id'] ?? 0);
    }
    if (function_exists('current_member_id')) {
        return (int) (current_member_id() ?? 0);
    }
    return 0;
}

/** Is this installation under the kernel? An adopted application says so with OS_ENABLED; a kit one is only ever under it. */
function os_switcher_enabled(): bool
{
    if (function_exists('os_enabled')) {
        return os_enabled();
    }
    return (string) env('OS_APPLICATION_TOKEN', '') !== '';
}

/** The launcher's address (OS_LAUNCHER_URL, the installer's — scheme and all), with a trailing slash. */
function os_switcher_launcher_url(): string
{
    $url = function_exists('os_launcher_url') ? os_launcher_url() : (string) env('OS_LAUNCHER_URL', '');
    return $url === '' ? '' : rtrim($url, '/') . '/';
}

/** A launch path from the kernel's feed (`/launch/<id>`, `?scope=<id>`) on the launcher's address. */
function os_launch_href(string $path): string
{
    return os_switcher_launcher_url() . ltrim($path, '/');
}

/**
 * The applications the signed-in person may open, as the kernel's launcher would list them (GET /api/v1/apps/mine.php as
 * this person): {launcher_url, os_url, applications[{id, key, name, icon, business_area, current, launch_path, scopes[{id,
 * name, role, launch_path}]}]}. Cached in the session for five minutes (a fresh hand-off starts a fresh session, so a new
 * grant shows at the next sign-on at the latest); a kernel that does not answer leaves the last answer in place and is
 * asked again after a minute. Null standalone, for a person the kernel does not know, or before the first answer.
 */
function os_my_applications(bool $refresh = false): ?array
{
    if (!os_switcher_enabled()) {
        return null;
    }
    $memberId = os_switcher_member_id();
    if ($memberId <= 0) {
        return null;
    }
    $cached = $_SESSION['os_apps'] ?? null;
    if (is_array($cached) && (int) ($cached['member'] ?? 0) === $memberId) {
        $fresh = ($cached['at'] ?? 0) > time() - (($cached['feed'] ?? null) === null ? 60 : 300);
        if (!$refresh && $fresh) {
            return $cached['feed'];
        }
    } else {
        $cached = null;
    }
    $answer = kernel_call('GET', '/api/v1/apps/mine.php', null, ['X-Acting-Member: ' . $memberId], 5);
    $feed = $answer !== null && ($answer['status'] ?? 0) === 200 && is_array($answer['body'] ?? null) ? ($answer['body']['data'] ?? $answer['body']) : null;
    if (!is_array($feed) || !isset($feed['applications'])) {
        $_SESSION['os_apps'] = ['member' => $memberId, 'at' => time(), 'feed' => $cached['feed'] ?? null];
        return $cached['feed'] ?? null;
    }
    $_SESSION['os_apps'] = ['member' => $memberId, 'at' => time(), 'feed' => $feed];
    return $feed;
}
```

```php
<?php
/**
 * The application switcher (K31; the design-system's "Header: the application switcher"): in `header-right`, before the
 * dark-mode toggle — the Helpdesk button (when the person holds the Help Desk and this is not it) and the dropdown of every
 * application the person may open, the current one marked, the launcher and (a super-admin) the operating system at the
 * foot. Every link leaves this application, so none is an HTMX swap. Renders nothing standalone or before the kernel answered.
 * @var ?array $feed  os_my_applications()'s answer (looked up when absent)
 */
$feed = $feed ?? os_my_applications();   // app/switcher.php
if (!is_array($feed) || empty($feed['applications'])) {
    return;
}
$apps = $feed['applications'];
$helpdesk = null;
foreach ($apps as $a) {
    if (($a['key'] ?? '') === 'helpdesk' && empty($a['current'])) {
        $helpdesk = $a;
    }
}
$item = static function (array $a, ?array $scope = null): string {
    $id = 'header-apps-item-' . (int) $a['id'] . ($scope !== null ? '-' . (int) $scope['id'] : '');
    $label = e($a['name']) . ($scope !== null ? ' <span class="text-muted">· ' . e($scope['name']) . '</span>' : '');
    $icon = '<i class="' . e($a['icon'] ?? 'feather-grid') . '"></i>';
    if (!empty($a['current'])) {            // this application — every row of it, a scoped one's sites too (its own site switcher changes the site)
        return '<span class="dropdown-item active" id="' . $id . '" aria-current="page">' . $icon . '<span>' . $label . '</span></span>';
    }
    return '<a href="' . e(os_launch_href(($scope ?? $a)['launch_path'])) . '" class="dropdown-item" id="' . $id . '">' . $icon . '<span>' . $label . '</span></a>';
};
?>
<?php if ($helpdesk !== null): ?>
<div class="nxl-h-item">
    <a href="<?= e(os_launch_href($helpdesk['launch_path'])) ?>" class="nxl-head-link me-0 header-helpdesk" id="header-helpdesk-btn" title="Help Desk"><i class="<?= e($helpdesk['icon'] ?? 'feather-life-buoy') ?>"></i><span class="d-none d-sm-inline">Helpdesk</span></a>
</div>
<?php endif; ?>
<div class="dropdown nxl-h-item">
    <a href="javascript:void(0);" class="nxl-head-link me-0" data-bs-toggle="dropdown" role="button" data-bs-auto-close="outside" id="header-apps-toggle" aria-label="Your applications" title="Your applications"><i class="feather-grid"></i></a>
    <div class="dropdown-menu dropdown-menu-end nxl-h-dropdown header-apps-menu" id="header-apps-menu">
        <div class="dropdown-header"><h6 class="text-dark mb-0">Your applications</h6></div>
        <div class="dropdown-divider"></div>
        <div class="header-apps-list" id="header-apps-list">
        <?php foreach ($apps as $a): ?>
            <?php if (!empty($a['scopes'])): foreach ($a['scopes'] as $s): ?><?= $item($a, $s) ?><?php endforeach; else: ?><?= $item($a) ?><?php endif; ?>
        <?php endforeach; ?>
        </div>
        <div class="dropdown-divider"></div>
        <a href="<?= e(os_switcher_launcher_url()) ?>" class="dropdown-item" id="header-apps-launcher"><i class="feather-layout"></i><span>All applications</span></a>
        <?php if (!empty($feed['os_url'])): ?><a href="<?= e($feed['os_url']) ?>" class="dropdown-item" id="header-apps-os"><i class="feather-cpu"></i><span>Operating system</span></a><?php endif; ?>
    </div>
</div>
```
