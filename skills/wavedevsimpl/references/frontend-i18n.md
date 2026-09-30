# Front-end and multilingual UI

Reference code: youlvat [`index.php`](https://github.com/breizhwave/youlvat/blob/main/index.php), [`inscription.php`](https://github.com/breizhwave/youlvat/blob/main/inscription.php) (mobile-first public form), [`assets/app.css`](https://github.com/breizhwave/youlvat/blob/main/assets/app.css), [`assets/i18n.js`](https://github.com/breizhwave/youlvat/blob/main/assets/i18n.js).

## Front-end

- **Vanilla HTML/CSS/JS**, one `<script>` per page. No bundler, no transpiling: the code that runs is the code you read.
- State in one object `S`; one render function per view (`renderDashboard()`, `renderPlan()`…) that rebuilds the HTML of its area. Fast enough for a few hundred items.
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
