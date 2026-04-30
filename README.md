# Chatia Skills

Skills for Claude Code (and any compatible coding agent that supports
the [Skills](https://www.npmjs.com/package/skills) catalog) to build
on top of [Chatia](https://chatia.pro) — a SaaS that builds AI agents
for freelancers, with a public REST API and signed webhooks.

## What's in this repo

Three composable skills, install whichever ones fit your workflow:

| Skill | Use when… |
|---|---|
| **`chatia`** | The general overview — Chatia design language, prompt structure, REST endpoints, security rules, stack reference. Start here if you only install one. |
| **`chatia-build-agent`** | You're writing code that creates / configures / publishes Chatia agents from outside Chatia. Step-by-step recipes with curl + Python + Node. |
| **`chatia-webhooks`** | You're consuming Chatia webhooks. HMAC signature verification (constant-time), replay protection, idempotency, retry behaviour, full event catalog. |

`chatia-build-agent` and `chatia-webhooks` are companions of `chatia`
— they go deeper into the two domains where most external developers
need exact code.

## Install

### From GitHub (recommended)

```bash
# General overview
npx skills add alemart87/chatia-skill --skill chatia -a claude-code -g -y

# How to build agents from your code
npx skills add alemart87/chatia-skill --skill chatia-build-agent -a claude-code -g -y

# How to consume webhooks safely
npx skills add alemart87/chatia-skill --skill chatia-webhooks -a claude-code -g -y
```

You can install all three at once if you're going to integrate
end-to-end. They don't conflict — different triggers in their
descriptions, no overlap.

### Other coding agents

Replace `-a claude-code` with `-a codex` or `-a cursor`:

```bash
npx skills add alemart87/chatia-skill --skill chatia-build-agent -a cursor -g -y
```

### Local development (this repo)

From the cloned repo root:

```bash
npx skills add . --skill chatia -a claude-code -g -y
```

That points your agent at the local `skills/<name>/SKILL.md` so you
can iterate without committing.

## Verify

After install, in any Claude Code session:

```
Use chatia-build-agent to create a sales agent with the calendar tools
and webhooks pointing at https://my-app.com/hooks/chatia.
```

If the skill loaded correctly, the agent will:

- Show the exact `POST /api/developers/agents` body with `tools`
  including `create_calendar_event`, `list_availability`,
  `capture_lead`.
- Walk through `POST /api/developers/webhooks` with the right `events`
  array.
- Reference signature verification from `chatia-webhooks` (if also
  installed).

## What these skills do NOT do

- They don't run code or hit APIs on their own — they're read-only
  context.
- They don't include authentication credentials. You still need a JWT
  or `CHATIA_API_KEY` to call the API.
- They don't bundle a CLI; install the skill in your coding agent and
  let it generate code.

## Repo layout

```
chatia-skill/
├── README.md                            ← this file
└── skills/
    ├── chatia/
    │   └── SKILL.md                     ← general overview
    ├── chatia-build-agent/
    │   └── SKILL.md                     ← build agents from your code
    └── chatia-webhooks/
        └── SKILL.md                     ← consume webhooks safely
```

That structure is fixed — `npx skills add` looks for
`skills/<name>/SKILL.md` from the repo root.

## Updating

1. Edit the SKILL.md files in this repo.
2. Bump `version` in the frontmatter (semver-ish — bump on breaking
   changes to anchors people might link).
3. Commit + push to `main`.
4. Re-run `npx skills add alemart87/chatia-skill --skill <name> ...`
   on each machine — `npx skills` does not auto-update.

For Chatia API additions (new endpoints, new events), the live source
of truth is `https://chatia.pro/SKILL.md`. Keep this repo aligned with
that file when shipping new product surface.

## License

MIT. Use freely; attribution welcome but not required.
