---
name: wavedevsimpl
description: Build small, durable web apps with PHP 8 + SQLite (PDO) + vanilla HTML/CSS/JS — no framework, no Composer, no npm, no build step — that deploy by copying one folder to any cheap shared PHP host. Use when the user wants a simple, portable, low-maintenance web tool (association, club, festival, small business, internal tool; tens to hundreds of users), asks for "the simple stack", "wavedevsimpl", something that "runs anywhere" or "on shared hosting / FTP", or wants to avoid Firebase/Supabase/framework lock-in. Pages are rendered server-side in PHP first, with JavaScript only as a small add-on. Covers project layout, SQLite setup and self-migrating schema, a single JSON API with whitelists and CSRF, shared-password admin, vanilla front-end rendering, multilingual UI, local testing and deployment. Not for multi-server, high write-rate, fine-grained user accounts or real-time collaborative apps.
---

# wavedevsimpl — the simple portable stack

**PHP 8 + SQLite (PDO) + vanilla HTML/CSS/JS. No framework, no Composer, no build.**
One folder you upload by FTP, and it works. Moving host = copying the folder.

Reference implementation (read it when you need real code): **[breizhwave/youlvat](https://github.com/breizhwave/youlvat)** — a trilingual (Breton / French / English) volunteer scheduler for festivals built exactly this way. Its `tech.md` is the original French version of this skill.

## Why this stack

| Constraint | Consequence |
|---|---|
| Cheapest possible hosting (any shared PHP host) | PHP is everywhere; nothing to install server-side. |
| Change host painlessly | SQLite: the database is a file. Migrate = copy the folder. |
| Maintained by non-specialists for years | No dependencies to update, no `npm install`, no build that breaks in 3 years. |
| No external service dependency | No Firebase/Supabase (quotas, free-tier pausing, changing terms). |
| Modest traffic with occasional peaks | SQLite in WAL mode easily handles a few writes per second. |

Rejected: **JSON files** (concurrent writes lose data), **MySQL by default** (DB creation + credentials + export/import at each move; keep the door open via a DSN in `config.php` and standard SQL), **Laravel/Symfony/React/Vue** (overkill for a few screens, forces Composer/npm and a build).

### Do NOT use this stack when

- several web servers sit behind a load balancer (SQLite = one server);
- writes are very frequent and concurrent (more than a few dozen per second);
- you need real per-user accounts with fine roles, password recovery, etc.;
- the UI is very rich and strictly real-time (collaborative editor…).

If one of these applies, say so to the user and propose something else.

## Project layout

```
app/
  index.php          main page (HTML rendered by PHP, a little JS only if needed)
  other-page.php     secondary public pages
  api.php            single JSON API (?action=…), only for the JS parts
  export.php         CSV export / full JSON backup (admin)
  config.php         app name, admin password hash, DSN   (gitignored; ship config.example.php)
  lib/db.php         PDO connection, pragmas, schema creation, migrations
  lib/auth.php       session, login, CSRF token
  lib/tables.php     table/column whitelist, camelCase <-> snake_case
  lib/html.php       rich-HTML sanitiser (only if there is an editor)
  assets/app.css     shared styles
  assets/i18n.js     translations (if multilingual)
  database/schema.sql, seed_*.sql
  data/              the SQLite file (+ .htaccess "Require all denied", empty index.php)
```

Rules:
- **Relative paths everywhere**: the app must work in a subfolder (`https://example.org/myapp/`).
- Each PHP page is self-contained: PHP at the top (session, read data, handle the form POST), then HTML with `<?= h($value) ?>`, and a `<script>` only if the page needs one. No template engine.
- `.gitignore`: `data/*.sqlite`, `data/*.sqlite-*`, `config.php`.

## PHP first, JavaScript second

**Write the page in PHP by default; add JavaScript only for what PHP cannot do.** The main reason: it is easier to edit for the people who maintain the tool.

- One language to read: open the `.php` file and you see the HTML and the data that fills it.
- No state duplicated between server and browser, no client-side re-rendering, no API to evolve in step with each screen.
- Business rules (calculations, checks, permissions) are written **once, in PHP**; JS never re-implements them.
- "View source" shows the real output; PHP errors land in the server log, not in a phone's console.
- Pages work without JS, and links/bookmarks work naturally.

So: render HTML in PHP (escape with `h()` = `htmlspecialchars($s, ENT_QUOTES, 'UTF-8')`); plain `<form method="post">` handled by PHP then **POST → redirect → GET**; navigation by real links with GET parameters (`?event=…&tab=…`). Use JS only for instant filtering/toggling, two-click confirmation, clipboard, a rich-text editor, or periodically refreshing one area — always as progressive enhancement. Details and the i18n consequence: [references/frontend-i18n.md](references/frontend-i18n.md).

(The reference app youlvat predates this rule: its pages are still rendered in JS because they were ported from a prototype. Take its PHP libraries, security and SQL as the model, not its JS rendering.)

## Core rules (always apply)

1. **Every PDO connection**: `ERRMODE_EXCEPTION`, `FETCH_ASSOC`, then `PRAGMA foreign_keys = ON; PRAGMA journal_mode = WAL; PRAGMA busy_timeout = 5000;`
2. **Zero-install schema**: missing DB → create it from `database/schema.sql`. Schema changes → idempotent `migrate()` run on each connection (`PRAGMA table_info` + `ALTER TABLE … ADD COLUMN`). Unwritable `data/` → clear error message, never a blank page.
3. **Concurrency-sensitive business rules live in SQL** (conditional `INSERT … SELECT … WHERE`), not in PHP.
4. **One API endpoint** (for the JS parts only), prepared statements only, **whitelist** of tables and columns, camelCase ↔ snake_case in one place, private fields **removed server-side** for anonymous visitors.
5. **Shared admin password** as `password_hash()` in `config.php`; empty hash = nobody is admin. CSRF token on every POST (hidden form field, or `X-CSRF-Token` header for `fetch`).
6. **Front-end**: **PHP renders the HTML**, vanilla JS only as an add-on; escape everything from the DB (`h()` in PHP, `textContent` / `esc()` in JS), mobile first, CSS variables, light theme only (`color-scheme: only light`).
7. **Test on a copy**: never delete or reset the real database, never overwrite `config.php`.

## Details — read the matching reference when working on that part

- Database connection, schema, migrations, conventions, backup → [references/database.md](references/database.md)
- JSON API, auth, CSRF, public forms, XSS, rich text → [references/api-security.md](references/api-security.md)
- Front-end structure and multilingual UI → [references/frontend-i18n.md](references/frontend-i18n.md)
- Local dev, testing, screenshots, deployment → [references/dev-deploy.md](references/dev-deploy.md)

Real code for each piece in youlvat: [`lib/db.php`](https://github.com/breizhwave/youlvat/blob/main/lib/db.php), [`lib/auth.php`](https://github.com/breizhwave/youlvat/blob/main/lib/auth.php), [`lib/tables.php`](https://github.com/breizhwave/youlvat/blob/main/lib/tables.php), [`lib/html.php`](https://github.com/breizhwave/youlvat/blob/main/lib/html.php), [`api.php`](https://github.com/breizhwave/youlvat/blob/main/api.php), [`export.php`](https://github.com/breizhwave/youlvat/blob/main/export.php), [`assets/i18n.js`](https://github.com/breizhwave/youlvat/blob/main/assets/i18n.js), [`data/.htaccess`](https://github.com/breizhwave/youlvat/blob/main/data/.htaccess). Its identifiers and comments are in French; adapt names to the new project's language.

## New-project checklist

- [ ] Scaffold the layout above; start `lib/db.php`, `lib/auth.php`, `lib/tables.php`, `api.php` from youlvat (or write them from the references).
- [ ] Write `database/schema.sql` with foreign keys and `ON DELETE CASCADE`.
- [ ] Declare tables and columns in the whitelist.
- [ ] Decide what is public, what is admin-only, and which fields are private.
- [ ] Put concurrency constraints in SQL.
- [ ] Public forms: honeypot field + rate limit.
- [ ] Public pages tested at 375 px wide.
- [ ] JSON/CSV export working before go-live.
- [ ] Record project decisions in `CLAUDE.md` and the stack in `tech.md` (link back to this skill).
