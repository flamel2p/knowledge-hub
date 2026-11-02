---
title: n8n - WhatsApp Cloud API Workflows & Template Messaging
aliases: [WhatsApp Cloud API n8n, WhatsApp templates, 24-hour customer service window, per-message pricing, wamid, BSUID, X-Hub-Signature-256, WhatsApp Business Cloud node, WhatsApp opt-in]
type: deep-dive
domain: automation
tags: [domain/automation, type/deep-dive, topic/n8n, topic/whatsapp, topic/messaging-api]
status: draft
created: 2026-11-02
updated: 2026-11-02
version_checked: "Meta per-message pricing (since 2025-07) · BSUID in webhooks (2026 rollout) · n8n 2.42.x — 2026-11"
parent: "[[n8n]]"
related: ["[[n8n - AI Agent Nodes & LLM Workflows]]", "[[n8n - Error Handling & Workflow Patterns]]", "[[REST API - Webhooks & Idempotency]]", "[[Cloudflare - Workers for Webhooks & Edge APIs]]", "[[Redis - Rate Limiting & Distributed Locks]]", "[[Design Patterns - Hexagonal Architecture in TypeScript & Dart]]"]
---

# n8n - WhatsApp Cloud API Workflows & Template Messaging

> [!info] Deep dive of [[n8n]]

> [!abstract] TL;DR
> - **Inbound**: Webhook node with raw body → verify `X-Hub-Signature-256` → **dedupe by `wamid`** → store → reply 200 within seconds → process asynchronously.
> - **Outbound**: free-form replies work only inside the **24-hour customer service window**. Outside it you must send an **approved template**, billed **per message** by category (marketing > utility ≈ authentication). Utility templates inside an open window are free.
> - Track `last_inbound_at`, opt-ins, template status and delivery statuses in Postgres. Pace sends per number.
> - **Identity is changing**: with WhatsApp usernames, the sender may arrive as a **BSUID** (`user_id`, e.g. `MY.1349…`) instead of a phone number. Don't key contacts only on E.164.

## Concept
| Concept | Meaning | Practical rule |
|---|---|---|
| WABA / phone number ID | Business account and sender number | One credential set per client WABA, client-owned |
| System user token | Long-lived business token for API calls | Client's Business Manager, least scopes, stored in n8n credentials |
| App secret | HMAC key for webhook signatures | Verify every POST |
| Customer service window | 24 h after the user's last inbound message | Free-form text/media allowed only inside it |
| Template | Pre-approved message (marketing / utility / authentication) | Required outside the window. Variables `{{1}}`… |
| `wamid` | Unique message ID | Idempotency key for inbound and outbound |
| Status webhooks | `sent`, `delivered`, `read`, `failed` | Update message rows. Alert on spikes of `failed` |
| Quality rating / messaging limits | Per number and portfolio, based on blocks/reports | Low quality → limits cut, number restricted |
| BSUID | Business-scoped user ID (`CC.digits`), regenerated on phone change | New `user_id` column. `wa_id`/`from` may be a BSUID |

**Pricing model (per delivered template message)**:

| Category | Typical use | Malaysia (USD, third-party figures) |
|---|---|---|
| Marketing | Promotions, re-engagement, abandoned cart | ≈ $0.086 |
| Utility | Order/booking confirmations, reminders | ≈ $0.014. **Free inside an open window** |
| Authentication | OTP codes | ≈ $0.014 |
| Service (free-form inside window) | Replies to customer messages | Free |

## How It Works
```mermaid
sequenceDiagram
  participant M as Meta Cloud API
  participant E as Edge (Cloudflare Worker, optional)
  participant N as n8n Webhook workflow
  participant DB as Postgres
  participant W as n8n worker workflow
  M->>E: POST /wa/acme (X-Hub-Signature-256)
  E->>N: forward raw body (or verify at edge)
  N->>N: HMAC verify, parse entries
  N->>DB: INSERT message ON CONFLICT (wamid) DO NOTHING; contact.last_inbound_at = now()
  N-->>M: 200 OK (< 5 s)
  N->>W: Execute Workflow / queue (new rows only)
  W->>W: route: human takeover? buttons? AI agent?
  W->>M: POST /{phone_number_id}/messages (text if window open, else template)
  M-->>N: status webhook (delivered/read/failed) → update row
```

