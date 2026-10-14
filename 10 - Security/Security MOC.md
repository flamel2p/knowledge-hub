---
title: "Security MOC"
aliases: ["Security"]
type: moc
domain: security
tags: [domain/security, type/moc]
status: draft
created: 2026-09-28
updated: 2026-10-14
parent: "[[00 - Home]]"
---

# Security MOC

> [!abstract] Scope
> Network and application security, authentication and authorization.

## Notes
- [[JWT]] — token format, verification rules, algorithm-confusion CVEs
- [[OAuth 2.0 & OIDC]] — delegated authz + identity layer, PKCE, OAuth 2.1, token-theft incidents
- [[Network Security]] — firewalls, segmentation, zero trust, TLS (200-day certs), DDoS; Docker-bypasses-UFW, xz, regreSSHion
  - [[Network Security - Common Web Attacks]]
- [[Authentication & Authorization]] — passkeys/MFA, sessions vs tokens, RBAC/ABAC/ReBAC, RLS, NIST 800-63-4; session theft + IDOR risks
  - [[Authentication & Authorization - Interview Questions]] — interview prep reference (finance-app auth, BNM rules)
- [[OWASP Top 10]] — 2025 edition: A01 access control (+SSRF), A03 supply chain, A10 exceptional conditions; ASVS + sibling lists

## Planned
_None — all planned overviews are written; queued deep dives are listed under Notes as (planned)._

## Cross-domain Links
- [[00 - Home]]
