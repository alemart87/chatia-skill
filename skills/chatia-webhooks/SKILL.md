---
name: chatia-webhooks
description: Use this skill when implementing or debugging Chatia webhook handlers — verifying HMAC signatures, replay protection, idempotency, retry behaviour, and the full event payload catalog. Use whenever the user mentions Chatia webhooks, webhook signatures, "x-chatia-*" headers, or handling Chatia events (agent.lead.captured, agent.reply.created, etc.).
version: 0.1
---

# Chatia · Webhook handlers (deep-dive)

How to consume Chatia webhooks safely and idempotently from any
server (Node, Python, Edge, Bun). Companion to `chatia-build-agent`
which covers creating webhook endpoints — this one covers handling
the deliveries.

## Delivery format

Every webhook is a `POST` with `Content-Type: application/json`. The
body is a single JSON object:

```json
{
  "id": "evt_<32-hex>",
  "type": "agent.lead.captured",
  "created_at": "2026-04-30T12:00:00+00:00",
  "agent_id": 42,
  "conversation_id": "conv_abc123",
  "data": {
    "lead_id": 17,
    "name": "Ana",
    "email": "ana@acme.com",
    "phone": "+5491111",
    "interest": "demo"
  }
}
```

Headers we send:

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `User-Agent` | `Chatia-Webhooks/1.0` |
| `x-chatia-event` | event type, e.g. `agent.lead.captured` |
| `x-chatia-event-id` | `evt_<hex>` (idempotency key, unique per event) |
| `x-chatia-timestamp` | Unix seconds when we signed |
| `x-chatia-signature` | `sha256=<hex>` — HMAC-SHA256 over `{ts}.{body}` |

## Signature verification

The signed string is **`{timestamp}.{raw_body}`**, NOT the body alone.
Use `hmac.compare_digest` (or constant-time equivalent) — `==` is
vulnerable to timing attacks.

### Python (FastAPI)

```python
import hmac
import hashlib
import time
from fastapi import APIRouter, Request, HTTPException

router = APIRouter()
WEBHOOK_SECRET = "<the secret returned when you created the endpoint>"

@router.post("/hooks/chatia")
async def chatia_webhook(request: Request):
    raw = await request.body()
    ts = request.headers.get("x-chatia-timestamp", "")
    sig = request.headers.get("x-chatia-signature", "")
    if not ts.isdigit() or not sig.startswith("sha256="):
        raise HTTPException(400, "bad headers")
    if abs(time.time() - int(ts)) > 300:
        raise HTTPException(400, "stale (>5 min)")
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        f"{ts}.{raw.decode()}".encode(),
        hashlib.sha256,
    ).hexdigest()
    if not hmac.compare_digest(expected, sig.removeprefix("sha256=")):
        raise HTTPException(401, "bad signature")
    payload = await request.json()
    # ... handle by event type
    return {"ok": True}
```

### Node (Express)

```js
import express from "express";
import crypto from "node:crypto";

const app = express();
const SECRET = process.env.CHATIA_WEBHOOK_SECRET;

app.post(
  "/hooks/chatia",
  // IMPORTANT: capture raw body BEFORE JSON parsing — the signature
  // is over the exact bytes we sent. Stringifying parsed JSON does
  // NOT round-trip and will fail verification.
  express.raw({ type: "application/json" }),
  (req, res) => {
    const ts = req.get("x-chatia-timestamp");
    const sig = req.get("x-chatia-signature");
    if (!ts || !sig?.startsWith("sha256=")) return res.sendStatus(400);
    if (Math.abs(Date.now() / 1000 - Number(ts)) > 300)
      return res.sendStatus(400);
    const raw = req.body.toString("utf8");
    const expected = crypto
      .createHmac("sha256", SECRET)
      .update(`${ts}.${raw}`)
      .digest("hex");
    const got = sig.slice(7);
    if (
      expected.length !== got.length ||
      !crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(got))
    )
      return res.sendStatus(401);
    const payload = JSON.parse(raw);
    // ... handle
    res.json({ ok: true });
  },
);
```

### Edge / Cloudflare Workers / Vercel Edge

```js
export default {
  async fetch(request, env) {
    const raw = await request.text();
    const ts = request.headers.get("x-chatia-timestamp");
    const sig = request.headers.get("x-chatia-signature");
    if (!ts || !sig?.startsWith("sha256=")) return new Response("bad", { status: 400 });
    if (Math.abs(Date.now() / 1000 - Number(ts)) > 300)
      return new Response("stale", { status: 400 });
    const key = await crypto.subtle.importKey(
      "raw",
      new TextEncoder().encode(env.CHATIA_WEBHOOK_SECRET),
      { name: "HMAC", hash: "SHA-256" },
      false,
      ["sign"],
    );
    const mac = await crypto.subtle.sign(
      "HMAC",
      key,
      new TextEncoder().encode(`${ts}.${raw}`),
    );
    const expected = [...new Uint8Array(mac)]
      .map((b) => b.toString(16).padStart(2, "0"))
      .join("");
    if (expected !== sig.slice(7)) return new Response("bad sig", { status: 401 });
    const payload = JSON.parse(raw);
    // ... handle
    return Response.json({ ok: true });
  },
};
```

## Replay protection

We send `x-chatia-timestamp` so you can reject stale deliveries.
Standard window is **5 minutes** (300 s); anything older → 400.
Combined with the signature, a leaked old payload can't be replayed
indefinitely.

## Idempotency

