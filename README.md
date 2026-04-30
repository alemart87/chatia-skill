# Chatia Skill

Skill for Claude Code (and any compatible coding agent that supports
the [Skills](https://www.npmjs.com/package/skills) catalog) to build
on top of [Chatia](https://chatia.pro) — a SaaS that builds AI agents
for freelancers, with a public REST API and HMAC-signed webhooks.

## What's inside

A single `chatia` skill that covers everything an external developer
needs:

1. **Overview & rules** — Chatia design language, prompt structure,
   security non-negotiables, stack reference, full endpoint catalog.
2. **Building agents** — step-by-step recipes (curl + Python + Node)
   to create / configure / publish agents from your code.
3. **Consuming webhooks** — HMAC signature verification (constant-
   time), replay protection, idempotency, retry behaviour, full
   event catalog (24 types).

## Install

### One command — recommended

```bash
npx skills add alemart87/chatia-skill --skill chatia -a claude-code -g -y
```

That's it. The skill loads with all three sections — your coding
agent will know how to build agents AND consume webhooks AND
follow Chatia's design language.

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

After install, in any Claude Code session:

```
Use chatia to create a sales agent with the calendar tools and
webhooks pointing at https://my-app.com/hooks/chatia.
```

If the skill loaded correctly, the agent will:

- Show the exact `POST /api/developers/agents` body with `tools`
  including `create_calendar_event`, `list_availability`,
  `capture_lead`.
- Walk through `POST /api/developers/webhooks` with the right
  `events` array.
- Provide signature verification code (Python or Node, depending
  on your stack).

## What this skill does NOT do

- It doesn't run code or hit APIs on its own — it's read-only
  context.
- It doesn't include authentication credentials. You still need a
  JWT or `CHATIA_API_KEY` to call the API.
- It doesn't bundle a CLI; install the skill in your coding agent
  and let it generate code.

## Repo layout

```
chatia-skill/
├── README.md              ← this file
└── skills/
    └── chatia/
        └── SKILL.md       ← the all-in-one skill (~1000 lines)
```

That structure is fixed — `npx skills add` looks for
`skills/<name>/SKILL.md` from the repo root.

## Updating

1. Edit `skills/chatia/SKILL.md`.
2. Bump `version` in the frontmatter.
3. Commit + push to `main`.
4. Re-run the install command on each machine — `npx skills` does
   not auto-update.

For Chatia API additions (new endpoints, new events), the live
source of truth is `https://chatia.pro/SKILL.md`. Keep this repo
aligned with that file when shipping new product surface.

## License

MIT. Use freely; attribution welcome but not required.
