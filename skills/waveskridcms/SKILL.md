---
name: waveskridcms
description: Wrap an existing (or freshly written) static HTML website in a one-shot WYSIWYG CMS built on the wavedevsimpl stack (PHP 8 + SQLite + vanilla JS) — in-place editing of the site's own texts, images, links and blocks, no templates, no free layout, SEO + GEO (llms.txt, JSON-LD, AI-crawler robots) on publish, optional new pages composed from existing blocks. Use when the user wants "a CMS for my static site", "let my client edit this HTML site", "waveskridcms", or an editable version of a site delivered as HTML/zip/URL.
---

# waveskridcms — a one-shot « Skrid » CMS for a static HTML site

Goal: in one pass, wrap a static website (existing folder / zip / URL, or one you write first) in a small, safe
WYSIWYG CMS. The site's own HTML, CSS and JS stay the design: the CMS lets people **change what is already there**
(texts, images, links, show/hide blocks, SEO) and republishes clean static HTML with SEO + GEO extras.

It is the light sibling of Skrid CMS (a fuller WYSIWYG site builder with multi-site, kits and free layout). It builds on
the **wavedevsimpl** skill (same repository, `skills/wavedevsimpl`): read it for the stack rules. Two things are left
out on purpose:

1. **No templates** — no kits, no section/page templates, no « new site » dialog, no vectors menu. By default only the
   pages that exist are edited (see option `new-pages` to compose new pages from existing blocks).
2. **No absolute positioning, no free layout** — elements stay in the site's own flow. You edit the content of existing
   elements/blocks; you never drag, resize, rotate or insert free-floating elements or new element types.

## Options

Ask once at the start (one AskUserQuestion, multiSelect, with the defaults pre-explained). If nobody answers, use the
defaults and say so. Record the choice as constants in `config.php` (and `config.example.php`) and in `CLAUDE.md`, and
build only the code paths of the enabled options — a disabled option leaves no UI, no API action, no dead code.

| Option | Default | Constant | What it adds |
|---|---|---|---|
| `new-pages` | off | `OPT_NEW_PAGES` | Create new pages by picking blocks from existing pages (§ Option new-pages). Adds complexity: only enable when asked. |
| `repeat-items` | on | `OPT_REPEAT_ITEMS` | Duplicate / remove items of repeated lists (cards, roster, nav links, FAQ entries). |
| `colors` | on if the CSS has `:root` colour tokens | `OPT_COLORS` | Edit the site's colour tokens in « Site ». |
| `ai-crawlers` | allow | setting, not constant | robots.txt Allow vs Disallow for AI crawlers (changeable later in « Site »). |

## Stack and house rules (wavedevsimpl — github.com/breizhwave/wavedevsimpl)

- PHP 8 + SQLite (PDO: WAL, foreign_keys, busy_timeout) + vanilla HTML/CSS/JS. No Composer, npm or build step.
- Zero-install: `database/schema.sql` (IF NOT EXISTS) run on first request + idempotent `migrate()` (add columns, never drop).
- One JSON endpoint `api.php`: action whitelist, column whitelist (`lib/tables.php`), camelCase ⇄ snake_case.
- Admin: one shared password stored as `password_hash` in `config.php` (gitignored; ship `config.example.php`),
  session cookie httponly + SameSite=Lax, `session_regenerate_id` on login, CSRF token in `X-CSRF-Token` on every POST.
