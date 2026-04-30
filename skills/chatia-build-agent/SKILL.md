---
name: chatia-build-agent
description: Use this skill when a developer needs to create a new AI agent on Chatia from external code (Python, Node, curl, n8n, Zapier, custom backend). Covers the canonical end-to-end flow — get an API key, create the agent, upload knowledge, attach tools, publish, embed, and listen to webhooks. Use whenever the user mentions creating, deploying, or programmatically managing a Chatia agent.
version: 0.1
---

# Chatia · Build an agent from your code

Step-by-step recipe for any external developer to spin up a Chatia
agent end-to-end. Every step has a concrete `curl` you can paste, plus
Python and Node equivalents.

## Prerequisites

1. A Chatia owner account at https://chatia.pro/register (free, 100
   trial messages, no card needed).
2. A developer API key — generate it at
   `https://chatia.pro/dashboard/developers` → "API keys" → "Create".
   The plaintext is shown **once**; store it as `CHATIA_API_KEY` in
   your env. Format: `chatia_<base64>` (~64 chars).

Set:

```bash
export CHATIA_API_KEY="chatia_..."
export CHATIA_BASE="https://api.chatia.pro"
```

## Step 1 — Create the agent

`POST /api/developers/agents` with the developer key. Returns the
agent id, slug and (if `publish: true`) the public webchat URL.

**Request body** (canonical fields, all optional except `name`):

```json
{
  "name": "Asistente comercial",
  "description": "Capta leads y deriva a un humano cuando hace falta.",
  "system_prompt": "Sos el asistente comercial de Acme. Captas leads y respondés FAQ. Nunca inventes precios.",
  "first_message": "¡Hola! ¿En qué te puedo ayudar?",
  "language": "es",
  "provider": "chatia",
  "model": "chatia-lite",
  "tools": ["capture_lead", "query_knowledge", "send_whatsapp_handoff"],
  "publish": true
}
```

**curl**:

```bash
curl -X POST "$CHATIA_BASE/api/developers/agents" \
  -H "Authorization: Bearer $CHATIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Asistente comercial",
    "description": "Capta leads y deriva a humano",
    "tools": ["capture_lead", "query_knowledge"],
    "publish": true
  }'
```

**Python**:

```python
import os
import httpx

resp = httpx.post(
    f"{os.environ['CHATIA_BASE']}/api/developers/agents",
    headers={"Authorization": f"Bearer {os.environ['CHATIA_API_KEY']}"},
    json={
        "name": "Asistente comercial",
        "description": "Capta leads y deriva a humano",
        "tools": ["capture_lead", "query_knowledge"],
        "publish": True,
    },
    timeout=20,
)
resp.raise_for_status()
agent = resp.json()
print(agent["id"], agent["public_slug"], agent["chat_url"])
```

**Response**:

```json
{
  "id": 42,
  "name": "Asistente comercial",
  "status": "draft",
  "is_published": true,
  "public_slug": "asistente-comercial-42",
  "chat_url": "/chat/asistente-comercial-42",
  "tools": ["capture_lead", "query_knowledge"]
}
```

The `public_slug` is auto-generated as `slug(name) + "-" + id` so
duplicates between owners can't collide.

### `provider` / `model` choices

| provider | model | what it costs you |
|---|---|---|
| `chatia` | `chatia-lite` | $0.015/msg managed (default — no key needed) |
| `openai` | `gpt-4o-mini`, `gpt-5.4-mini`, etc. | $0.004/msg BYOK if you load your OpenAI key in `/dashboard/api-settings`, otherwise $0.015 trial-managed |
| `anthropic` | `claude-3-5-sonnet-...`, etc. | $0.004/msg BYOK only — load your Anthropic key first |

Most external devs want `provider: "chatia"` — zero setup, our chain
(OpenAI primary → Groq → Cerebras) is OpenAI-compatible and tool-
calling stable.

## Step 2 — Built-in tools catalog

Pass any subset in the `tools` array. Reference (verified against
`builder_tools.py`):

**Sales & lead capture**
- `capture_lead` — store name, phone, email, interest.
- `update_lead_status` — move a lead through CRM kanban (new →
  contacted → qualified → won/lost).
- `add_lead_note` — timestamped note on a lead.
- `send_whatsapp_handoff` — hand the visitor off to a human
  WhatsApp number.

**Calendar**
- `list_availability` — return appointments between two ISO 8601
  dates so the model can propose free slots.
- `create_calendar_event` — book an appointment (anti-overlap; rejects
  collisions automatically). Auto-sends a confirmation email if
  `contact_email` is provided AND the agent has email configured.
- `cancel_appointment` — cancel by id with a reason.
- `reschedule_appointment` — move an existing appointment.
- `schedule_appointment` — legacy alias for `create_calendar_event`.

**Knowledge & comms**
- `query_knowledge` — semantic search over the agent's KB chunks.
- `send_email` — send a plain-text email to the contact (requires
  Resend/SMTP config in the agent).
- `whatsapp_send_template` — send a pre-approved WA template via
  Kapso.
- `send_whatsapp_text` — send free text inside the 24h window.

**Recruitment / HR**
- `register_candidate` — register a new candidate for a job position.
- `score_candidate` — score 0-10 per competency.
- `flag_for_review` — flag for human review.

**Support / ops**
- `send_nps_survey` — send a 0-10 survey link at end of conversation.
- `http_request` — call any user-configured external HTTP API
  (escape-hatch for custom integrations).
- `intro` — soft greeting helper used by the conversational builder.

If you pass a tool name that isn't in this list, the backend silently
drops it — no 4xx. So spelling matters.

## Step 3 — Upload knowledge

Three ingestion modes. Pick whichever fits your data source.

### a) Plain text chunks

