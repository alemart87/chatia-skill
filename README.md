# Chatia Skill

Skill for Claude Code (and any compatible coding agent that supports the
[Skills](https://www.npmjs.com/package/skills) catalog) to work on
[Chatia](https://chatia.pro) — a SaaS that builds AI agents for
freelancers.

## What you get

When this skill is loaded, the agent gets context about:

- Chatia's design language (premium / minimal / monochromatic).
- The canonical structure for writing agent prompts.
- API endpoints, auth model and pricing tiers.
- Tool-calling discipline (date injection, honesty rule, anti-hallucination).
- UI rules for cards, pills, motion, color usage.
- Security non-negotiables (key hygiene, signature verification,
  multi-tenant isolation, prompt-injection defense).
- The actual stack (FastAPI 0.115 + Next.js 16 + Postgres + Polar + Kapso
  + Resend) so suggestions are grounded.

## Install

### From GitHub (recommended for collaborators)

```bash
npx skills add alemart87/chatia-skill --skill chatia -a claude-code -g -y
```

> Replace `alemart87` with your GitHub user if you fork the repo.

### Local install (during development)

From the cloned repo root:

```bash
npx skills add . --skill chatia -a claude-code -g -y
```

That points Claude Code at the `skills/chatia/SKILL.md` in your working
copy so you can iterate without committing.

### Install on Codex / Cursor / other agents

```bash
# Codex
npx skills add alemart87/chatia-skill --skill chatia -a codex -g -y

# Cursor
npx skills add alemart87/chatia-skill --skill chatia -a cursor -g -y
```

## Verify

After install, in any Claude Code session:

```
Use the Chatia skill to improve this sales agent prompt.
```

Or:

```
Use the Chatia skill to make these agent cards look enterprise.
```

If the skill loaded correctly, the agent will reference Chatia's design
rules (monochromatic palette, mono labels, hairline borders) and the
canonical prompt structure (identity / goal / capabilities / tools /
tone / rules / escalation).

## What this skill does NOT do

- It does not run code or hit APIs on its own — it is read-only context.
- It does not include Chatia's full endpoint catalog (40+ routes); for
  that, point your agent at `https://chatia.pro/SKILL.md`.
- It does not bundle authentication credentials. You still need a JWT
  or `CHATIA_API_KEY` to call the API.

## Repo layout

```
chatia-skill/
├── README.md                  ← this file
└── skills/
    └── chatia/
        └── SKILL.md           ← the actual skill content
```

That structure is fixed — `npx skills add` looks for
`skills/<name>/SKILL.md` from the repo root.

## Updating the skill

1. Edit `skills/chatia/SKILL.md`.
2. Bump anything API-related against the live `https://chatia.pro/SKILL.md`.
3. Commit + push to `main`.
4. Re-run `npx skills add alemart87/chatia-skill --skill chatia -a claude-code -g -y`
   on each machine — `npx skills` does not auto-update by default.

## License

MIT. Use freely, attribution welcome but not required.
