---
title: n8n - MyInvois e-Invoicing Integration
aliases: [MyInvois, LHDN e-invoice, HASiL e-invoicing, Malaysia e-invoice API, UBL 2.1 Malaysia, consolidated e-invoice, self-billed e-invoice, e-invoice intermediary, MyInvois API n8n, e-invoice QR code, TIN validation]
type: deep-dive
domain: automation
tags: [domain/automation, type/deep-dive, topic/n8n, topic/e-invoicing, topic/malaysia-compliance]
status: draft
created: 2026-11-18
updated: 2026-11-28
version_checked: "MyInvois API v1.0 (documents v1.1 signed) · Phase 4 (RM1–5m) from 2026-01-01 · relaxation to 2027-12-31 (reported) · exemption threshold RM1m, reportedly RM3m from 2026-09-01 (unconfirmed) — 2026-11"
parent: "[[n8n]]"
related: ["[[Next.js - Payments with Stripe & Malaysian Gateways]]", "[[n8n - Error Handling & Workflow Patterns]]", "[[n8n - Security, Credentials & Self-Hosting Hardening]]", "[[REST API - Webhooks & Idempotency]]", "[[PostgreSQL - Audit Logging & Change History]]", "[[n8n - Managing Multiple Client Instances]]"]
---

# n8n - MyInvois e-Invoicing Integration

> [!info] Deep dive of [[n8n]]

> [!abstract] TL;DR
> Malaysia's **MyInvois** (LHDN/HASiL) requires in-scope businesses to submit e-invoices (UBL 2.1 JSON/XML) for **validation** before or at the time of the transaction, and to share the **validated QR link** with buyers.
> - **Integration paths**: the free MyInvois Portal (manual, low volume), accounting software with built-in MyInvois, or **direct API** (ERP/POS/your SaaS), optionally via an **intermediary** acting for many taxpayers. n8n fits as the orchestration layer: source system → map → sign → submit → poll → store UUID/QR → notify.
> - **Status (Nov 2026, verify)**: Phase 4 (turnover RM1–5m) started 2026-01-01 with a **relaxation period reported to run to 2027-12-31**. The exemption threshold is RM1m, with secondary sources claiming **RM3m from 2026-09-01**. Confirm each client's obligation on hasil.gov.my before selling a build.
> - **Engineering essentials**: idempotent submission per invoice, validation errors surfaced to staff, 72-hour cancellation window, document signing (v1.1), and an audit trail of every submission.

## Concept
| Concept | Meaning | Implementation note |
|---|---|---|
| e-Invoice types | Invoice, credit note, debit note, refund note (+ self-billed variants) | Map from source document type |
| Validation | MyInvois returns UUID + long ID when valid | Store both. Generate QR/link from them |
| Consolidated e-invoice | B2C sales without buyer request, aggregated monthly | Submit within 7 days after month end (check guideline) |
| Self-billed | Buyer issues e-invoice (foreign suppliers, agents, etc.) | Separate flow and rules |
| General public TIN | Placeholder TIN for consumers/foreign buyers | Use the TIN codes from the guideline |
| Intermediary | Your system submits on behalf of a taxpayer | Taxpayer authorizes intermediary in the portal. `onbehalfof` header |
| Digital signature | Document v1.1 requires signing with the taxpayer's certificate | Certificate from an approved CA. Signing step before submit |
| Cancellation | Issuer can cancel within 72 h of validation | After that: credit/debit note |
| Rejection | Buyer may request rejection within 72 h | Track status changes |

**Integration options**:

| Option | Effort | Pick it when… |
|---|---|---|
| MyInvois Portal (manual) | None | < ~50 invoices/month, owner does it |
| Accounting software with MyInvois (SQL, AutoCount, Xero via partners, etc.) | Config | Client already uses it for invoicing |
| n8n + direct API | Medium | Invoices originate in custom systems (POS, booking app, Supabase SaaS) |
| Dedicated e-invoice middleware/peppol provider | Subscription | High volume, want vendor-managed compliance |