```bash
curl -X POST "$CHATIA_BASE/api/v1/agents/42/knowledge" \
  -H "Authorization: Bearer $CHATIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Pricing FAQ",
    "content": "El plan Pro cuesta $12/mes con 5.000 mensajes...",
    "tags": ["pricing", "faq"]
  }'
```

### b) Upload a file (PDF / TXT / MD / CSV)

We auto-extract text + auto-chunk. Multipart form upload:

```bash
curl -X POST "$CHATIA_BASE/api/v1/agents/42/knowledge/upload-file" \
  -H "Authorization: Bearer $CHATIA_API_KEY" \
  -F "file=@./pricing.pdf" \
  -F "title=Pricing one-pager" \
  -F "tags=pricing,faq"
```

### c) Crawl a URL

We fetch the page, extract readable text and chunk it:

```bash
curl -X POST "$CHATIA_BASE/api/v1/agents/42/knowledge/import-url" \
  -H "Authorization: Bearer $CHATIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://acme.com/pricing", "tags": ["pricing"]}'
```

The agent will use `query_knowledge` automatically when you attach
that tool — no extra wiring.

## Step 4 — Customize branding (optional)

The webchat at `/{public_slug}` inherits global owner branding by
default. Override per agent:

```bash
curl -X PATCH "$CHATIA_BASE/api/v1/agents/42/branding" \
  -H "Authorization: Bearer $CHATIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "primary_color": "#10a37f",
    "logo_url": "https://acme.com/logo.png",
    "theme": "dark",
    "powered_by_text": "by Acme",
    "show_powered_by": true
  }'
```

Hide the "Powered by Chatia" badge requires a paid plan; the API
silently keeps it shown for Free/PayG accounts (`allow_hide_badge`
gate).

## Step 5 — Embed the agent

You have **three ways** to put the agent in front of users:

### a) Direct webchat link

`https://chatia.pro/chat/{public_slug}` — full-page conversational
UI, mobile-friendly, no embed code.

### b) Orb widget

Drop one script tag on any page. The orb opens a chat panel:

```html
<script src="https://chatia.pro/widget.js" data-slug="asistente-comercial-42" async></script>
```

### c) Custom integration via the public chat API

For server-side or in-app chat (your own UI calling our brain):

```bash
curl -X POST "$CHATIA_BASE/api/public/agents/asistente-comercial-42/chat/stream" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hola, quiero info de precios",
    "history": [],
    "visitor_id": "anon-123"
  }'
```

Returns SSE. Each event is a JSON line with `type` ∈ `text_delta`,
`tool_called`, `tool_output`, `done`, `error`.

## Step 6 — Listen to webhooks

Wire a HTTPS endpoint at `https://chatia.pro/dashboard/developers`
or via API:

```bash
curl -X POST "$CHATIA_BASE/api/developers/webhooks" \
  -H "Authorization: Bearer $CHATIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Lead capture sink",
    "url": "https://your-app.com/hooks/chatia",
    "events": ["agent.lead.captured", "agent.message.created"]
  }'
```

The response includes the `secret` (shown once) — store it; you'll
need it to verify signatures.

For signature verification, retry behaviour, replay protection and
event payload shapes, **load the `chatia-webhooks` skill** — it's
the deep-dive companion.

## Common pitfalls

- **Tool typos are silent** — the backend drops unknown tool names
  without erroring. After `POST /agents`, check `response.tools` matches
  what you sent.
- **`first_message` doesn't appear in WhatsApp** — only in the webchat
  greeting. WhatsApp requires the user to message first.
- **Free trial bills BUILDER + chat from the same bucket** — if you
  call `/api/v1/chat/completions` 100 times during dev, your trial is
  burned. Load BYOK before scaling.
- **`provider: "anthropic"` without BYOK** returns the canned fallback
  reply ("Entiendo, contame más…") — there's no platform Anthropic
  key.
- **`http_request` tool requires per-agent config** — the URL +
  headers live in `agent.config.http_actions`. Set them via the owner
  dashboard or via `PATCH /api/agents/{id}` with the right config
  blob; just adding the tool name doesn't make it functional.
- **Slugs are immutable post-publish**. Pick a good `name` first.

## Pricing reminder

- Free: 100 lifetime trial messages + $3 welcome credit.
- After trial: $0.015/msg managed or $0.004/msg BYOK (73% cheaper —
  always cargá tu OpenAI key in `/dashboard/api-settings` if you can).
- Subscriptions (Starter $5, Pro $12, Studio $29, Agency $59) include
  message quotas; overage at the rates above.
- Builder messages, public agent messages, AND `/api/v1/chat/completions`
  calls all draw from the **same bucket**.

## Quick test recipe

A 30-second end-to-end smoke test from scratch:

```bash
# 1. create
AGENT=$(curl -s -X POST "$CHATIA_BASE/api/developers/agents" \
  -H "Authorization: Bearer $CHATIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","tools":["capture_lead"],"publish":true}')
SLUG=$(echo "$AGENT" | jq -r .public_slug)

# 2. chat (SSE; first event chunks the streaming reply)
curl -N "$CHATIA_BASE/api/public/agents/$SLUG/chat/stream" \
  -H "Content-Type: application/json" \
  -d '{"message":"hola","history":[]}'

# 3. open the webchat in browser
echo "https://chatia.pro/chat/$SLUG"
```

If step 2 streams text deltas, your agent is alive. ✓

## When in doubt

- Read the live catalog at `https://chatia.pro/SKILL.md` before
  inventing endpoints — it's the authoritative source.
- `GET /api/developers/skills` returns the live tool catalog with
  schemas; never hardcode a tool list, fetch this.
- `GET /api/developers/events/catalog` returns the webhook event
  catalog (24 types as of v0.1).