- PHP renders pages; JS only for the editor. All SQL with prepared statements. Escape everything with `h()`.
- Use `INSERT OR REPLACE` (never `ON CONFLICT … DO UPDATE`: older SQLite builds bundled with PHP on Macs reject it).
- Write `README.md` (user-facing, in the site's language), `CLAUDE.md` (decisions, enabled options) and `tech.md`
  (stack). Editor UI language = site language.
- Editor look: a dark → blue gradient glass UI for the editor only (record it in CLAUDE.md as an exception to
  wavedevsimpl's light-theme rule); the published site keeps its own design.

## Step 0 — Get the site

- **Existing site**: a folder, a .zip, or a URL. Copy it into `source/` with its structure intact (relative paths,
  assets, CSS, JS). Zip: extract into its own empty directory first, drop `__MACOSX`, `.DS_Store`. URL: download the
  pages you can; if the site is rendered by JavaScript, ask for « Enregistrer sous… › Page web complète » or the files.
- **New site**: first write a clean static site (semantic HTML5, one CSS file, minimal JS, real headings, `<header>`,
  `<nav>`, `<main>`, `<footer>`), then treat it as an existing site.
- Ask only what you cannot infer: destination folder, admin password, public base URL, language, options.

## Architecture: source pages are the templates, edits are overrides

```
site-cms/
  index.php          live site (renders source + overrides; drafts visible to admin with ?preview=1)
  editor.php         login + editor shell (page list, iframe canvas, inspector)
  render.php         render one page: ?page=slug[&edit=1] (edit=1 → admin only, injects assets/inline.js)
  api.php            JSON API: save_text, save_img, save_link, toggle_block, save_page, save_site, upload, publish
                     [+ dup_item, del_item if repeat-items] [+ page_create, page_delete, blocks_set if new-pages]
  export.php         ?format=zip | json | page&id=N (single self-contained file)
  config.php         DB_NAME, PUBLIC_DIR, ADMIN_PASSWORD_HASH, OPT_* (gitignored)
  lib/ db.php auth.php tables.php html.php ingest.php render.php seo.php publish.php [compose.php]
  assets/ studio.css studio.js (editor shell) inline.css inline.js (in-page editing)
  database/schema.sql
  source/            the original static site (pages + css/js/fonts/images), ids added by ingest
  images/            uploaded images (library)
  data/              sqlite + zip (.htaccess deny + index.php)
  public/            published output
```

Schema (minimum):

```sql
CREATE TABLE IF NOT EXISTS pages (id INTEGER PRIMARY KEY, slug TEXT UNIQUE, file TEXT, title TEXT, meta_title TEXT,
  meta_desc TEXT, og_image TEXT, in_sitemap INTEGER DEFAULT 1, hidden INTEGER DEFAULT 0, sort INTEGER, updated_at TEXT,
  kind TEXT DEFAULT 'source',   -- 'source' | 'composed' (new-pages option)
  shell TEXT DEFAULT '',        -- composed: slug of the page lending head/header/footer
  blocks TEXT DEFAULT '');      -- composed: JSON [{id, from, key}]
CREATE TABLE IF NOT EXISTS edits (scope TEXT, key TEXT, kind TEXT, value TEXT, updated_at TEXT,
  PRIMARY KEY(scope, key));     -- scope = page slug, or 'site' for shared blocks
CREATE TABLE IF NOT EXISTS meta (key TEXT PRIMARY KEY, value TEXT);   -- site settings
```

## Step 1 — Ingest (`lib/ingest.php`, run once at install, re-runnable)

For every `source/**/*.html` (skip 404 pages and partials):

1. Register the page: slug from the path (`index.html` → home `''`, `a/b.html` → `b`, dedupe), `<title>`,
   meta description, og:image; order = the home page's first `<nav>` link order, then the rest.
2. Mark editable elements with a stable `data-sk` attribute **written back into the source file** (deterministic:
   page slug + DOM path hash, never reused). Kinds:
   - `text`: h1–h6, p, li, blockquote, figcaption, dt/dd, td/th, label, button, and `a`/`span` whose content is only
     inline (b, i, em, strong, br, small, span, a).
   - `img`: every `<img>` (src + alt), `<picture>` via its `<img>` (drop stale `<source>` on change).
   - `link`: every `<a href>` (href; its text is a `text`).
   - `block`: section, article, aside, direct children of `<main>`, header/footer children, and each child of a
     grid/list repeating the same structure (`data-sk-item` + `data-sk-list` on the parent).
   Skip script, style, svg internals, iframes, forms (read-only).
3. Shared blocks: `<header>`, `<nav>`, `<footer>` (or any block) with identical normalized HTML across pages get the
   same key with scope `site` — editing once updates every page.
4. Design tokens: `:root { --name: colour }` in the site's CSS → « Couleurs » (if `colors`), injected as
   `<style>:root{…}</style>` after the site's stylesheets. Logo: first `<img>`/`<svg>` in the header's home link.
5. new-pages only: mark each page's content region `data-sk-main` (the `<main>`, else the element holding most
   top-level blocks) and give every top-level block a human label (its first heading, else its id/class, else
   « Bloc n ») in a `blocks_index` (page, key, label, text excerpt) for the block picker.
6. Change nothing else in the source. Re-running ingest keeps existing ids and only adds ids for new elements.

## Step 2 — Render with overrides (`lib/render.php`)

`render_page(slug, mode)`: load the page file with DOMDocument (`<?xml encoding="UTF-8">` prefix, LIBXML_NONET, silence
errors) — for a composed page, call `compose()` first (§ Option new-pages) — then apply `edits` for scope `site`, then
the page:
- text → replace children with sanitized inline HTML; img → src/alt (src must be `images/…` from the library or an
  original asset inside `source/`); link → href (validated); hidden block → removed (publish) or `data-sk-hidden`
  (editor, dimmed); repeated items → ordered list of item keys per `data-sk-list`.
- Asset URLs must work from `render.php` (base = the page's folder in `source/`) and from `public/` (keep relative paths).
- Inject head extras (Step 5); in edit mode add `assets/inline.css` + `assets/inline.js` + a `<base>`.
With **no edits, published source pages must equal the source except head extras and stripped `data-sk*`
attributes** — test it with a diff.

## Step 3 — The editor (no absolute positioning)

`editor.php`: left = page list (title, slug, draft badge, SEO score; source pages cannot be added, deleted or moved);
center = iframe `render.php?page=…&edit=1` with desktop / tablet / phone width toggles; right = inspector tabs
« Élément », « Page & SEO », « Site ».

`inline.js` (inside the iframe; postMessage to the parent, same-origin fetch to api.php with CSRF):
- Hover outlines on `[data-sk]`; click selects (Esc → parent block); block breadcrumb in the inspector.
- Text: double-click → `contenteditable` on that element only, paste as plain text, toolbar B / I / link / unlink /
  clear; blur or Ctrl+S saves. Enter in a heading saves instead of creating a block.
- Image: click → library dialog (upload JPG/PNG/WebP/GIF/SVG ≤ 8 MB, server-checked: getimagesize; SVG without
  script, on*, javascript:, foreignObject) → replace; alt text; update width/height attributes from the new file.
- Link: href field with a datalist of pages (`page:slug#anchor` internally), https://, mailto:, tel:, #anchor.
- Block: show / hide. repeat-items: duplicate (fresh keys for the copy's descendants) / remove an item.
- Undo/redo per page, save chip (« Enregistré » / « Modifications… »), autosave debounce.
- Never: drag, resize, free positioning, new element types, section or page templates.

« Page & SEO »: menu title, meta title + description with counters and a Google preview, og image (library), draft
toggle, live SEO + GEO score (`lib/seo.php`: one H1, heading order, alt texts, length, links, a quotable intro
paragraph, question-style subheadings).

« Site » (global config): site name, tagline, base URL, language, logo (replaces the detected logo), colour tokens
(if `colors`), header/footer = shared blocks edited in place, facts for search & AI engines: organisation type, AI
summary, e-mail, phone, address, founder, official profiles (sameAs), FAQ (« Question ? | Réponse » per line), AI crawlers.

## Option new-pages — compose a page from existing blocks

Only when `OPT_NEW_PAGES` is true. Still no templates and no free layout: a new page is a **stack of copies of blocks
that already exist**, inside the frame of an existing page.

- « ＋ Nouvelle page » (page list): title → slug (unique, sanitized), **cadre** = an existing page whose head, header,
  footer and wrappers are reused (default: the most similar inner page, not the home), then the block picker.
- Block picker: blocks grouped by source page, each with label, text excerpt and a live mini preview (scaled iframe of
  `render.php?page=…&block=key`); search box; click to append. On the new page: ↑ / ↓ to reorder, ✖ to remove,
  « Dupliquer ». Order changes only within the composed page's main region — still no positioning.
- Storage: `pages.kind='composed'`, `shell` = frame slug, `blocks` = `[{id:'b1', from:'artistes', key:'sk-…'}]`.
- `compose()` (`lib/compose.php`): load the shell file, empty its `[data-sk-main]`, then for each entry import a deep
  clone of the source block **with the source page's edits applied at that moment** (initial content), prefix every
  descendant `data-sk` with the instance id (`b1~sk-…`) so later edits belong to the new page only (scope = its slug),
  and suffix `id` attributes / anchors that would collide. Shared (`site`) blocks keep their key.
- Assets: the block's relative URLs are rebased from its source page's folder to the shell's folder. The published
  file goes next to the shell file (`dirname(shell)/slug.html` or `slug/index.html`, same scheme as the site) so every
  relative link stays valid.
- Composed pages are editable exactly like the others, deletable (with confirmation; their edits are removed), and
  included everywhere (sitemap, llms.txt, JSON-LD). They are not added to the menu automatically: add a link by
  duplicating a nav item (repeat-items) and pointing it to `page:slug` — tell the user.
- Guards: refuse a block containing `data-sk-main`, refuse an empty page, cap at 40 blocks, keep one H1 (demote the
  others to H2 in the clone and say so in the SEO score).
- Extra tests: compose a page from blocks of two different pages in different folders, edit a text in it, check the
  source pages did not change, publish, and check its images, links and CSS load at 1440 px and 390 px.

## Step 4 — Publish and export

`publish.php`: each non-draft page rendered in static mode → `public/<same path as source>` (keep the site's URL
scheme), copy the site's assets and only the library images in use, `404.html` if the source has one, then
sitemap/robots/llms (Step 5), then `data/site.zip`. Report pages, files, KB, warnings.
Exports: zip of the published site; one page as a single self-contained file (CSS/JS/images inlined as data URIs);
JSON backup (pages + edits + settings [+ composed pages]) and JSON restore.

## Step 5 — SEO + GEO extras added at publish

- Head: title, meta description, canonical (needs base URL), robots (`max-snippet:-1, max-image-preview:large`),
  Open Graph + Twitter, `og:locale`, `meta generator`, `article:modified_time`, `<link rel=alternate type=text/plain
  href=llms.txt>`. Replace existing tags instead of duplicating them.
- JSON-LD `@graph` linked by `@id`: Organization (typed: School, NGO, MusicGroup, LocalBusiness…) with description,
  email, telephone, PostalAddress, founder, sameAs, logo; WebSite; WebPage (dateModified, primary image, isPartOf);
  BreadcrumbList; FAQPage (site FAQ on home + blocks whose heading says FAQ and whose subheadings end with « ? »).
  Merge with JSON-LD already in the source instead of duplicating it.
- `sitemap.xml` (lastmod, priority); `robots.txt` (Allow all; explicit Allow or Disallow for GPTBot, OAI-SearchBot,
  ChatGPT-User, ClaudeBot, Claude-User, Claude-SearchBot, PerplexityBot, Google-Extended, Applebot-Extended, CCBot,
  Meta-ExternalAgent, Bytespider; Sitemap line); `llms.txt` (name, summary, facts, pages with descriptions, FAQ,
  profiles) and `llms-full.txt` (all page text as Markdown in reading order).
- Never fabricate facts: use only what is in the source or what the user typed in settings.

## Security checklist

- Admin-only: editor, api, export, `render.php?edit=1`. `data/` not web-readable (.htaccess + index.php guard).
  Login slowed on failure.
- Sanitize every stored value server-side: inline HTML whitelist (b, strong, i, em, u, s, br, a[href], span), hrefs by
  whitelist (https?, mailto, tel, #anchor, page:slug#anchor; no spaces, quotes, backslashes, javascript:), image paths
  `images/…` or existing `source/` assets only, no `..`.
- Only keys that exist (source `data-sk`, or composed instance keys) can be edited; unknown keys are rejected.
  new-pages: `from`/`key` must exist in `blocks_index`, slugs sanitized and unique.

## Verification before delivering

1. `php -l` on every PHP file, `node --check` on every JS file.
2. `php -S 127.0.0.1:PORT` + Playwright (or any headless browser): log in, edit a heading, a paragraph
   with a link, replace an image, hide a block, [repeat-items: duplicate an item], edit the shared footer once and check
   two pages, [colors: change a token], [new-pages: the extra tests above], publish.
3. Diff test: a fresh install published with no edits matches the source pages (except head extras).
4. Screenshots of the editor and of published pages at 1440 px and 390 px; look at them before claiming success.
5. Check `robots.txt`, `sitemap.xml`, `llms.txt`, JSON-LD (json_decode), no JS console errors.

## Delivery

Write into the user's chosen folder (check what exists first; never overwrite their `config.php`, `data/`, `images/` or
`public/`). Tell the user briefly: how to start (`php -S 127.0.0.1:8000` → `/editor.php`, initial password and how to
change it with `password_hash`), the enabled options, what can be edited, what is intentionally not possible
(templates, free layout, and new pages unless `new-pages` is on), and where the published site lands.
