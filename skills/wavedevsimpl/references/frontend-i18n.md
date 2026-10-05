# Front-end and multilingual UI

Reference code: youlvat [`index.php`](https://github.com/breizhwave/youlvat/blob/main/index.php), [`inscription.php`](https://github.com/breizhwave/youlvat/blob/main/inscription.php) (mobile-first public form), [`assets/app.css`](https://github.com/breizhwave/youlvat/blob/main/assets/app.css), [`assets/i18n.js`](https://github.com/breizhwave/youlvat/blob/main/assets/i18n.js).

## PHP renders the page

The default page shape — HTML produced by PHP, a form handled by the same file:

```php
<?php
require __DIR__ . '/lib/auth.php';
function h($s) { return htmlspecialchars((string)$s, ENT_QUOTES, 'UTF-8'); }

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (!csrf_check($_POST['csrf'] ?? null)) { http_response_code(403); exit('Session expired: reload the page.'); }
    // validate, then write with a prepared statement…
    header('Location: ?event=' . urlencode($_POST['event']) . '&ok=1');   // POST -> redirect -> GET
    exit;
}
$rows = db()->query('SELECT id, name FROM items ORDER BY name')->fetchAll();
?>
<ul>
<?php foreach ($rows as $r): ?>
  <li><?= h($r['name']) ?></li>
<?php endforeach ?>
</ul>
<form method="post">
  <input type="hidden" name="csrf" value="<?= h(csrf_token()) ?>">
  …
</form>
```

- Tabs, days, filters that change the page = links with GET parameters, read by PHP.
- Errors: re-display the form with the message and the user's input, or redirect with `?error=…`.
- Business calculations (totals, fill state, overlaps…) live in PHP functions in `lib/`; if JS needs a value, PHP puts it in the HTML (`data-*` attribute) instead of JS recomputing it.

## JavaScript as an add-on

- **Vanilla JS**, at most one `<script>` per page. No bundler, no transpiling: the code that runs is the code you read.
- Only for: instant filter/toggle without reload, two-click delete confirmation, copy to clipboard, rich-text editor, periodic refresh of one area. The page must still work without it.
- When one area really must be rebuilt client-side (live board refreshed every 15 s): state in one object `S`, one render function per area, data from `api.php`. Keep that the exception.
- Boot data injected by PHP to save a request:

```php
<script>window.BOOT = <?= json_encode(['csrf' => csrf_token(), 'admin' => is_admin()], JSON_UNESCAPED_UNICODE) ?>;</script>
```

- A small `api(body)` helper: `fetch('api.php', {method:'POST', headers:{'Content-Type':'application/json','X-CSRF-Token':BOOT.csrf}, body:JSON.stringify(body)})`, shows the server's message in a toast on error, then reloads state.
- **Mobile first** for public pages; CSS Grid/Flex, simple breakpoints (e.g. 980 px and 560 px), no horizontal scroll.
- Colours as CSS variables on `:root`. Light theme only: no automatic dark theme, and `color-scheme: only light` on `:root` so mobile browsers (Chrome Android auto-dark, Samsung Internet) don't darken the page themselves.
- Google Fonts only; nothing else loaded from outside.
- View preferences (view, language) in `localStorage`, always inside `try/catch`.
- Context in the URL (`?event=<id>`, `?lang=en`): shareable links, working back button.
- Deletions: two clicks on the same button (an `armed()` state) instead of `confirm()`.

## Multilingual

- With PHP rendering, translations must be readable by PHP: e.g. `lang/br.php` returning an array, and a PHP `t()` with the same rules as below. Pass to JS only the few strings JS itself displays (`window.BOOT.i18n`). The JS-only variant below (`assets/i18n.js`) suits apps whose UI is mainly rendered in JS, like youlvat today.
- `assets/i18n.js`, no library. **The default-language text is the key**: `t('Save')`, `tn(n, '{n} shift', '{n} shifts')`, variables `t('{n} places', {n})`. One dictionary object per extra language, all using the same keys; the default language needs no dictionary.

```js
const DICT = {br: BR, en: EN}[lang] || {};
function fill(s, vars) { return vars ? s.replace(/\{(\w+)\}/g, (m, k) => vars[k] != null ? vars[k] : m) : s; }
function t(s, vars) { return fill(DICT[s] != null ? DICT[s] : s, vars); }
function tn(n, one, many, vars) { return t(n > 1 ? many : one, Object.assign({n}, vars || {})); }
```

- Static HTML text: `data-i18n="…"` (and `data-i18n-title`), translated on load.
- Language: `?lang=` then `localStorage`, default language otherwise; a small switcher in the header. Set `document.documentElement.lang`.
- Dates through `Intl` (`toLocaleDateString`); write month/day names by hand when the language is not covered by browsers (Breton, for instance).
- Server error messages are written in the default language and translated client-side with `t()` — add them to every dictionary.
- User-entered data (names, instructions) is not translated. Editable rich texts get one column per language, falling back to the default language when empty.
- Ship a small check script (grep `t('…')` / `tn(…)` / `data-i18n` and compare with each dictionary's keys) that reports missing and unused keys.
- Every new string goes into every dictionary; have translations reviewed by a native speaker.
