# Chatia Skill

Skill for Claude Code, Codex, Cursor, and any compatible coding agent
(catalog: [skills](https://www.npmjs.com/package/skills)) to build on
top of [Chatia](https://www.chatia.pro) — a SaaS that builds AI agents
for freelancers and SMBs, with a public REST API and HMAC-signed
webhooks.

## What's inside

A single `chatia` skill that covers everything an external developer
needs to deploy production AI agents end-to-end:

1. **Overview & rules** — Chatia design language, prompt structure,
   security non-negotiables, stack reference, full endpoint catalog
   with rate limits.
2. **Onboarding from zero** — registration, API key creation, env
   setup. So an AI agent can guide a new user from "no account" to
   "agent live in production" without manual hand-off.
3. **Building agents** — step-by-step recipes (curl + Python + Node)
   to create / configure / publish agents in **one API call** that
   returns absolute `chat_url`, `dashboard_url`, `widget_snippet`,
   and `wordpress_plugin_url` ready to paste.
4. **Consuming webhooks** — HMAC signature verification (constant-
   time), replay protection, idempotency, retry behaviour, full
   event catalog (**27 types** including `agent.sale.*` for the
   sales pipeline).
5. **Embed targets** — HTML, WordPress (official plugin GPL-2.0,
   downloadable as `.zip`), Shopify (via `theme.liquid`), Wix (via
   Custom Code), Webflow.
6. **Vertical -> tools mapping** — recommended tool-set defaults for
   sales / support / appointments / recruitment / mixed verticals.
7. **Common errors and retry patterns** — HTTP status table with
   action per code + exponential backoff recipe.

## Install

### One command — recommended

```bash
npx skills add alemart87/chatia-skill --skill chatia -a claude-code -g -y
```

That's it. The skill loads with all sections — your coding agent
will know how to onboard a user, build agents, embed them, consume
webhooks, and follow Chatia's design language.

### Other coding agents

```bash
# Codex
npx skills add alemart87/chatia-skill --skill chatia -a codex -g -y

# Cursor
npx skills add alemart87/chatia-skill --skill chatia -a cursor -g -y
```

### Local development (this repo)

From the cloned repo root:

```bash
npx skills add . --skill chatia -a claude-code -g -y
```

That points your agent at the local `skills/chatia/SKILL.md` so you
can iterate without committing.

## Verify

After install, in any Claude Code / Cursor / Codex session:

```
Use chatia to create a sales agent for my dental clinic with
calendar tools and webhooks pointing at https://my-app.com/hooks/chatia.
```

If the skill loaded correctly, the agent will:

- Show the exact `POST /api/developers/agents` body with `tools`
  including `create_calendar_event`, `list_availability`,
  `capture_lead`.
- Surface the response with `chat_url`, `dashboard_url`,
  `widget_snippet`, `wordpress_plugin_url` ready to copy-paste back
  to you.
- Walk through `POST /api/developers/webhooks` with the right
  `events` array.
- Provide signature verification code (Python or Node, depending
  on your stack).
- Surface rate limits (30/min on agent creation, 120/min on events)
  and a backoff recipe for 429 / 5xx.

## Alternative — paste-ready operational instructions

If you don't want to install a skill and prefer to paste markdown
into your AI agent's context:

```
https://www.chatia.pro/AGENTS.md
```

That's a ~210-line concentrated version designed to be a system
prompt context for AI agents. The full SKILL.md (this repo) is
~1166 lines with deeper detail (data models, multi-language recipes,
full webhook signature verification examples).

## What this skill does NOT do

- It doesn't run code or hit APIs on its own — it's read-only
  context.
- It doesn't include authentication credentials. You still need a
  `CHATIA_API_KEY` (create one at
  `https://www.chatia.pro/dashboard/developers`).
- It doesn't bundle a CLI; install the skill in your coding agent
  and let it generate the code.

## Repo layout

```
chatia-skill/
README.md              <- this file
skills/
  chatia/
    SKILL.md           <- the all-in-one skill (~1166 lines)
```

That structure is fixed — `npx skills add` looks for
`skills/<name>/SKILL.md` from the repo root.

## Live machine-readable surfaces

Chatia publishes the same content at these public URLs (no auth
required) so AI assistants can fetch them on-demand:

| URL | Purpose |
|---|---|
| [chatia.pro/SKILL.md](https://www.chatia.pro/SKILL.md) | Full spec (mirrors this repo's `skills/chatia/SKILL.md`) |
| [chatia.pro/AGENTS.md](https://www.chatia.pro/AGENTS.md) | Paste-ready operational instructions (~210 LOC) |
| [chatia.pro/llms.txt](https://www.chatia.pro/llms.txt) | Standard LLM ingestion file (https://llmstxt.org) |
| [chatia.pro/llms-full.txt](https://www.chatia.pro/llms-full.txt) | Extended factual context for LLM retrieval |
| [chatia.pro/api/developers/events/catalog](https://www.chatia.pro/api/developers/events/catalog) | Live JSON of the 27 webhook events |
| [chatia.pro/api/developers/skills](https://www.chatia.pro/api/developers/skills) | Live JSON of the 18+ runtime tools |

## Updating

1. Edit `skills/chatia/SKILL.md`.
2. Bump `version` in the frontmatter.
3. Commit + push to `main`.
4. Re-run the install command on each machine — `npx skills` does
   not auto-update.

For Chatia API additions (new endpoints, new events), the live
source of truth is `https://www.chatia.pro/SKILL.md`. Keep this repo
aligned with that file when shipping new product surface.

## Versions

| Version | Date | Changes |
|---|---|---|
| 0.4 | 2026-05-09 | +Custom HTTP webhook tools section (the owner defines their own tools that hit their API). +Composio integrations OAuth flow end-to-end (25+ toolkits, 5 meta-tools model, billing x5 markup). +Meta direct webhooks for Facebook/Instagram/Messenger inbound (1 platform Meta App, auto-subscribe Pages, anti-loop filters). +Telegram direct webhooks setup. ~280 LOC added, total ~1166 LOC. |
| 0.3 | 2026-05-09 | +27 events (sale.* added), enriched response shape (widget_snippet, wordpress_plugin_url, dashboard_url absolute), rate limits per API key documented (30/min agents, 120/min events), AGENTS.md cross-link, onboarding from zero, multi-language recipes (Python/Node), error table with retry pattern, vertical to tools mapping. |
| 0.2 | 2026-04 | Initial public release. |

## License

MIT. Use freely; attribution welcome but not required.
