---
name: chatia
description: Use this skill when working with Chatia — a SaaS that builds AI agents (chatbot flows, WhatsApp agents, Instagram/Messenger/Telegram agents, sales/support agents) and exposes a developer REST API at /api/v1 + /api/developers + an OpenAI-compatible /api/v1/chat/completions endpoint with HMAC-signed webhooks. Covers (1) onboarding from zero — registration + API key creation + setup; (2) end-to-end recipes to CREATE agents from external code with curl + Python + Node, where one POST returns chat_url + dashboard_url + widget_snippet + wordpress_plugin_url ready to paste; (3) deep-dive on CONSUMING webhooks — HMAC signature verification, replay protection, full event catalog (27 types including agent.sale.*); (4) rate limits per API key (30/min for agent creation, 120/min for events); (5) embed targets HTML / WordPress (official plugin) / Shopify / Wix; (6) 25+ OAuth integrations via Composio. Use whenever the user mentions creating, deploying, or programmatically managing a Chatia agent, embedding the Chatia chat widget, consuming Chatia events, or building UI/integrations against the Chatia API.
version: 0.4
---

# Chatia · Skill for AI agents

Use this file to let Claude Code, Codex, Cursor, GPT agents and similar tools
operate Chatia from the API.

> **¿Solo querés "instrucciones operativas paste-ready" para tu agente IA?**
> Hay un destilado más corto en
> [`/AGENTS.md`](https://www.chatia.pro/AGENTS.md) (~200 LOC, formato
> system-prompt ready). Este SKILL.md es la referencia completa
> (~700 LOC, modelos de datos + recetas multi-lenguaje + verificación de
> firma + endpoints exhaustivos). Pegá cualquiera de los dos a tu
> Claude / Cursor / Codex como contexto.

## What is Chatia

Chatia spins up AI agents that freelancers deploy to their own clients in
minutes. Every agent gets:

- A **webchat** at `/{public_slug}` and an **orbe widget** (`/widget.js`) for
  any website.
- A **WhatsApp connection** (Kapso BYOK) — optional.
- **CRM integrado** — kanban de leads (`new → contacted → qualified → won
  | lost`) con drag-drop + edición timestamped. Las tools `update_lead_status`
  y `add_lead_note` dejan al agente mover leads en el pipeline conversando.
