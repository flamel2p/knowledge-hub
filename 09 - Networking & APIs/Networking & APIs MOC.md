---
title: "Networking & APIs MOC"
aliases: ["Networking & APIs"]
type: moc
domain: networking
tags: [domain/networking, type/moc]
status: draft
created: 2026-09-28
updated: 2026-10-25
parent: "[[00 - Home]]"
---

# Networking & APIs MOC

> [!abstract] Scope
> Protocols and API styles: HTTP, REST, WebSocket, GraphQL, gRPC.

## Notes
- [[HTTP & HTTPS]] — semantics, HTTP/2–3, TLS 1.3, 200→47-day certs, smuggling & Rapid Reset
  - [[HTTP & HTTPS - TLS Handshake & Certificates]]
  - [[HTTP & HTTPS - HTTP-2, HTTP-3 & Connection Management]]
- [[REST API]] — resource design, OpenAPI 3.2, RFC 9457 errors, pagination, idempotency, BOLA
  - [[REST API - Webhooks & Idempotency]]
  - [[REST API - Versioning, Pagination & Error Design]]
- [[WebSocket]] — RFC 6455, auth on upgrade, heartbeats, Redis fan-out, CSWSH
  - [[WebSocket - Scaling & Reliability]]
- [[GraphQL]] — typed client-driven queries; Sept 2025 spec (OneOf), DataLoader, federation, cost-limit/IDOR risks
  - [[GraphQL - Security Hardening]]
- [[gRPC]] — protobuf + HTTP/2 RPC for service-to-service; streaming, deadlines, Buf/Connect, HTTP/2 DoS risks
  - [[gRPC - Protobuf Schema Evolution]]

## Planned
_None — all planned overviews are written; queued deep dives are listed under Notes as (planned)._

## Cross-domain Links
- [[00 - Home]]