## Practical Usage
**Webhook verification (GET)**: return `hub.challenge` when `hub.verify_token` matches:
```js
// Code node behind a GET Webhook
const q = $input.first().json.query;
if (q['hub.mode'] === 'subscribe' && q['hub.verify_token'] === $env.WA_VERIFY_TOKEN) {
  return [{ json: { challenge: q['hub.challenge'] } }];   // Respond to Webhook: text = {{ $json.challenge }}
}
throw new Error('verify token mismatch');
```
**Signature check (POST)**: Webhook node with **Raw Body** on; requires `NODE_FUNCTION_ALLOW_BUILTIN=crypto`:
```js
const crypto = require('crypto');
const item = $input.first();
const raw = Buffer.from(item.binary.data.data, 'base64');            // raw body as received
const sig = (item.json.headers['x-hub-signature-256'] || '').replace('sha256=', '');
const mac = crypto.createHmac('sha256', $env.WA_APP_SECRET).update(raw).digest('hex');
if (sig.length !== mac.length || !crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(mac))) {
  throw new Error('invalid signature');
}
const body = JSON.parse(raw.toString('utf8'));
const out = [];
for (const e of body.entry ?? []) for (const c of e.changes ?? []) {
  for (const m of c.value.messages ?? []) out.push({ json: { kind: 'message', phoneNumberId: c.value.metadata.phone_number_id, m, contact: c.value.contacts?.[0] } });
  for (const s of c.value.statuses ?? []) out.push({ json: { kind: 'status', s } });
}
return out;
```
n8n 2.x blocks `$env` in Code nodes by default (`N8N_BLOCK_ENV_ACCESS_IN_NODE`). Either allow it deliberately on a webhook-only instance, or verify at the edge (Worker) and pass a trusted header.

**Idempotent storage and window tracking** (Postgres node):
```sql
insert into wa_messages (wamid, tenant_id, contact_key, direction, type, body, received_at)
values ($1, $2, $3, 'in', $4, $5, to_timestamp($6))
on conflict (wamid) do nothing
returning wamid;                                   -- empty result = duplicate delivery → stop

insert into wa_contacts (tenant_id, contact_key, wa_id, bsuid, last_inbound_at)
values ($2, $3, $7, $8, now())
on conflict (tenant_id, contact_key) do update set last_inbound_at = now(), bsuid = coalesce(excluded.bsuid, wa_contacts.bsuid);
```
**Send: text inside the window, template outside it** (HTTP Request node body):
```json
{
  "messaging_product": "whatsapp",
  "to": "{{ $json.to }}",
  "type": "template",
  "template": {
    "name": "order_ready_v2",
    "language": { "code": "ms" },
    "components": [
      { "type": "body", "parameters": [
        { "type": "text", "text": "{{ $json.customerName }}" },
        { "type": "text", "text": "{{ $json.orderNo }}" }
      ] }
    ]
  }
}
```
Window check before choosing the payload: `{{ DateTime.fromISO($json.last_inbound_at) > $now.minus({ hours: 23, minutes: 50 }) }}` (10-minute safety margin).