The same `evt_<hex>` is **never re-emitted** to the same endpoint
under normal conditions. But during a network blip we may retry the
delivery (see Retry below) — your handler must be **idempotent on
`x-chatia-event-id`**.

Pattern: use the event id as a unique key in your DB before doing the
work.

```sql
CREATE TABLE chatia_events_seen (
  event_id TEXT PRIMARY KEY,
  event_type TEXT NOT NULL,
  received_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```python
async def handle(payload, db):
    try:
        await db.execute(
            "INSERT INTO chatia_events_seen (event_id, event_type) VALUES (?, ?)",
            (payload["id"], payload["type"]),
        )
    except UniqueViolation:
        return  # already processed
    # do the actual work HERE — only runs once per event_id
```

## Retry behaviour

- **Delivery timeout**: 8 seconds. If your handler hasn't returned a
  2xx in 8s, we mark it failed and move on. Acknowledge fast — queue
  heavy work to a background job.
- **Retries on failure**: not yet implemented in v0.1. Failed deliveries
  show in the dashboard but are not auto-retried. To resync, refetch
  via `GET /api/developers/agents/{id}/conversations`.
- **Roadmap**: exponential backoff retries with `attempts` counter
  (the `WebhookDelivery.attempts` field in our DB is reserved for this).

Until retries land: **respond 2xx fast** and don't rely on us trying
again. If you need durability, push the event to your own queue
synchronously, ack 200, and process async.

## Event catalog (v0.1)

The authoritative live catalog: `GET /api/developers/events/catalog`.
Fetch it instead of hardcoding — we may add events without bumping
the API version.

### Lifecycle (`group: lifecycle`)

| Event | When |
|---|---|
| `agent.created` | Agent created via dashboard or API. |
| `agent.published` | Agent's webchat went live. |
| `agent.branding.updated` | Logo / color / powered_by changed. |

### Conversation (`group: conversation`)

| Event | `data` shape |
|---|---|
| `agent.message.created` | `{ text, channel, visitor_id }` — user wrote. |
| `agent.reply.created` | `{ text, latency_ms, key_source }` — agent answered. |
| `agent.lead.captured` | `{ lead_id, name, email, phone, interest, source }`. |
| `agent.tool.called` | `{ tool, arguments, ok }` — agent ran a tool. |
| `agent.handoff.requested` | `{ reason }` — escalate to human. |

### WhatsApp delivery (`group: whatsapp`)

| Event | When |
|---|---|
| `agent.message.delivered` | Meta marked as delivered. |
| `agent.message.read` | User read the message. |
| `agent.message.failed` | Delivery failed (network, blocked, etc.). |
| `agent.outbound.skipped_24h_window` | We didn't send because >24h since user last spoke. |
| `agent.phone_number.quality_changed` | Meta reduced/raised number quality. |
| `agent.phone_number.banned` | Meta banned/limited the number. |

### Clients & team (`group: clients` / `team`)

For owners managing sub-users (agencies):

- `client.created`, `client.updated`, `client.removed`
- `client.password_reset`
- `client.branding.updated`
- `client.agent_grant.added`, `client.agent_grant.removed`
- `client.team_member.added`, `client.team_member.removed`

### Portal (`group: portal`)

- `portal.branding.updated` — global owner branding changed.

## Testing your endpoint

Trigger a test delivery from the dashboard:

```bash
curl -X POST "$CHATIA_BASE/api/developers/webhooks/{webhook_id}/test" \
  -H "Authorization: Bearer $CHATIA_API_KEY"
```

This sends an `agent.message.created` event with placeholder data so
you can verify the signature flow without doing real chat.

## Common pitfalls

- **Don't re-stringify the parsed body** before signing — the signed
  string is over the **raw bytes** we sent. JSON re-stringification
  will reorder keys, normalize spaces, drop nulls — all of which
  break the HMAC. In Express, use `express.raw({ type: "application/json" })`
  and parse INSIDE the handler. In FastAPI, use `await request.body()`
  not `await request.json()` for the verification step.
- **Don't compare with `==`** — timing attack. Use
  `hmac.compare_digest` / `crypto.timingSafeEqual`.
- **Don't trust `Content-Length` for signature inputs**. Read the
  raw body once and reuse the bytes.
- **Don't chain heavy work in the handler**. We give you 8s; spend
  ≤200ms. Push to a queue and ack.
- **Don't filter events server-side via the `events` array if you'll
  need to add new types later** — pass `[]` (subscribe to all) and
  filter in your handler. We added 6 events between v0.1 patches; if
  you hardcoded the list at create time, you'd miss them silently.
- **Don't expose the secret in logs**. `WebhookEndpoint.secret` is
  `Chatia_<hex>` — treat it like a password.

## Rotating a secret

The secret is generated at endpoint creation and **shown once**. To
rotate:

1. Create a new endpoint with the same `events`.
2. Cut over your handler to the new secret.
3. `DELETE /api/developers/webhooks/{old_id}` once you've verified
   the new one is receiving traffic.

There's no in-place secret rotation endpoint in v0.1 — we'd rather
force a clean cutover than juggle two valid secrets at once.

## When in doubt

- The single source of truth for events is
  `GET /api/developers/events/catalog`. Fetch it on app boot if you
  display them in your UI.
- The `WebhookDelivery` table is queryable per endpoint via the
  dashboard — you'll see status_code, response_body, attempts. Useful
  for debugging "why isn't my handler getting called".
- If signature verification fails repeatedly, log the **raw body**
  (carefully, contains PII) plus the headers, and re-compute the HMAC
  manually against the secret you have stored — 99% of the time the
  bug is body re-stringification.
