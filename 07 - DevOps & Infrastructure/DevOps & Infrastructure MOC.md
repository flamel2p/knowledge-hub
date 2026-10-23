---
title: "DevOps & Infrastructure MOC"
aliases: ["DevOps & Infrastructure"]
type: moc
domain: devops
tags: [domain/devops, type/moc]
status: draft
created: 2026-09-28
updated: 2026-10-23
parent: "[[00 - Home]]"
---

# DevOps & Infrastructure MOC

> [!abstract] Scope
> Containers, orchestration, PaaS, reverse proxies, edge, CI-CD, OS.

## Notes
- [[Docker]] — images, containers, Compose, Engine 29, runc escapes, UFW bypass
  - [[Docker - Dockerfile & Image Best Practices]]
  - [[Docker - Networking & Volumes]]
  - [[Docker - Compose Patterns for Production]]
- [[Coolify]] — self-hosted PaaS over SSH + Traefik; v4 stable, Jan 2026 critical CVEs
  - [[Coolify - Operations, Backups & Hardening]]
- [[Kubernetes]] — declarative orchestrator; Gateway API over retired Ingress NGINX, cgroup v2 only, k3s for small VPS
  - [[Kubernetes - Core Objects]]
- [[Traefik]] — label-driven reverse proxy + ACME; Coolify default, v3.7, 2026 ForwardAuth-bypass CVEs, Docker socket risk
  - [[Traefik - Middlewares, TLS & Routing]]
- [[Cloudflare]] — DNS/CDN/WAF edge, Tunnel + Access, R2 zero-egress, Workers; 2025 global outages = concentration risk
  - [[Cloudflare - Zero Trust Tunnel & Access]]
  - [[Cloudflare - R2 Storage & Presigned Uploads]]
- [[Git]] — content-addressed DAG VCS; merge vs rebase, secret hygiene, clone-time CVEs, Git 3.0 (main, SHA-256)
  - [[Git - Branching, Releases & Recovery]]
- [[Linux Essentials]] — VPS operations: users/perms, systemd/journald, ss/nftables, disk/OOM triage, LTS distros, kernel CVEs
  - [[Linux Essentials - VPS Hardening Runbook]]
- [[CI-CD]] — pipelines, deploy strategies, DORA; GitHub Actions → GHCR → Coolify; SHA-pinning after tj-actions compromise
  - [[CI-CD - GitHub Actions Security Hardening]]
- [[Nginx]] — event-driven web server/proxy/cache; stable 1.30, 2026 memory-safety CVEs, SPA/PHP-FPM container use
  - [[Nginx - Caching & Performance Tuning]]
  - [[Nginx - Security Hardening]]

## Planned
_None — all planned overviews are written; queued deep dives are listed under Notes as (planned)._

## Cross-domain Links
- [[00 - Home]]
