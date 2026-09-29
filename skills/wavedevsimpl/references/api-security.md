# JSON API and security

Reference code: youlvat [`api.php`](https://github.com/breizhwave/youlvat/blob/main/api.php), [`lib/auth.php`](https://github.com/breizhwave/youlvat/blob/main/lib/auth.php), [`lib/tables.php`](https://github.com/breizhwave/youlvat/blob/main/lib/tables.php), [`lib/html.php`](https://github.com/breizhwave/youlvat/blob/main/lib/html.php).

## One endpoint: `api.php`

- `GET ?action=all` → the whole state the UI needs, in one request. Fine while data fits in a few hundred KB.
- `POST {action: create|update|delete, table, id?, data}` → generic CRUD, **admin only**.
- Dedicated business actions for anything public or delicate (`signup`…), each with its own checks.
- `POST {action: login|logout}`.

Rules:
- **Whitelist** of tables and writable columns (`lib/tables.php`), with required and integer columns. Nothing is interpolated into SQL without going through it.
- **Prepared statements only.**
- Front-end speaks camelCase, DB speaks snake_case; convert in the API, in one place:

```php
function snake(string $k): string { return strtolower(preg_replace('/[A-Z]/', '_$0', $k)); }
function camel(string $k): string { return lcfirst(str_replace('_', '', ucwords($k, '_'))); }
```

- Validate every field server-side (date/time formats, `https://` URLs, phone, e-mail, length).
- Responses: `Content-Type: application/json; charset=utf-8`, `Cache-Control: no-store`, `JSON_UNESCAPED_UNICODE`, meaningful HTTP codes (400, 403, 409…), readable message in the body.
- Private fields (phone, e-mail, notes) **stripped server-side** for anonymous visitors — never just hidden with CSS.

### "Real time" without WebSockets

The front-end reloads `action=all` every 15–20 s and after each write. Simple, robust, works on any host, enough for a few people editing at once.

## Authentication

- **One shared password** for organisers/admins, stored as `password_hash()` in `config.php` (never in clear). Empty hash = nobody is admin.
  Generate: `php -r 'echo password_hash("secret", PASSWORD_DEFAULT), PHP_EOL;'`
- Session with a named cookie, `HttpOnly`, `SameSite=Lax`, `Secure` when on HTTPS; `session_regenerate_id(true)` on login; `sleep(1)` on a wrong password.

```php
session_name('myapp_sid');
session_set_cookie_params([
    'httponly' => true,
    'samesite' => 'Lax',
    'secure'   => !empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off',
]);
session_start();
```

## CSRF

Token in the session (`bin2hex(random_bytes(32))`), injected in the page (`window.BOOT.csrf`), sent in the `X-CSRF-Token` header of every POST, checked with `hash_equals`. Applies to public POSTs too.

## Public forms

- Honeypot field (e.g. `website`, hidden); if filled, pretend success and do nothing.
- Per-session rate limit (e.g. 20 submissions/hour).
- When matching an existing person by name, ignore case; submitted contact data only fills empty fields, never overwrites.

## Protecting the data folder

`data/.htaccess`:

```apache
<IfModule mod_authz_core.c>
  Require all denied
</IfModule>
<IfModule !mod_authz_core.c>
  Order allow,deny
  Deny from all
</IfModule>
```

plus an empty `data/index.php`. After deploying, check that `https://…/data/app.sqlite` returns 403. On nginx hosts, add an equivalent `location` rule or move `data/` outside the web root and adjust the DSN.

## XSS

- Client side, every string from the DB is escaped before entering the DOM (`textContent`, or an `esc()` helper in template strings) — never raw `innerHTML`.
- Rich text (editor): sanitise **server-side** with a tag whitelist (`DOMDocument`, see `lib/html.php`: p, br, b/strong, i/em, u, a, ul/ol/li, h3/h4, blockquote) and again when displaying (`DOMParser`). Links limited to `https:`, `http:`, `mailto:`, `tel:`. On paste, keep plain text only.
