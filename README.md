# wavedevsimpl

An open-source [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) for building small, durable web apps with **PHP 8 + SQLite + vanilla HTML/CSS/JS** — no framework, no Composer, no npm, no build step. One folder you upload by FTP to any cheap shared host, and it works. Moving host = copying the folder.

Made for association tools, clubs, festivals, small businesses and internal tools: tens to hundreds of users, maintained by volunteers for years.

The repository also ships **waveskridcms**, a second skill built on the same stack: it wraps an existing static HTML site in a one-shot WYSIWYG CMS (see below).

Reference implementation: **[breizhwave/youlvat](https://github.com/breizhwave/youlvat)** — a trilingual (Breton / French / English) volunteer scheduler for festivals.

## Install

**Claude Code (plugin marketplace)**

```
/plugin marketplace add breizhwave/wavedevsimpl
/plugin install wavedevsimpl@wavedevsimpl
```

**Claude Code (manual)**

```sh
git clone https://github.com/breizhwave/wavedevsimpl
cp -r wavedevsimpl/skills/wavedevsimpl ~/.claude/skills/
cp -r wavedevsimpl/skills/waveskridcms ~/.claude/skills/   # optional: the static-site CMS skill
```

**claude.ai / Claude Desktop**: zip the `skills/wavedevsimpl` folder (and/or `skills/waveskridcms`) and upload it under *Settings → Capabilities → Skills*.

Other agents that read the `SKILL.md` format can use the same folder.

## Use

Ask for it naturally — "build me a small sign-up app for our club that runs on our OVH hosting", "use the simple stack" — or name it: "use wavedevsimpl".

## waveskridcms — a CMS for a static HTML site

Give it a static site (folder, zip or URL — or let it write one first) and it builds a small editor around it:

- the site's own HTML/CSS/JS stay the design; edits are stored as overrides and republished as clean static HTML;
- in-place editing of existing texts, images (library), links, show/hide blocks, repeated items, shared header/footer,
  colour tokens, per-page SEO with a live score;
- no templates and no free positioning — content stays in the site's own flow;
- on publish: canonical/Open Graph tags, linked JSON-LD (organisation, pages, breadcrumb, FAQ), sitemap, robots.txt with
  explicit AI-crawler rules, `llms.txt` and `llms-full.txt`;
- options chosen at the start: `new-pages` (compose new pages from blocks of existing pages, off by default),
  `repeat-items`, `colors`.

Ask for it: "make a waveskridcms for this site", "let my client edit this HTML site".

## What's inside

```
skills/wavedevsimpl/
  SKILL.md                    stack, why, when not to use it, layout, core rules, checklist
  references/database.md      SQLite/PDO, self-migrating schema, SQL-level concurrency, backup
  references/api-security.md  single JSON API, whitelists, shared-password admin, CSRF, XSS
  references/frontend-i18n.md vanilla front-end patterns, multilingual UI without a library
  references/dev-deploy.md    local dev, safe testing, screenshots, FTP deployment
skills/waveskridcms/
  SKILL.md                    static site → one-shot CMS: ingest, overrides, editor, options, SEO + GEO publish
```

## License

MIT