**Wrong way**:
```text
❌ Process the AI reply before responding 200          → Meta retries; duplicate replies to the customer
❌ Free-form text 30 hours after the last message      → error 131047 (re-engagement required); message never sent
❌ Marketing blast to everyone in the CRM              → blocks/reports → quality drops → number restricted
❌ contacts.phone as the only key                      → BSUID-only senders create orphan/duplicate contacts
❌ Using EvolutionAPI / unofficial WhatsApp Web bridges for client production → ToS violation, ban risk, no support
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Ingest workflow (verify, dedupe, store, 200) separate from processing workflows | All bots | One long workflow doing everything before responding |
| `wamid` unique constraint | Inbound + outbound | Trusting Meta to deliver exactly once |
| Window-aware sender sub-workflow (text vs template) | Every outbound message | Each workflow deciding on its own |
| Templates versioned (`order_ready_v2`) per language (`ms`, `en`, `zh_CN`) | Template management | Editing a live template's meaning in place |
| Opt-in records + STOP keyword handling | Marketing | Messaging purchased or scraped lists |
| Per-number pacing via BullMQ/Redis limiter | Bulk utility/marketing sends | Looping HTTP calls as fast as n8n runs |
| Channel adapter (internal `InboundMessage`) | Multi-channel products | Meta's payload shape spread through every workflow |

## Performance & Trade-offs
- **Throughput**: a Cloud API number defaults to tens of messages per second (≈ 80 mps per number reported, upgradable). The real limit for small businesses is **messaging limits** (unique users per 24 h for business-initiated messages) and quality, not mps.
- **Per-pair limits**: rapid messages to the same user trigger pair rate limits (error 131056). Batch content into fewer messages.
- **Cost control**: use utility templates triggered by customer actions (often free inside the window), and reserve marketing templates for opted-in segments. Track cost per tenant from status webhooks with `pricing.category`.
- **n8n load**: status webhooks often outnumber messages 3:1 (sent/delivered/read). Handle them in a lightweight branch, or filter statuses at the edge (Worker) to save executions ([[Cloudflare - Workers for Webhooks & Edge APIs]]).

## Tips & Reminders
> [!tip] In ZP's stack
> - **Ownership**: the WABA, Business Manager, phone number and app belong to the **client**. You get a system user with scoped permissions, as in the SOW. If you own them, a dispute or ban takes the client's channel hostage and you carry the liability.
> - **Stack**: Cloudflare Worker (verify + queue) or an n8n Webhook on a dedicated webhook-only instance → Postgres (Supabase) → n8n queue-mode workers → AI agent / Chatwoot handoff ([[n8n - AI Agent Nodes & LLM Workflows]], [[n8n - Queue Mode & Scaling]]).
> - **Billing your clients**: pass Meta message costs through at cost plus margin, and show the per-category counts in the monthly report.

- The built-in **WhatsApp Trigger** node registers the app's webhook subscription itself, so only one active trigger per Meta app works reliably. For multi-client setups, use a plain Webhook node and your own routing per `phone_number_id`.
- Pin the Graph API version in one credential/env variable (`WA_GRAPH_VERSION`), and upgrade on Meta's deprecation schedule.
- Download inbound media promptly (media URLs expire) and store it in private storage ([[Supabase - Storage, Uploads & Image Transformations]]).
- Map Meta error codes to actions: `131047` → send a template, `130429` → back off, `131026` → mark undeliverable, `131056` → slow down per recipient.

## Version Notes
| Date | Change |
|---|---|
| 2023 | Cloud API becomes the default (On-Premises API sunset 2025) |
| 2024-11 | Service conversations free for all businesses |
| 2025-07-01 | Switch from conversation-based to **per-message** pricing for templates. Utility templates free inside the service window |
| 2026 (from Mar/Jun) | BSUIDs in webhooks (`user_id`), then usernames rolling out. `wa_id`/`from` may no longer be a phone number |
| 2026 | Additional local-currency rate cards (published Mar, effective Apr 2026) |

> [!warning] Unverified — check before relying on this
> Malaysia rates come from BSP sites (SleekFlow, ChatMaxima, Go4whatsup) that disagree on effective dates. BSUID/username rollout dates differ between sources. The current Graph API version wasn't checked. Verify on https://developers.facebook.com/docs/whatsapp/pricing and Meta's changelog before quoting clients.

## Critical Issues & Gotchas
> [!danger] Platform risk: one Meta decision can stop the business
> Quality drops, policy violations (prohibited goods, aggressive marketing) or Business Manager verification issues can restrict or disable a number overnight, and appeals are slow. For SME clients whose orders run through WhatsApp, document this in the SOW, keep a fallback channel (SMS, email, web chat, Telegram), and keep the contact list and consent exportable.

> [!danger] Unverified webhooks and duplicate processing
> Without HMAC verification, anyone who learns the webhook URL can inject fake orders or prompts into the AI agent. Without `wamid` dedupe, Meta's retries produce double replies and duplicate orders. Verify the signature, dedupe, and respond fast ([[REST API - Webhooks & Idempotency]]).

- **PDPA + Meta policy**: marketing requires opt-in. Record when and how consent was given, honour opt-outs immediately, and state retention for chat logs.
- **Template rejection/pausing**: templates with misleading content or low engagement get rejected or paused. Keep fallbacks (a second approved variant).
- **Token leaks**: a system user token in an n8n export or screenshot gives full send access. Rotate it, and use n8n credentials, not hardcoded headers ([[n8n - Security, Credentials & Self-Hosting Hardening]]).
- **Time zones**: schedule reminders in `Asia/Kuala_Lumpur`. A UTC cron sends "good morning" at 8 pm.

## Related
- [[n8n]]
- [[n8n - AI Agent Nodes & LLM Workflows]] — the agent behind free-text messages
- [[n8n - Error Handling & Workflow Patterns]] — retries, error workflows, DLQs
- [[REST API - Webhooks & Idempotency]] — signatures and idempotency keys
- [[Cloudflare - Workers for Webhooks & Edge APIs]] — edge ingest and status filtering
- [[Redis - Rate Limiting & Distributed Locks]] — per-number and per-tenant pacing

## References
- https://developers.facebook.com/docs/whatsapp/cloud-api
- https://developers.facebook.com/docs/whatsapp/pricing
- https://developers.facebook.com/docs/whatsapp/cloud-api/support/error-codes
- https://ycloud.com/blog/whatsapp-usernames-and-business-scoped-user-ids
- https://sleekflow.io/blog/malaysia-whatsapp-business-api-case-study
- https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.whatsapp/
