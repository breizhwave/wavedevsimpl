# wavedevsimpl

An open-source [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) for building small, durable web apps with **PHP 8 + SQLite + vanilla HTML/CSS/JS** — no framework, no Composer, no npm, no build step. One folder you upload by FTP to any cheap shared host, and it works. Moving host = copying the folder.

Made for association tools, clubs, festivals, small businesses and internal tools: tens to hundreds of users, maintained by volunteers for years.

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
```

**claude.ai / Claude Desktop**: zip the `skills/wavedevsimpl` folder and upload it under *Settings → Capabilities → Skills*.

Other agents that read the `SKILL.md` format can use the same folder.

## Use

Ask for it naturally — "build me a small sign-up app for our club that runs on our OVH hosting", "use the simple stack" — or name it: "use wavedevsimpl".

## What's inside

```
skills/wavedevsimpl/
  SKILL.md                    stack, why, when not to use it, layout, core rules, checklist
  references/database.md      SQLite/PDO, self-migrating schema, SQL-level concurrency, backup
  references/api-security.md  single JSON API, whitelists, shared-password admin, CSRF, XSS
  references/frontend-i18n.md vanilla front-end patterns, multilingual UI without a library
  references/dev-deploy.md    local dev, safe testing, screenshots, FTP deployment
```

## License

MIT
