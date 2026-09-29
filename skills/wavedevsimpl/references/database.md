# Database: SQLite via PDO

Reference code: [youlvat `lib/db.php`](https://github.com/breizhwave/youlvat/blob/main/lib/db.php), [`database/schema.sql`](https://github.com/breizhwave/youlvat/blob/main/database/schema.sql).

## Connection

```php
$pdo = new PDO($cfg['dsn'], $cfg['db_user'], $cfg['db_pass'], [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
]);
$pdo->exec('PRAGMA foreign_keys = ON; PRAGMA journal_mode = WAL; PRAGMA busy_timeout = 5000;');
```

- `foreign_keys`: SQLite does not enforce them by default; required for `ON DELETE CASCADE`.
- `journal_mode = WAL`: reads during writes, better under load.
- `busy_timeout`: wait instead of failing immediately when the DB is locked.

`config.php` holds `'dsn' => 'sqlite:' . __DIR__ . '/data/app.sqlite'`, so switching to MySQL later is a config change plus a data import.

Before connecting, check that `pdo_sqlite` is loaded and that `data/` exists and is writable; if not, throw a dedicated exception whose message tells the user exactly what to do (`chmod 775 data/`). Pages catch it and show the message instead of a blank page or stack trace.

## Schema creation and evolution

- DB file missing or empty → run `database/schema.sql` (plus an optional seed file). No install step.
- Evolution → an idempotent `migrate()` called on every connection:

```php
function migrate(PDO $pdo): void
{
    $cols = array_column($pdo->query('PRAGMA table_info(events)')->fetchAll(), 'name');
    foreach (['image_url', 'contact_html'] as $c) {
        if (!in_array($c, $cols, true)) $pdo->exec("ALTER TABLE events ADD COLUMN $c TEXT");
    }
}
```

Existing databases upgrade themselves on the next page load. Also add new columns to `schema.sql` so fresh installs get them directly.

## Conventions

- Text IDs generated server-side (`bin2hex(random_bytes(8))`); dates as ISO 8601 text (`gmdate('Y-m-d\TH:i:s\Z')`).
- Standard SQL (avoid needless SQLite-isms) to keep MySQL possible.
- `ON DELETE CASCADE` on relations: deleting a parent cleans children, no application code for it.
- **Concurrency-sensitive constraints in SQL**, e.g. never exceed the number of places:

```sql
INSERT INTO assignments (id, shift_id, person_id, status)
SELECT ?, ?, ?, 'planned'
WHERE (SELECT COUNT(*) FROM assignments WHERE shift_id = ? AND status <> 'absent')
      < (SELECT needed FROM shifts WHERE id = ?);
```

Check `rowCount()`: 0 means full → HTTP 409 with a readable message.

- Multi-step writes in a transaction, `rollBack()` in the `catch`.

## Backup

An admin-only `export.php` producing:
- a full JSON backup (every table) — also the way to change host or DBMS;
- a CSV readable in a spreadsheet (UTF-8 with BOM, `;` separator for European Excel if relevant).

Remind the user to export before any important date, or to copy `data/` while the site is quiet.
