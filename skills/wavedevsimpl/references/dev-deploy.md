# Local development, testing, deployment

## Run locally

```sh
php -S 127.0.0.1:8000        # from the app folder
```

No build, no install. The database is created on first load.

## Test safely

- **Always test on a copy**: `rsync -a --exclude 'data/*.sqlite*' app/ /tmp/app-test/` (or copy the real DB aside and restore it). Never delete or reset the real database — it may contain the user's own data. Never overwrite `config.php` (it holds the user's password hash).
- Disable opcache in CLI if you change `config.php` during tests: `php -d opcache.enable_cli=0 -S …`.
- API tests with `curl`, keeping a cookie jar for the session and reading the CSRF token from the page:

```sh
curl -s -c jar -b jar http://127.0.0.1:8000/index.php | grep -o '"csrf":"[^"]*"'
curl -s -c jar -b jar -H 'Content-Type: application/json' -H "X-CSRF-Token: $TOKEN" \
     -d '{"action":"login","password":"secret"}' http://127.0.0.1:8000/api.php
```

- Screenshots with headless Chrome to check desktop and mobile layout:

```sh
chrome --headless=new --hide-scrollbars --screenshot=desk.png --window-size=1280,900 http://127.0.0.1:8000/
chrome --headless=new --screenshot=phone.png --window-size=390,844 http://127.0.0.1:8000/
```

  (Headless Chrome may clamp very narrow windows to a minimum width; if a phone shot looks cropped, compare with a known-good page or use device emulation before concluding there is horizontal overflow.)

- Test every language (`?lang=…`) at least once.

## Deploy

1. Upload the folder by FTP/SFTP (without `data/*.sqlite*` from your machine, unless you mean to ship data).
2. Make sure `data/` is writable by PHP.
3. Put the password hash in `config.php` (copied from `config.example.php`).
4. Open the page: the database creates itself.
5. Check that `https://…/data/<db>.sqlite` returns 403.

Back up regularly with `export.php` (or by copying `data/` while the site is quiet).

## Moving host

Copy the folder (including `data/`), check `data/` permissions, done. To move to MySQL: change the DSN, create the schema, import the JSON backup.