## How It Works
```mermaid
sequenceDiagram
  participant SRC as Source (Supabase orders / POS)
  participant N as n8n workflow
  participant ID as MyInvois Identity (OAuth2)
  participant API as MyInvois API
  participant DB as Postgres (einvoices table)
  SRC->>N: order.paid event (outbox / webhook)
  N->>DB: insert einvoice (status=pending, idempotency key = order id)
  N->>N: map to UBL 2.1 JSON, sign (v1.1), base64 + SHA-256
  N->>ID: client_credentials token (cached ~1 h)
  N->>API: POST /api/v1.0/documentsubmissions
  API-->>N: submissionUid + accepted/rejected documents
  loop until Valid/Invalid
    N->>API: GET /api/v1.0/documentsubmissions/{uid}
  end
  N->>DB: store uuid, longId, status, errors
  N->>SRC: attach QR/validation link to receipt; alert staff on Invalid
```

## Practical Usage
**Token request** (HTTP Request node, cache in Redis/Data Table until expiry):
```http
POST https://preprod-api.myinvois.hasil.gov.my/connect/token
Content-Type: application/x-www-form-urlencoded

client_id={{CLIENT_ID}}&client_secret={{CLIENT_SECRET}}&grant_type=client_credentials&scope=InvoicingAPI
# intermediary acting for a taxpayer: add header  onbehalfof: <taxpayer TIN or ROB>
```
**State table** (one row per source document):
```sql
create table public.einvoices (
  id bigint generated always as identity primary key,
  tenant_id uuid not null,
  source_type text not null,              -- 'order', 'refund'
  source_id text not null,
  doc_type text not null,                 -- '01' invoice, '02' credit note, ...
  status text not null default 'pending', -- pending|submitted|valid|invalid|cancelled
  submission_uid text, uuid text, long_id text,
  errors jsonb, payload_sha256 text,
  created_at timestamptz default now(), validated_at timestamptz,
  unique (tenant_id, source_type, source_id, doc_type)   -- idempotency
);
```
**Submit** (after mapping + signing in a Code node):
```js
// Code node: prepare submission body
const doc = JSON.stringify($json.ublDocument);                       // signed UBL 2.1 JSON (v1.1)
const crypto = require('crypto');
return [{ json: { body: { documents: [{
  format: 'JSON',
  document: Buffer.from(doc).toString('base64'),
  documentHash: crypto.createHash('sha256').update(doc).digest('hex'),
  codeNumber: $json.invoiceNo,                                      // your invoice number
}]}}}];
```
```http
POST https://preprod-api.myinvois.hasil.gov.my/api/v1.0/documentsubmissions
Authorization: Bearer {{token}}
Content-Type: application/json
```
**Poll status, then build the share link**:
```text
Wait 3–5 s → GET /api/v1.0/documentsubmissions/{{submissionUid}}
  → per document: status Valid → save uuid + longId
       share URL pattern: https://myinvois.hasil.gov.my/{uuid}/share/{longId}  (→ QR code on receipt)
  → status Invalid → save validation errors → notify staff (Telegram) with order link
  → still InProgress → loop with backoff (max ~10 tries)
```
**Validate buyer TIN before issuing** (B2B):
```http
GET /api/v1.0/taxpayer/validate/{tin}?idType=BRN&idValue={brn}
```
**Wrong way**:
```text
❌ Submitting from n8n with no idempotency key           → duplicate e-invoices after retries
❌ Treating HTTP 202 as "validated"                     → submission accepted ≠ document valid
❌ Production credentials in sandbox testing (and vice versa) → rejected tokens, confusing errors
❌ Cancelling after 72 h                                → not allowed; issue a credit note instead
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Outbox from the source DB → n8n | Orders/payments | Submitting from the checkout request path |
| One `einvoices` row per source doc (unique key) | Always | Relying on n8n execution history |
| Separate sub-workflows: map, sign, submit, poll | Maintainability | One 60-node workflow |
| Sandbox (preprod) end-to-end tests per document type | Before go-live | Testing only standard invoices |
| Consolidated monthly B2C job + per-request individual e-invoice | Retail/F&B | Individual e-invoice for every walk-in sale |
| Staff alert queue for Invalid documents | Operations | Silent failures discovered at audit |
| Per-client credentials and certificates in n8n credentials | Multi-client | Shared intermediary credentials across clients |

## Performance & Trade-offs
- **Rate limits** apply per endpoint (submission, polling, search). Batch documents per submission (up to the documented max count and size), and back off on 429.
- **Synchronous vs async**: validation usually completes within seconds but isn't guaranteed. Never block the customer's checkout on MyInvois. Attach the QR link later (email/WhatsApp/receipt reprint).
- **Build vs buy**: direct API integration is a few days of work per document type, plus ongoing guideline changes. For clients already on accounting software with MyInvois, configure that instead.
- **Signing** requires certificate handling (PKCS#12, XAdES-like signature for JSON/XML). A small signing microservice (Node/.NET) is cleaner than complex Code nodes.

## Tips & Reminders
> [!tip] In ZP's stack
> - **Productizable service**: "MyInvois integration for custom POS/booking systems" for RM1–5m SMEs. Fixed-price setup + monthly monitoring retainer. Sell only after confirming the client is actually in scope.
> - **Liability boundary**: the taxpayer remains responsible for tax correctness (classification codes, tax types, amounts). Your SOW should cover the technical submission and monitoring, not tax advice. Have the client's accountant sign off the field mapping.
> - **Data**: e-invoices contain buyer TIN/IC/contact details. Keep `einvoices` and payload logs private, retained per tax record rules (typically 7 years), and access-controlled ([[PostgreSQL - Audit Logging & Change History]]).

- Keep classification codes, tax type codes, MSIC codes and state codes as lookup tables, refreshed from LHDN's published code lists.
- Use the preprod environment (`preprod-api.myinvois.hasil.gov.my`) with separate credentials.
- Store the exact submitted payload hash and response for audits.
- Monitor daily: count of pending/invalid documents per client, as part of the n8n alerting baseline ([[n8n - Monitoring, Metrics & Alerting]]).

## Version Notes
| Date | Change relevant here |
|---|---|
| 2024-08-01 | Phase 1: turnover > RM100m |
| 2025-01-01 / 2025-07-01 | Phases 2–3: RM25–100m, RM5–25m |
| 2025-06 | LHDN revised schedule: RM1–5m deferred to 2026-01-01. Smaller threshold changes announced |
| 2025-12 | Exemption threshold raised to RM1m (Cabinet decision reported) |
| 2026-01-01 | Phase 4 (RM1–5m) mandatory, with relaxation period (reported to 2027-12-31 in guideline v4.7–4.9) |
| 2026-09-01 (unconfirmed) | Exemption reportedly raised to RM3m |

> [!warning] Unverified — check before relying on this
> Thresholds, phase dates, relaxation end, consolidated e-invoice grace periods, API endpoint paths, rate limits and the share-URL pattern come from secondary sources and memory. One source even reports Phase 4 postponed to 2027. Verify on https://www.hasil.gov.my/e-invoice/ and https://sdk.myinvois.hasil.gov.my/ before quoting or building.

## Critical Issues & Gotchas
> [!danger] Selling compliance work on outdated rules
> MyInvois rules changed repeatedly in 2025–2026 (thresholds, phase dates, relaxation periods). Quoting a client an integration they don't legally need, or missing one they do, damages trust and may cause penalties. Confirm scope against the current HASiL timeline and the client's accountant, in writing, before starting.

> [!danger] Duplicate or wrong e-invoices are tax records
> Retries without idempotency create duplicate validated e-invoices, and wrong amounts or tax codes become official records needing credit notes and explanations. Use a unique key per source document, validate mapping in preprod with the accountant, and gate production submission on passing internal checks.

- **72-hour windows** for cancellation and buyer rejection. Build tooling for staff to act within them.
- **Clock and timezone**: issue dates/times must be UTC in the document per spec. Convert from MYT correctly.
- **Certificate expiry** breaks signing silently. Alert 30 days before expiry.
- **Intermediary authorization** can be revoked by the taxpayer at any time. Handle 401/403 with a clear alert.

## Related
- [[n8n]]
- [[n8n - Accounting Integrations with Xero, QuickBooks & SQL Accounting]] — accounting system postings
- [[Next.js - Payments with Stripe & Malaysian Gateways]] — payments that trigger e-invoices
- [[n8n - Error Handling & Workflow Patterns]] — retries and failure alerts
- [[n8n - Security, Credentials & Self-Hosting Hardening]] — credentials and certificates
- [[REST API - Webhooks & Idempotency]] — idempotent submission design
- [[n8n - Managing Multiple Client Instances]] — per-client credentials and monitoring

## References
- https://www.hasil.gov.my/en/e-invoice/
- https://sdk.myinvois.hasil.gov.my/
- https://sdk.myinvois.hasil.gov.my/einvoicingapi/
- https://lookuptax.com/docs/einvoicing/myinvois-einvoicing-malaysia
- https://www.thestar.com.my/news/nation/2025/06/05/e-invoicing-deadlines-for-businesses-with-revenue-below-rm5mil-revised-says-lhdn