- **Calendario** con anti-overlap automático (`create_calendar_event`
  rechaza horarios pisados), `cancel_appointment`, `reschedule_appointment`,
  y **recordatorios automáticos por email** 24h y 1h antes (FROM "CITA
  CHATIAI" via Resend).
- **HR / Recruitment module** — `JobPosition` con competencias ponderadas,
  `Candidate` con scoring 0-10 por competencia, `InterviewSession` con
  resumen + recomendación. Kanban dedicado en el dashboard.
- **NPS** — encuesta 0-10 al cierre de conversación con score agregado en
  stats.
- **Builder conversacional** — crea el agente en 1 mensaje y lo refina
  conversando (`update_agent_meta`, `set_agent_tools`, etc.) — nunca
  recrea para arreglar.
- **Tools**: 18+ tools nativas más `http_request` para integraciones
  custom. Inventario completo en sección "Agent runtime tools" abajo.
- **Usage-based billing** via Polar (metered events per message,
  $0.010 managed / $0.004 BYOK, plus paid plans Starter/Pro/Studio/Agency).
- **Per-conversation human takeover** — flip `ai_paused=true` and reply as
  operator; the AI pauses for that thread.

## Base URL

All API calls go to the backend base URL, which is configured per environment:

- Local dev: `http://localhost:8000`
- Production: whatever you put in `NEXT_PUBLIC_API_URL` / `BACKEND_URL`.

**Do not hardcode** domains. Use the env var.

## Authentication

Two kinds of bearer tokens:

1. **User JWT** — emitted by `POST /api/auth/login`. Used by the dashboard
   app. Scope: the logged-in user's resources.
2. **Developer API key** — create one at `/dashboard/developers`. Scope: the
   user's agents + events ingestion. Prefix is `chatia_…`.

Both go in the `Authorization: Bearer <token>` header.

## Core API

### Agents

| Verb   | Path                                  | Notes                                                |
| ------ | ------------------------------------- | ---------------------------------------------------- |
| GET    | `/api/agents`                         | Your agents (paginated summary).                     |
| GET    | `/api/agents/{id}`                    | Full detail with tools.                              |
| PATCH  | `/api/agents/{id}`                    | Update identity, model, API keys, Kapso config.      |
| POST   | `/api/agents/{id}/publish`            | Publish and mint a `public_slug`.                    |
| POST   | `/api/agents/{id}/tools`              | Attach a builtin tool.                               |
| DELETE | `/api/agents/{id}`                    | Destructive — body `{ confirmation: "<agent name>" }`. |

### Builder (create agents by chat)

| Verb | Path                   | Notes                                      |
| ---- | ---------------------- | ------------------------------------------ |
| POST | `/api/builder/stream`  | SSE. Body: `{ message, images[], thread_id }`. |

Events emitted: `thread`, `plan_created`, `plan_step_updated`, `tool_called`,
`tool_output`, `external_action`, `text_delta`, `done`, `error`. The model is
`gpt-5.4`. Plans must use descriptive task names (not "Paso 1").

### Public webchat (no auth)

| Verb | Path                                            |
| ---- | ----------------------------------------------- |
| GET  | `/api/public/agents/{slug}`                     |
| POST | `/api/public/agents/{slug}/chat`                |
| POST | `/api/public/agents/{slug}/leads`               |

Chat body: `{ message, history, conversation_id?, visitor_id? }`.
Response: `{ reply, conversation_id, key_source, free_messages_remaining, human_takeover? }`.

### Conversations + human takeover

| Verb | Path                                                                               |
| ---- | ---------------------------------------------------------------------------------- |
| GET  | `/api/clients/me/agents/{id}/conversations`                                        |
| GET  | `/api/clients/me/agents/{id}/conversations/{cid}`                                  |
| POST | `/api/clients/me/agents/{id}/conversations/{cid}/takeover` · body `{ paused: bool }` |
| POST | `/api/clients/me/agents/{id}/conversations/{cid}/messages` · body `{ content }`    |

Owners and clients (sub-users with a grant) can read these; the backend gates
by `ClientAgentGrant.scopes` (`messages` / `leads`).

### CRM — Leads

Leads are captured by the agent (via `capture_lead` tool) or via the public
endpoint. The owner sees a kanban with 5 stages (`new → contacted → qualified
→ won | lost`) at `/dashboard/agents/{id}/conversations` (tab Leads). The
agent can move leads in the pipeline via `update_lead_status` and append
notes via `add_lead_note`.

| Verb  | Path                                                                  | Notes |
| ----- | --------------------------------------------------------------------- | ----- |
| GET   | `/api/clients/me/agents/{id}/leads?status=qualified`                  | filtro opcional por stage |
| PATCH | `/api/clients/me/agents/{id}/leads/{lead_id}`                         | `{ status?, name?, email?, phone?, interest?, note? }` — note se apendea timestamped al campo `notes` |

The lead model has `status` (free-form string), `notes` (Text, append-only
timestamped), `updated_at`. Status changes from manual edits get logged with
`(manual)` suffix in `notes` to distinguish from agent-driven moves.

### Recruitment — HR module

Vertical RRHH for interviewer agents. Pipeline: `applied → screening →
interviewed → finalist → hired | rejected`. Three models:

- **JobPosition** — vacante (title, description, competencies as
  `[{key, label, weight}]`).
- **Candidate** — postulante (name, email, phone, cv_url, position_id,
  status, score_avg cached, notes append-only).
- **InterviewSession** — atada a `Candidate` + `Conversation`. Scoring por
  competencia: `{competency_key: {score: 0-10, note}}`. Summary +
  `final_recommendation` (`advance` | `reject` | `hold`).

UI live at `/dashboard/agents/{id}/conversations` (tab RRHH): chip-bar of
positions on top + drag-drop kanban of candidates. Click a card to open the
detail drawer with sessions + scoring per competency.

| Verb   | Path                                                          | Notes |
| ------ | ------------------------------------------------------------- | ----- |
| GET    | `/api/agents/{id}/positions`                                  | list |
| POST   | `/api/agents/{id}/positions`                                  | `{ title, description?, competencies?: [{key, label, weight?}] }` |
| PATCH  | `/api/agents/{id}/positions/{pid}`                            | partial — `status: open | closed`, etc. |
| DELETE | `/api/agents/{id}/positions/{pid}`                            | candidates keep `position_id=NULL` (SET NULL) |
| GET    | `/api/agents/{id}/candidates?status=&position_id=`            | filtros opcionales |
| GET    | `/api/agents/{id}/candidates/{cid}`                           | incluye `sessions[]` con scoring detallado |
| PATCH  | `/api/agents/{id}/candidates/{cid}`                           | `{ status?, name?, email?, phone?, cv_url?, position_id?, note? }` |
| DELETE | `/api/agents/{id}/candidates/{cid}`                           | borra el candidate + sus sessions (CASCADE) |

### Agent runtime tools (live agents)

Tools que el agente publicado puede invocar durante una conversación. Están
declaradas en `BUILTIN_TOOL_LIBRARY` (backend) y se attachean al agente via
`POST /api/agents/{id}/tools` (manual) o el builder (`attach_builtin_tools`).

#### Captura y CRM

- **`capture_lead`** `{ name?, email?, phone?, interest }` — guarda lead
  básico. Devuelve `lead_id`.
- **`update_lead_status`** `{ lead_id, status, note? }` — mueve el lead en
  el pipeline (`new → contacted → qualified → won | lost`). Si pasa `note`
  la apendea al historial.
- **`add_lead_note`** `{ lead_id, note }` — apendea nota timestamped sin
  cambiar status.

#### Calendario

- **`create_calendar_event`** `{ title, start_at (ISO), duration_minutes?,
  contact_*?, reason? }` — crea cita. **Valida overlap** automáticamente
  contra otras citas activas; si choca devuelve `{ ok: false, error:
  "slot_taken", conflict_appointment_id, conflict_start_at, message }` y el
  agente debe proponer otro slot.
- **`list_availability`** `{ from_date, to_date }` — devuelve citas activas
  en el rango. Útil para que el agente proponga huecos libres antes de
  pedirle datos al cliente.
- **`cancel_appointment`** `{ appointment_id, reason? }` — marca
  `status="canceled"` y appendea la razón al campo `reason`.
- **`reschedule_appointment`** `{ appointment_id, new_start_at,
  new_duration_minutes? }` — mueve la cita con anti-overlap (excluyendo a
  sí misma).

#### Recordatorios automáticos

Si `Appointment.contact_email` está seteado, el sistema dispara emails
recordatorios automáticamente:
- **24h antes** de la cita
- **1h antes** de la cita

FROM display name: **"CITA CHATIAI"** (sobre el address verificado de
plataforma). El loop chequea cada 10 min y deduplica via
`Appointment.meta["reminders_sent"]`. Skipea citas con status
`canceled | no_show | completed` o sin `contact_email`. Idempotency
garantizada via `idempotency_key=appt-reminder-{id}-{window}` en Resend.

#### Comunicación

- **`send_email`** `{ to, subject, body }` — vía Resend o SMTP. Requiere
  config en `agent.config.email` (set desde `/dashboard/agents/{id}#email`).
- **`send_whatsapp_handoff`** `{ reason, summary? }` — emite el evento
  `agent.handoff.requested` por webhook al equipo del owner.
- **`send_whatsapp_text`** `{ phone, text }` — texto libre vía Kapso. Sólo
  válido dentro de la ventana de 24h del último inbound del cliente.
- **`whatsapp_send_template`** `{ template_name, phone, variables? }` —
  plantilla pre-aprobada (afuera de la ventana 24h).
- **`send_whatsapp_image`** `{ image_url (https), phone, caption? }` —
  imagen pública via Kapso.

#### Knowledge

- **`query_knowledge`** `{ query, limit? }` — búsqueda LIKE en
  `agent_knowledge` chunks.

#### Encuestas

- **`send_nps_survey`** `{ intro? }` — manda link único 0-10. El intro
  debe contener el placeholder literal `{survey_url}` que el sistema
  reemplaza por el link real. Una vez por conversación.

#### Recruitment / HR (sólo agentes entrevistadores)

- **`register_candidate`** `{ name, email?, phone?, position_id?, cv_url? }`
  — crea Candidate + InterviewSession atada a la conversación. Devuelve
  `candidate_id`.
- **`score_candidate`** `{ candidate_id, competency_key, score (0-10),
  note? }` — apendea/updatea scoring de la sesión activa para esta conv.
  Si no hay sesión la crea lazy. Recalcula `Candidate.score_avg`
  automáticamente como promedio simple sobre todas las sesiones.
- **`flag_for_review`** `{ candidate_id, status?, recommendation?,
  summary? }` — cierra la entrevista: actualiza `status` (loggea cambio
  en notes), cierra la sesión con `summary` + `final_recommendation`
  (`advance | reject | hold`).

#### HTTP genérico

- **`http_request`** `{ endpoint (https), method?, payload? }` — escapa
  para integraciones custom del owner.

### Custom HTTP webhook tools (feature de poder — el owner define sus propias tools)

Cuando los builtins (~25 tools nativas) y Composio (1000+ apps) no
cubren un caso del owner, este puede definir **tools custom** que pegan
HTTP a su propia API. Ejemplo: tool `crear_pedido(producto, cantidad,
cliente)` que el agente invoca y dispara `POST https://miapi.com/orders`
con un Bearer token.

El agente las ve como tools nativas — invoca `crear_pedido` y el
backend de Chatia hace el HTTP request al endpoint del owner con los
args que el LLM le pasó + auth + `_meta` injectado.

#### Modelo de datos

Filas en `agent_tools` con `config.kind == "http_webhook"`:

```jsonc
{
  "kind": "http_webhook",
  "url": "https://api.miempresa.com/orders",
  "method": "POST",                     // GET | POST | PUT | PATCH | DELETE
  "headers": { "X-Tenant": "acme" },     // opcional
  "auth": {                               // opcional
    "type": "bearer",                     // "bearer" | "header"
    "token": "secret_xxx",                // jamás se expone en GET responses
    "header_name": "X-API-Key"            // solo si type=header
  },
  "timeout_s": 15,                        // default 15, max 60
  "include_meta": true                    // default true → inyecta
                                          // _meta:{agent_id,conversation_id} al body
}
```

El campo `schema` de `agent_tools` (JSON) guarda el JSON Schema de los
`parameters` que el LLM debe pasarle a la tool — exactamente igual que
un builtin.

#### Endpoints CRUD (auth: session JWT del owner)

```
GET    /api/agents/{id}/custom-tools                 — list (oculta el token)
POST   /api/agents/{id}/custom-tools                 — create (valida URL, no permite collision con builtins)
PATCH  /api/agents/{id}/custom-tools/{tool_id}       — update (preserva token si viene vacío)
DELETE /api/agents/{id}/custom-tools/{tool_id}       — delete
```

#### Ejemplo: crear una tool custom

```bash
curl -X POST https://www.chatia.pro/api/agents/42/custom-tools \
  -H "Authorization: Bearer <SESSION_JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "crear_pedido",
    "display_name": "Crear pedido",
    "description": "Crea un pedido en el ERP del owner cuando el cliente confirma compra.",
    "schema": {
      "type": "object",
      "properties": {
        "producto": { "type": "string" },
        "cantidad": { "type": "integer" },
        "cliente_email": { "type": "string" }
      },
      "required": ["producto", "cantidad", "cliente_email"]
    },
    "config": {
      "kind": "http_webhook",
      "url": "https://api.miempresa.com/orders",
      "method": "POST",
      "auth": { "type": "bearer", "token": "secret_xxx" },
      "timeout_s": 15,
      "include_meta": true
    }
  }'
```

Cuando el agente invoca `crear_pedido({producto:"X", cantidad:2, cliente_email:"foo@bar"})`,
el backend hace:

```
POST https://api.miempresa.com/orders
Authorization: Bearer secret_xxx
Content-Type: application/json

{
  "producto": "X",
  "cantidad": 2,
  "cliente_email": "foo@bar",
  "_meta": { "agent_id": 42, "agent_name": "Soporte", "conversation_id": 108 }
}
```

#### Defensas baked-in

- **Anti-SSRF**: bloquea `localhost`, `127.0.0.1`, `0.0.0.0`, `169.254.x`
  (link-local), IPs decimales / hex, DNS rebinding.
- **Method whitelist**: GET / POST / PUT / PATCH / DELETE.
- **Auth**: Bearer (`Authorization: Bearer ...`) o header custom
  (`X-API-Key: ...`).
- **Body cap**: response truncado a 4000 chars antes de pasarlo al LLM.
- **Timeout**: 60s máx.
- **Token nunca leakea**: el GET de la tool oculta `auth.token`.

#### UI: visual builder de schema (sin pedir JSON Schema raw al user)

`/dashboard/agents/{id}#customTools` tiene un editor visual: el owner
agrega filas con nombre + tipo (Texto / Número / Entero / Sí-No) +
descripción + checkbox "obligatorio". Eso se traduce a JSON Schema
internamente. Un AI agent externo puede generar este JSON Schema desde
una descripción en lenguaje natural del user.

### Composio integrations (el owner conecta apps externas con OAuth)

Para conectar el agente a Gmail / Calendar / Sheets / Slack / Notion /
HubSpot / Stripe / Discord / Telegram / Facebook / Instagram / Messenger
y +1000 apps del catálogo de Composio. Modelo **session-based** con
**5 meta-tools** — el LLM no ve los 1000+ slugs sueltos.

#### Modelo de auth — OWNER conecta, NO end-user

Las credenciales son del **owner** del agente (la clínica conecta SU
calendar; cuando un paciente pide turno, el agente escribe en EL CALENDAR
DE LA CLÍNICA). Los visitantes nunca ven OAuth flows.

#### Endpoints

```
GET   /api/integrations/toolkits                                  — catálogo público de toolkits soportados
GET   /api/agents/{id}/toolkits                                   — estado por toolkit del agente (connected, pending, etc.)
POST  /api/agents/{id}/toolkits/{slug}/connect                    — inicia OAuth → devuelve {redirect_url}
DELETE /api/agents/{id}/toolkits/{slug}/connection                — desconecta + revoca
PATCH /api/agents/{id}/toolkits/{slug}                            — toggle is_active

GET   /api/integrations/oauth/callback?connectedAccountId=&agent_id=&slug=
                                                                  — callback (Composio redirige acá)
POST  /api/integrations/composio/webhook                          — receiver triggers Composio
GET/POST /api/integrations/meta/webhook                           — receiver Meta directo (FB/IG/Messenger)
POST  /api/integrations/telegram/webhook/{agent_id}               — receiver Telegram
```

#### Flujo OAuth (3 pasos)

```
1. POST /api/agents/{id}/toolkits/gmail/connect
   → response: { redirect_url: "https://accounts.google.com/o/oauth/..." }

2. Frontend hace window.location = redirect_url
   → user autoriza con su Google

3. Composio redirige a /api/integrations/oauth/callback?...
   → backend marca connection_status='connected' + populates page_ids (Meta)
   → redirect al panel del agente con banner verde
```

#### Catálogo de toolkits soportados (25+ apps)

```
Mensajería:  whatsapp (vía Kapso, no Composio), telegram, discord, slack
Email:       gmail, outlook
Calendar:    googlecalendar, googlemeet
Spreads:     googlesheets, excel
Files:       googledrive, googledocs, googleslides, googleforms,
             googletasks, onedrive
Productiv.:  notion, airtable, linear, github, youtube
M365:        microsoft_teams
CRM:         hubspot
Pagos:       stripe
Analytics:   google_analytics
Meta:        facebook (cubre Pages + IG Business + Messenger en 1 OAuth)
```

#### Meta-tools que ve el LLM (5 funciones, no 1000)

```
COMPOSIO_SEARCH_TOOLS        — búsqueda semántica de actions ("send email")
COMPOSIO_GET_TOOL_SCHEMAS    — schema detallado de una action
COMPOSIO_MULTI_EXECUTE_TOOL  — ejecuta 1+ actions en paralelo
COMPOSIO_REMOTE_WORKBENCH    — Python sandbox para post-processing
COMPOSIO_REMOTE_BASH_TOOL    — bash para data extraction
```

#### Billing ×5 markup

Composio cobra por execute_action upstream (~$0.0008 Sheets, ~$0.0012
Gmail, ~$0.0010 Calendar). Chatia aplica ×5 markup → cliente paga
~$0.004-$0.006 por action. Pre-flight checkea balance antes de ejecutar.

#### Triggers (push de eventos entrantes)

Para los toolkits que SÍ exponen Triggers (Slack, Gmail, GitHub),
Composio postea eventos a `POST /api/integrations/composio/webhook`
firmados con HMAC. Discovery dinámico por patrones de naming
(`NEW_*MESSAGE`, `MESSAGE_RECEIVED`, etc.) — sin hardcodeo de slugs.

### Meta direct webhooks (FB / IG / Messenger inbound)

Composio expone **Triggers=0 para Meta toolkits**. Los comments en
posts FB / DMs Messenger / DMs IG llegan vía webhooks **directo de
Meta** a Chatia, no via Composio.

#### Arquitectura multi-tenant

```
PLATAFORMA (1 vez, por chatia)        CLIENTES (cada owner)
─────────────────────────────         ──────────────────────────
developers.facebook.com                /dashboard/agents/X#integrations
  ├── 1 Meta App                        ├── Click "Conectar Facebook"
  ├── webhook URL                       ├── OAuth con TU Meta App
  ├── App Review aprobada               │   (managed por Composio
  └── App Secret en env                 │    con custom OAuth)
                                        ├── Eligen sus Pages
composio.dev dashboard                  └── Listo. Sus eventos
  └── Auth Config Facebook                  llegan a chatia.pro
      "Use custom OAuth"
        ├── Meta App ID
        └── Meta App Secret
```

Una sola Meta App recibe events de TODAS las Pages que clientes
autoricen via OAuth. Cada event payload trae `entry[].id = page_id`
que Chatia rutea al `agent_id` que tenga esa Page conectada
(`agent_toolkits.metadata_json.page_ids`).

#### Auto-subscribe Pages al webhook

Después del OAuth, Chatia llama `FACEBOOK_LIST_MANAGED_PAGES` para
descubrir Pages, persiste sus IDs + access_tokens, y suscribe
automáticamente cada Page al webhook de la Meta App con
`POST graph.facebook.com/v19.0/{page_id}/subscribed_apps?subscribed_fields=feed,messages,messaging_postbacks`.
Cliente NO tiene que entrar a Meta Business → Webhook Subscriptions
manualmente.

#### Flow del event entrante

```
1. Visitante comenta en post FB / manda DM a Page
2. Meta hace POST /api/integrations/meta/webhook (HMAC X-Hub-Signature-256)
3. verify_meta_signature() valida con META_APP_SECRET
4. process_meta_webhook() rutea entry.id → agent_toolkit (page_ids[])
5. Construye trigger slug sintético:
   FACEBOOK_NEW_PAGE_COMMENT / MESSENGER_NEW_MESSAGE /
   INSTAGRAM_NEW_DM / INSTAGRAM_NEW_COMMENT
6. dispatch_event() → run_public_agent (synthetic prompt)
7. Agente responde via COMPOSIO_MULTI_EXECUTE_TOOL con la action
   correspondiente:
     · FACEBOOK_CREATE_COMMENT (object_id en formato pageId_commentId)
     · FACEBOOK_SEND_MESSAGE
     · INSTAGRAM_REPLY_TO_COMMENT
     · INSTAGRAM_SEND_DIRECT_MESSAGE
```

#### Filtros importantes

- Skipea `sender.id == page.id` o comentario de la propia Page
  (anti-loop del agente respondiéndose a sí mismo).
- Skipea `verb != "add"` o `item != "comment"` en field=feed (ignora
  edits/reactions, solo procesa comments nuevos).

#### Costo del modelo

| Evento | Cobro |
|---|---|
| Cliente conecta FB/IG/Messenger | $0 |
| Llega comment / DM (Meta empuja) | $0 (Meta no cobra webhooks) |
| Agente RESPONDE | markup ×5 sobre Composio action (`FACEBOOK_SEND_MESSAGE` ~$0.005, etc.) |
| Conexión inactiva (nadie escribe) | $0 absoluto |

A diferencia del polling (fee fijo aunque no haya eventos), el modelo
webhook escala con USO real.

### Telegram direct webhooks (mismo patrón que Meta)

Telegram tampoco tiene Composio Triggers. Endpoint
`POST /api/integrations/telegram/webhook/{agent_id}` con verify de
header `X-Telegram-Bot-Api-Secret-Token`.

**Setup manual del owner**: Composio NO expone setWebhook como tool.
El owner debe ejecutar `curl https://api.telegram.org/bot<TOKEN>/setWebhook?url=...&secret_token=...`
manualmente. Chatia guarda el secret_token + webhook_url en
`metadata_json` para validar firma de webhooks entrantes.

### Builder — crear y EDITAR el agente conversando

El builder (POST `/api/builder/stream`) sigue la filosofía
**crear-primero, iterar-después**. Apenas el primer mensaje da pista
mínima del caso de uso, dispara `plan_tasks → create_agent_draft →
attach_builtin_tools → publish_webchat`. A partir del turno 2 **NO
recrea** el agente — usa las tools de iteración para refinarlo conversando.

#### Tool-set default según vertical inferido

| Vertical | Tools default |
| -------- | ------------- |
| Ventas | `capture_lead`, `send_whatsapp_handoff`, `update_lead_status`, `add_lead_note` |
| Soporte/FAQ | `query_knowledge`, `send_whatsapp_handoff`, `send_nps_survey` |
| Turnos/Agenda | `create_calendar_event`, `list_availability`, `cancel_appointment`, `reschedule_appointment`, `capture_lead` |
| Vacante/RRHH | `register_candidate`, `score_candidate`, `flag_for_review`, `send_email` |

#### Tools de iteración del builder (sobre agente ya creado)

Estas tools NO consumen agent quota — un user en límite puede iterar sobre
agentes existentes sin pagar más, sólo no puede crear nuevos.

- **`update_agent_meta`** `{ agent_id, name?, description?, system_prompt?,
  first_message?, language?, model? }` — patch parcial. Sólo toca los
  fields no-`null`.
- **`set_agent_tools`** `{ agent_id, tool_names[] }` — reemplaza el set
  ENTERO. Idempotente: borra las que sobran, agrega las nuevas.
- **`attach_builtin_tools`** `{ agent_id, tool_names[] }` — additive
  (alias del anterior pero sólo agrega).
- **`detach_tools`** `{ agent_id, tool_names[] }` — saca tools específicas.
- **`get_agent_summary`** `{ agent_id }` — devuelve fields + tools
  attachadas + branding. Llamala antes de proponer cambios para no pisar
  config existente.
- **`update_agent_branding`** `{ agent_id, primary_color?, theme? }` —
  branding del webchat.
- **`add_knowledge_chunk`** / **`import_knowledge_url`** — carga del KB.

#### Mapping pedidos naturales → tools

| El user dice… | Builder llama |
| ------------- | ------------- |
| "el tono es muy formal, hacelo más cálido" | `update_agent_meta(system_prompt=...)` |
| "cambiale el nombre a X" | `update_agent_meta(name="X")` |
| "agregale email" | `attach_builtin_tools(["send_email"])` |
| "saquemos el handoff" | `detach_tools(["send_whatsapp_handoff"])` |
| "que sea verde / modo claro" | `update_agent_branding(...)` |
| "agregale info: …" | `add_knowledge_chunk` |
| "qué tiene puesto ahora?" | `get_agent_summary` |

### Billing

| Verb | Path                              | Notes                                              |
| ---- | --------------------------------- | -------------------------------------------------- |
| GET  | `/api/billing/plans`              | Public. Returns all plans + rates + seat price.    |
| GET  | `/api/billing/account`            | Current user's plan, usage, PM, agents used/limit. |
| POST | `/api/billing/checkout-session`   | `{ plan_slug }` → `{ url }` for PolarEmbedCheckout. |
| POST | `/api/billing/portal-session`     | → `{ url }` to open Polar Customer Portal.         |

### Developers

| Verb | Path                                                          | Rate limit |
| ---- | ------------------------------------------------------------- | ---------- |
| GET  | `/api/developers/overview`                                    | sin límite explícito |
| GET  | `/api/developers/usage?days=14`                               | sin límite explícito |
| GET  | `/api/developers/skills`                                      | sin límite explícito |
| GET  | `/api/developers/events/catalog`                              | sin límite explícito |
| POST | `/api/developers/api-keys`                                    | sin límite explícito |
| POST | `/api/developers/api-keys/{id}/reveal`                        | sin límite explícito |
| POST | `/api/developers/api-keys/{id}/regenerate`                    | sin límite explícito |
| POST | `/api/developers/webhooks`                                    | sin límite explícito |
| PATCH| `/api/developers/webhooks/{id}`                               | sin límite explícito |
| POST | `/api/developers/webhooks/{id}/test`                          | **20/min por API key** |
| GET  | `/api/developers/agents`                                      | sin límite explícito |
| POST | `/api/developers/agents`                                      | **30/min por API key** |
| POST | `/api/developers/agents/{id}/publish`                         | **30/min por API key** |
| GET  | `/api/developers/agents/{id}/conversations?limit=50&offset=0` | sin límite explícito |
| GET  | `/api/developers/conversations/{id}`                          | sin límite explícito |
| GET  | `/api/developers/conversations/{id}/export?format=json|csv|txt`| sin límite explícito |
| POST | `/api/developers/events`                                      | **120/min por API key** |

**Rate limiting**: los endpoints sensibles (que disparan side-effects: crear/publicar agente, fan-out de webhooks, emisión de eventos) tienen límite por API key — no por IP, así varios devs detrás del mismo NAT corporativo no comparten cupo. Cuando excedés el límite recibís `429 Too Many Requests` con header `Retry-After` en segundos. Los demás endpoints no tienen límite explícito hoy pero pueden agregarse — diseñá tu integración con backoff exponencial defensivo.

### Knowledge base (REST v1)

Los agentes pueden tener una base de conocimiento que el tool `query_knowledge` consulta. Estos endpoints permiten cargarla por API:

| Verb   | Path                                                     | Notes |
| ------ | -------------------------------------------------------- | ----- |
| GET    | `/api/v1/agents/{id}/knowledge`                          | listar chunks |
| POST   | `/api/v1/agents/{id}/knowledge`                          | `{ title, content }` — chunk manual |
| PATCH  | `/api/v1/agents/{id}/knowledge/{chunk_id}`               | actualizar chunk |
| DELETE | `/api/v1/agents/{id}/knowledge/{chunk_id}`               | borrar chunk |
| POST   | `/api/v1/agents/{id}/knowledge/upload-file` (multipart)  | PDF/TXT — extrae texto + chunks |
| POST   | `/api/v1/agents/{id}/knowledge/import-url`               | `{ url }` — fetch + chunks |

### Clients (sub-users) — public API

A developer/owner can manage their entire client roster via API key. Clients
are end-users invited to the `/client` portal to read messages, leads and
take human takeover. Each client can also invite up to 5 team-members
(seats) so an entire team can split the work — `team_members` cannot invite
to more, cannot see metrics, and don't appear in this list.

| Verb   | Path                                                          | Notes |
| ------ | ------------------------------------------------------------- | ----- |
| GET    | `/api/developers/clients`                                     | list |
| POST   | `/api/developers/clients`                                     | { email, name, password, agent_ids[] } — emits `client.created`, sends bienvenida + tip email |
| PATCH  | `/api/developers/clients/{id}`                                | { name?, is_active? } — emits `client.updated` |
| DELETE | `/api/developers/clients/{id}`                                | irreversible — emits `client.removed` |
| POST   | `/api/developers/clients/{id}/reset-password`                 | returns new password ONCE — emits `client.password_reset` |
| PATCH  | `/api/developers/clients/{id}/branding`                       | brand_name, primary_color, powered_by_text/url, show_powered_by — emits `client.branding.updated` |
| GET    | `/api/developers/clients/{id}/team-members`                   | listar seats |
| POST   | `/api/developers/clients/{id}/team-members`                   | { email, name, password } — cap=5, requires owner plan != free |
| DELETE | `/api/developers/clients/{id}/team-members/{member_id}`       | emits `client.team_member.removed` |
| PATCH  | `/api/developers/clients/{id}/features`                       | { dashboard_enabled? } — habilita/deshabilita el dashboard del cliente |

### Clients (session-auth equivalents — used by the dashboard)

Same shape; useful when scripting the in-app dashboard:

| Verb | Path                                              |
| ---- | ------------------------------------------------- |
| GET  | `/api/clients`                                    |
| POST | `/api/clients`                                    |
| PATCH| `/api/clients/{id}`                               |
| DELETE | `/api/clients/{id}`                             |
| POST | `/api/clients/{id}/reset-password`                |
| PATCH| `/api/clients/{id}/branding`                      |
| POST | `/api/clients/{id}/branding/logo` (multipart)     |
| DELETE | `/api/clients/{id}/branding/logo`               |
| PATCH| `/api/clients/{id}/features`                      |
| GET  | `/api/clients/me/dashboard?days=N`                |
| POST | `/api/clients/{id}/grants`                        |
| DELETE | `/api/clients/{id}/grants/{agent_id}`           |
| GET  | `/api/clients/me/agents`                          |
| GET  | `/api/clients/me/team-members`                    |
| POST | `/api/clients/me/team-members`                    |
| DELETE | `/api/clients/me/team-members/{id}`             |

### Branding (white-label)

Two zones, both with sane Chatia defaults if not configured:

| Verb   | Path                                              | Who    |
| ------ | ------------------------------------------------- | ------ |
| GET    | `/api/branding/agents/{id}`                       | owner  |
| PATCH  | `/api/branding/agents/{id}`                       | owner  |
| POST   | `/api/branding/agents/{id}/logo` (multipart)      | owner  |
| DELETE | `/api/branding/agents/{id}/logo`                  | owner  |
| GET    | `/api/branding/portal`                            | owner  |
| PATCH  | `/api/branding/portal`                            | owner  |
| POST   | `/api/branding/portal/logo` (multipart)           | owner  |
| DELETE | `/api/branding/portal/logo`                       | owner  |
| GET    | `/api/branding/client`                            | client / team_member — devuelve el branding aplicado al portal |
| GET    | `/api/branding/logo/{kind}/{filename}`            | público |

Plan gating:

- **Free** → `show_powered_by` siempre forzado a `true` server-side.
- **PayG / Starter / Pro / Studio / Agency** → el toggle es respetado.

### Profile / 2FA (any logged-in user)

| Verb | Path                          | Notes |
| ---- | ----------------------------- | ----- |
| GET  | `/api/auth/me`                | quién soy |
| PATCH| `/api/auth/me`                | { name } |
| POST | `/api/auth/change-password`   | { current_password, new_password } |
| POST | `/api/auth/2fa/setup`         | devuelve QR + secret |
| POST | `/api/auth/2fa/enable`        | { code } 6 dígitos |
| POST | `/api/auth/2fa/disable`       | { password } |

### Admin (superadmin only)

| Verb  | Path                                 |
| ----- | ------------------------------------ |
| GET   | `/api/admin/metrics`                 |
| GET   | `/api/admin/users`                   |
| GET   | `/api/admin/users/{id}`              |
| PATCH | `/api/admin/users/{id}/status`       |
| POST  | `/api/admin/users/{id}/impersonate`  |
| POST  | `/api/admin/billing/report-usage`    |
| GET   | `/api/admin/billing/config`          |
| GET   | `/api/admin/voice`                   |
| PATCH | `/api/admin/voice`                   |
| POST  | `/api/admin/voice/users/grant-minutes` |
| POST  | `/api/admin/voice/sessions/cleanup-orphans` |

### Voice Control (OpenAI Realtime API)

Conversational voice interface for the dashboard. Owner habla, el orbe
ejecuta tools contra la API con su JWT. Owner-only (clients y team_members
reciben 403). Tres minutos managed por owner como trial vitalicio; después
BYOK obligatorio (owner usa su `UserApiSettings.openai_api_key`).

**Costo cap-protected**: `VOICE_TRIAL_DAILY_USD_CAP=5` y
`VOICE_TRIAL_MONTHLY_USD_CAP=20` aseguran un peor mes platform-wide
acotado. Override desde DB en `platform_flags` via `/admin/voice` sin
redeploy. Sesiones huérfanas (sin `/end` call) se cuentan pesimistamente
en el cap para evitar fugas.

| Verb | Path | Notes |
| ---- | ---- | ----- |
| GET  | `/api/voice/status` | Lo que el orbe lee al montar — enabled, has_byok, trial_remaining, caps_remaining_usd. Para no-owners devuelve enabled=false. |
| POST | `/api/voice/session` | Mintea ephemeral client_secret de OpenAI Realtime. Aplica gates en orden: kill switch global → owner role → BYOK del owner si está configurada → trial vitalicio si no → cap diario USD → cap mensual USD → no-concurrent-session por user. |
| POST | `/api/voice/session/{id}/end` | Cierra y persiste duration + cost_micro_usd. Si fue managed, suma al voice_trial_seconds_used del owner. Idempotente. |
| GET  | `/api/voice/tools/catalog` | Lista de tools shape OpenAI-compatible para inyectar al crear la session. |
| POST | `/api/voice/tools/exec` | Dispatcher único — `{name, arguments}` ejecuta una de las 18 tools disponibles. Devuelve `{ok, result, voice_summary}` listo para que el modelo lea. |

**Tools disponibles** (todas owner-only, todas con auditoría):

| Categoría | Tools |
| --- | --- |
| Crear | `create_agent`, `create_api_key`, `create_webhook`, `create_client` |
| Listar | `list_agents`, `list_clients`, `list_api_keys`, `list_webhooks`, `get_recent_conversations` |
| Editar | `update_agent`, `publish_agent` |
| Destructivas (requieren `confirm=true`) | `delete_agent`, `revoke_api_key`, `reset_client_password` |
| Otras | `get_account_status`, `query_knowledge` (RAG sobre SKILL.md), `navigate_to` |

**Reglas de seguridad para el voice agent**:
- **Secretos nunca leídos en voz**: el handler los marca `show_in_panel=true` y el orbe los renderiza en un overlay con CopyButton. La voz solo dice "te lo dejé en pantalla".
- **Tools destructivas**: el modelo debe pedir confirmación verbal explícita antes de invocar y pasar `confirm=true` solo si el usuario confirmó.
- **Owner-only**: hardcoded server-side en `_require_owner()`. Clients y team_members reciben 403 voice_owner_only.

## Events & webhooks

Outbound webhooks (HMAC-SHA256 signed, retries con back-off exponencial).
**Single source of truth**: `GET /api/developers/events/catalog` devuelve la
lista canónica con descripción y grupo. Categorías:

**Agent lifecycle**
- `agent.created`, `agent.published`
- `agent.branding.updated` — logo / colores / "Powered by" del webchat

**Conversation (web + WhatsApp)**
- `agent.message.created`, `agent.reply.created`
- `agent.lead.captured`
- `agent.tool.called` — incluye `{tool, arguments, ok}`
- `agent.handoff.requested`

**WhatsApp delivery status**
- `agent.message.delivered`, `agent.message.read`, `agent.message.failed`
- `agent.outbound.skipped_24h_window`
- `agent.phone_number.quality_changed`, `agent.phone_number.banned`

**Clientes (sub-users del owner)**
- `client.created`, `client.updated`, `client.removed`
- `client.password_reset` — payload NO incluye la nueva password
- `client.branding.updated`
- `client.agent_grant.added`, `client.agent_grant.removed`

**Equipo del cliente (los 5 seats)**
- `client.team_member.added`, `client.team_member.removed`

**Branding global**
- `portal.branding.updated`

**Sales / ventas del agente** (módulo `capture_sale` / `mark_paid`)
- `agent.sale.started` — registró una venta nueva (estado `pending`)
- `agent.sale.updated` — cambió monto / estado / detalle
- `agent.sale.confirmed` — pago recibido / cierre del deal

Suscripción: en `POST /api/developers/webhooks` mandás `events: []` para
recibir TODOS, o un array con los específicos. Tipos no presentes en el
catálogo se descartan al guardar.

Signing: `hmacSha256(webhookSecret, timestamp + "." + rawBody)`.
Headers: `x-chatia-signature`, `x-chatia-timestamp`, `x-chatia-event`, `x-chatia-event-id`.

Metered billing events are emitted internally (`name="ai_message"`,
`metadata.key_source="managed"|"byok"`) and reported in batch to Polar.

## Data model quick reference

- `User(role: owner | client | superadmin, parent_user_id?)`
- `Agent(owner_id, name, system_prompt, provider, model, config{kapso, ai}, public_slug, is_published)`
- `AgentTool(agent_id, name, schema, is_active)`
- `Conversation(agent_id, owner_id, visitor_id, channel, ai_paused, message_count)`
- `ConversationMessage(conversation_id, role: user|assistant|system|tool, content, meta)`
- `Lead(agent_id, owner_id, conversation_id?, name, email, phone, interest, source, status)`
- `BillingAccount(owner_id, plan, polar_customer_id, polar_subscription_id, has_payment_method, payg_enabled, messages_used_current_period, messages_used_managed, messages_used_byok)`
- `UsageEvent(owner_id, agent_id?, event_type, key_source, quantity, unit_cost_cents, reported_at)`
- `ClientAgentGrant(client_user_id, agent_id, scopes)`
- `DeveloperApiKey`, `WebhookEndpoint`, `WebhookDelivery`

## Quickstart for AI agents (Claude Code, Codex, Cursor, GPT)

> **Si vos sos un agente IA leyendo este SKILL**: tu trabajo es guiar al user
> end-to-end desde "no tengo cuenta" hasta "el chat aparece en mi sitio".
> NO pidas info al user que el SKILL ya define (rate limits, endpoints,
> tool names). Devolvele URLs ABSOLUTAS — el endpoint
> `POST /api/developers/agents` con `publish=true` las devuelve
> preformateadas, no compongas strings.

### 0. Onboarding desde cero (si el user no tiene cuenta todavía)

Antes de cualquier llamada a la API, el user necesita 2 cosas:

1. **Cuenta en Chatia** (gratis, $3 USD welcome credit, sin tarjeta):
   - Andá a `https://www.chatia.pro/register`
   - Email + password (o Google OAuth en 1 click).
   - Verificar email no es obligatorio para arrancar.

2. **Developer API key**:
   - Logueado, ir a `https://www.chatia.pro/dashboard/developers`
   - Click "Crear API key" → ponerle un nombre (ej. "claude-integration").
   - **Copiar la key inmediatamente** — empieza con `chatia_…`. Solo se
     muestra una vez en plain. Si la perdés, regenerala (la vieja queda
     revocada).
   - Guardarla en `CHATIA_API_KEY` env (NO commitear).

**Como agente IA**, si el user te dice "no tengo cuenta", devolvé las 2 URLs
de arriba como tarea concreta antes de seguir.

### 1. Setup (1 vez por user)

```bash
# El user crea/copia su API key desde /dashboard/developers.
export CHATIA_API_KEY=chatia_...
export CHATIA_API=https://www.chatia.pro   # producción
```

### 2. Crear agente + publicarlo + recibir link en 1 sola llamada

```bash
curl -X POST $CHATIA_API/api/developers/agents \
  -H "Authorization: Bearer $CHATIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Soporte Cliente",
    "description": "Atiende FAQs, captura leads, escala a humano.",
    "system_prompt": "Sos un agente de soporte. Respondé en tono cálido y profesional. Si te piden algo que no podés resolver, capturá el lead y derivá.",
    "first_message": "¡Hola! ¿En qué puedo ayudarte hoy?",
    "language": "es",
    "tools": ["capture_lead", "send_whatsapp_handoff", "send_nps_survey"],
    "publish": true
  }'
```

Response (todos los campos están listos para usar — copy-paste directo):

```json
{
  "id": 42,
  "name": "Soporte Cliente",
  "is_published": true,
  "public_slug": "soporte-cliente-42",
  "chat_url": "https://www.chatia.pro/chat/soporte-cliente-42",
  "dashboard_url": "https://www.chatia.pro/dashboard/agents/42",
  "widget_snippet": "<script src=\"https://www.chatia.pro/widget.js?slug=soporte-cliente-42\" defer></script>",
  "wordpress_plugin_url": "https://www.chatia.pro/wordpress",
  "tools": ["capture_lead", "send_whatsapp_handoff", "send_nps_survey"]
}
```

**Como agente IA**: devolvele al user los 4 campos:
- `chat_url` — para que pruebe el agente live (link clickeable).
- `dashboard_url` — para que lo edite (cambiar prompt, branding, etc.).
- `widget_snippet` — para pegar en su sitio HTML.
- `wordpress_plugin_url` — si su sitio es WordPress, link al plugin.

### 3. Conectar canales (opcional)

El agente recién creado solo tiene webchat público. Para sumarle WhatsApp, Instagram, Gmail, Google Calendar, Slack y +25 apps más:

| Canal | Cómo |
| --- | --- |
| WhatsApp | El user pega su Kapso API key en `/dashboard/agents/{id}#channels`. |
| Facebook + IG + Messenger | Click "Conectar Facebook" en `/dashboard/agents/{id}#integrations` (1 OAuth cubre los 3). |
| Telegram | Pegar bot_token de @BotFather en el card Telegram del panel. |
| Gmail / Calendar / Sheets / Slack / Notion / etc | OAuth desde el card correspondiente. Composio maneja todo. |

Lista completa: `https://www.chatia.pro/integrations`.

### 4. Suscribirse a eventos del agente

```bash
curl -X POST $CHATIA_API/api/developers/webhooks \
  -H "Authorization: Bearer $CHATIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Mi CRM",
    "url": "https://miempresa.com/chatia-webhook",
    "events": ["agent.lead.captured", "agent.sale.confirmed"]
  }'
```

Si pasás `events: []` recibís TODOS los 27 eventos del catálogo. Para suscribirte a tipos específicos, los nombres canónicos están en `GET /api/developers/events/catalog`.

### 5. Refinar el agente conversando

Una vez creado, el user puede iterar via dashboard, pero un agente IA externo también puede llamar al **builder conversacional** vía `POST /api/builder/stream` (SSE). El builder usa las tools de iteración (`update_agent_meta`, `set_agent_tools`, `update_agent_branding`) sin recrear — preserva history y leads.

Mensajes naturales que el builder mapea a tools concretas:

| User dice | Builder hace |
| --- | --- |
| "hacelo más cálido" | `update_agent_meta(system_prompt=...)` |
| "agregale email" | `attach_builtin_tools(["send_email"])` |
| "que sea verde / modo claro" | `update_agent_branding(primary_color="#10b981")` |
| "sumá conocimiento de…" | `add_knowledge_chunk(content=...)` |
| "qué tiene puesto?" | `get_agent_summary` |

### 6. Embed manual (fallback sin la response del POST)

Si por alguna razón no tenés `widget_snippet` listo:

```html
<!-- Footer de cualquier sitio HTML -->
<script src="https://www.chatia.pro/widget.js?slug=<SLUG>" defer></script>
```

Para WordPress: descargar plugin oficial desde `https://www.chatia.pro/wordpress` (~7 KB), subirlo en *Plugins → Agregar nuevo → Subir*, activar, pegar el slug en *Ajustes → Chatia*. Sin tocar código del tema.

Para Shopify: pegar el snippet en `theme.liquid` antes de `</body>`.

Para Wix: Settings → Custom Code → Body end.

### 7. Recetas en Python y Node.js (mismas llamadas, otros lenguajes)

#### Python (httpx)

```python
import httpx, os

CHATIA_API = os.environ.get("CHATIA_API", "https://www.chatia.pro")
CHATIA_API_KEY = os.environ["CHATIA_API_KEY"]

def create_agent(name: str, system_prompt: str, tools: list[str], publish: bool = True) -> dict:
    """Crea un agente en Chatia y devuelve TODOS los links listos para usar.

    Returns: dict con `chat_url`, `dashboard_url`, `widget_snippet`,
    `wordpress_plugin_url`, `id`, `public_slug`. Cero strings que componer.
    """
    resp = httpx.post(
        f"{CHATIA_API}/api/developers/agents",
        headers={"Authorization": f"Bearer {CHATIA_API_KEY}"},
        json={
            "name": name,
            "system_prompt": system_prompt,
            "tools": tools,
            "publish": publish,
        },
        timeout=30.0,
    )
    resp.raise_for_status()
    return resp.json()

agent = create_agent(
    name="Asistente Médico",
    system_prompt="Sos un asistente médico. Capturá síntomas, derivá al doctor si son urgentes.",
    tools=["capture_lead", "create_calendar_event", "send_email"],
)

print(f"Tu agente está vivo en: {agent['chat_url']}")
print(f"Pegá esto en tu sitio:\n{agent['widget_snippet']}")
```

#### Node.js (fetch nativo, Node 18+)

```javascript
const CHATIA_API = process.env.CHATIA_API || "https://www.chatia.pro";
const CHATIA_API_KEY = process.env.CHATIA_API_KEY;

async function createAgent({ name, systemPrompt, tools, publish = true }) {
  const resp = await fetch(`${CHATIA_API}/api/developers/agents`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${CHATIA_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      name,
      system_prompt: systemPrompt,
      tools,
      publish,
    }),
  });
  if (!resp.ok) {
    throw new Error(`Chatia API ${resp.status}: ${await resp.text()}`);
  }
  return await resp.json();
}

const agent = await createAgent({
  name: "Asistente Médico",
  systemPrompt: "Sos un asistente médico. Capturá síntomas, derivá al doctor si son urgentes.",
  tools: ["capture_lead", "create_calendar_event", "send_email"],
});

console.log("Chat público:", agent.chat_url);
console.log("Dashboard:   ", agent.dashboard_url);
console.log("Widget HTML: ", agent.widget_snippet);
```

#### Verificar la firma de un webhook entrante (Node.js)

```javascript
import { createHmac, timingSafeEqual } from "node:crypto";

function verifyChatiaWebhook(rawBody, signature, timestamp, secret) {
  // signature header viene como "sha256=<hex>" o "<hex>" según versión.
  const sig = signature.startsWith("sha256=") ? signature.slice(7) : signature;
  const expected = createHmac("sha256", secret)
    .update(`${timestamp}.${rawBody}`)
    .digest("hex");
  return timingSafeEqual(Buffer.from(sig, "hex"), Buffer.from(expected, "hex"));
}

// En tu Express handler:
app.post("/chatia-webhook", express.raw({ type: "application/json" }), (req, res) => {
  const ok = verifyChatiaWebhook(
    req.body.toString("utf8"),
    req.header("x-chatia-signature"),
    req.header("x-chatia-timestamp"),
    process.env.CHATIA_WEBHOOK_SECRET, // copia el secret del card del webhook
  );
  if (!ok) return res.status(401).send("invalid_signature");

  const event = JSON.parse(req.body.toString("utf8"));
  console.log("event:", event.type, "agent:", event.agent_id);
  // ... actuá según event.type (agent.lead.captured, agent.sale.confirmed, etc.)
  res.status(200).end();
});
```

### 8. Catálogo de tools nativas (qué darle al agente)

El catálogo vivo es `GET /api/developers/skills`. Como referencia rápida, las tools que casi todo agente quiere:

| Tool | Cuándo darla | Trigger natural en el chat |
|---|---|---|
| `capture_lead` | siempre que necesite contactar al user después | "dejame tu mail", "quién sos" |
| `update_lead_status` | si el flow tiene pipeline (vendedor) | "calificá este lead como qualified" |
| `add_lead_note` | para historial conversacional del CRM | "anotá que prefiere zoom" |
| `create_calendar_event` | clínicas, peluquerías, consultorías | "agendá turno", "reservá hora" |
| `list_availability` | siempre con calendar — para proponer huecos libres | "qué horarios hay" |
| `cancel_appointment` | con calendar — handle no-show / cancel | "cancelá mi turno" |
| `reschedule_appointment` | con calendar | "movélo al lunes" |
| `send_email` | seguimiento post-conversación | "mandame info por mail" |
| `send_whatsapp_handoff` | derivar a humano del owner | "necesito hablar con alguien" |
| `query_knowledge` | si cargaste KB con FAQs / catálogo | "tienen X producto?" |
| `send_nps_survey` | una vez al cierre | implícito al final del flow |
| `register_candidate` | RRHH — entrevistas | "quiero postularme" |
| `score_candidate` | RRHH — evaluar competencias | implícito durante entrevista |
| `flag_for_review` | RRHH — cierre de entrevista | implícito al cierre |
| `http_request` | cuando ninguna nativa cubre y no hay Composio toolkit | API custom del owner |

**Mapping vertical → set default**:

| Vertical | Tools recomendadas |
|---|---|
| Ventas / E-commerce | `capture_lead, send_whatsapp_handoff, update_lead_status, add_lead_note` |
| Soporte / FAQ | `query_knowledge, send_whatsapp_handoff, send_nps_survey` |
| Turnos / Agenda (clínicas, peluquerías) | `create_calendar_event, list_availability, cancel_appointment, reschedule_appointment, capture_lead` |
| RRHH / Reclutamiento | `register_candidate, score_candidate, flag_for_review, send_email` |
| Mixto | combinar las de arriba según el caso |

### 9. Errores comunes y cómo manejarlos

| HTTP | Detail | Causa | Qué hacer |
|---|---|---|---|
| 401 | Missing API key | Header `Authorization` ausente o vacío | Pedirle al user que cree key en `/dashboard/developers` |
| 401 | Invalid API key | Key revocada, regenerada o tipeada mal | Que la regenere desde el dashboard y la reemplace |
| 402 | insufficient_balance | Welcome credit gastado, sin PAYG activo | El user debe agregar tarjeta en `/billing` |
| 403 | agents_limit_reached | Plan free + 1 agente ya creado | Borrar agente viejo o subir de plan |
| 404 | Agent not found | `agent_id` incorrecto o de otro user | Listar con `GET /api/developers/agents` y verificar |
| 422 | Validation error | Body mal armado (ej. tools que no existen) | Verificar con `GET /api/developers/skills` los nombres válidos |
| 429 | Too many requests | Rate limit excedido | Leer `Retry-After` header y esperar. Implementar backoff exponencial |
| 5xx | Internal error | Bug del lado nuestro | Reintentar con backoff. Si persiste 5+ min, reportar a soporte |

**Patrón de retry recomendado** (Python):

```python
import httpx, time

def call_with_retry(method, url, **kwargs):
    for attempt in range(5):
        resp = httpx.request(method, url, **kwargs)
        if resp.status_code == 429:
            retry_after = int(resp.headers.get("retry-after", 5))
            time.sleep(retry_after)
            continue
        if resp.status_code >= 500:
            time.sleep(2 ** attempt)  # 1, 2, 4, 8, 16s
            continue
        resp.raise_for_status()
        return resp.json()
    raise RuntimeError("max retries")
```

### 10. Lista de tareas para un agente IA externo (checklist)

Cuando el user te pide "creame un agente Chatia para X", seguí este orden:

- [ ] Confirmar que tiene `CHATIA_API_KEY`. Si no, redirigir a `/dashboard/developers`.
- [ ] Inferir el VERTICAL del caso de uso (ventas / soporte / turnos / RRHH / mixto).
- [ ] Elegir tools del mapping de arriba según vertical.
- [ ] Redactar `system_prompt` natural en español, 2-4 oraciones, indicando rol + tono + cuándo capturar lead.
- [ ] Redactar `first_message` saludo de bienvenida natural.
- [ ] Llamar `POST /api/developers/agents` con `publish: true`.
- [ ] Devolverle al user los 4 campos de la response: `chat_url`, `dashboard_url`, `widget_snippet`, `wordpress_plugin_url` (si aplica).
- [ ] Si el user mencionó canales adicionales (WhatsApp, IG, etc.), explicarle que se conectan desde el dashboard `/dashboard/agents/{id}#integrations` con OAuth — no desde la API.
- [ ] Si el user mencionó un sitio WordPress / Shopify / Wix, mostrar el snippet adecuado del paso 6.
- [ ] Si el user quiere recibir eventos (CRM, notificaciones), explicarle el setup de webhooks del paso 4.
- [ ] (Opcional) Sugerir cargar Knowledge base con `POST /api/v1/agents/{id}/knowledge/import-url` o `/upload-file` si hay docs / FAQs.

## Pricing model (relevant for flows that trigger checkouts)

- **Free** — 50 msgs trial, 1 agente, sin tarjeta.
- **Pay-as-you-go** — activa cuando el user guarda tarjeta. Cobra metered.
- **Starter $5** · 2 agentes · 1k msgs.
- **Pro $12** · 4 agentes · 5k msgs.
- **Studio $29** · 6 agentes · 20k msgs.
- **Agency $59** · 15 agentes · 60k msgs.
- **Overage**: $0.010/msg (API nuestra) · $0.004/msg (BYOK).
- **Extra seat**: $5/mes cualquier plan pago.

## Safety & conventions

- Every destructive call needs explicit confirmation. `DELETE /api/agents/{id}`
  requires the user to type the exact agent name.
- Webchat never emits the `{ "reply": "..." }` before the message is persisted.
- The builder prompt forbids "Paso 1", "Paso 2". Use task names.
- `free_messages_remaining` is **per user**, not per agent.
- Do not redirect payments. Use `PolarEmbedCheckout.create(url, { theme: "dark" })`
  inline.

## When integrating with another AI agent platform

Read this file first. Then:

1. Ask for a `CHATIA_API_KEY` scoped to a single user.
2. Use only the endpoints listed above — no private routes.
3. Surface Chatia's webchat or widget as a resource the agent can embed or trigger.
4. Respect quotas: `/api/billing/account` tells you how close the user is to
   their agent/message limit.
