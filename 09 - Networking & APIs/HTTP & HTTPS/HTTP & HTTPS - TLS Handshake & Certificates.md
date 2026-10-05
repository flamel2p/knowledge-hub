---
title: HTTP & HTTPS - TLS Handshake & Certificates
aliases: [TLS 1.3 handshake, X.509, Certificate chain, ACME, Let's Encrypt, mTLS, SNI, ECH, Certificate Transparency]
type: deep-dive
domain: networking
tags: [domain/networking, type/deep-dive, topic/tls, topic/certificates]
status: draft
created: 2026-10-05
updated: 2026-10-05
version_checked: "TLS 1.3 · CA/B SC-081v3 (200-day certs from 2026-03) — 2026-10"
parent: "[[HTTP & HTTPS]]"
related: ["[[Traefik]]", "[[Cloudflare]]", "[[Coolify]]", "[[OAuth 2.0 & OIDC]]"]
---

# HTTP & HTTPS - TLS Handshake & Certificates

> [!info] Deep dive of [[HTTP & HTTPS]]

> [!abstract] TL;DR
> TLS 1.3 gives HTTPS three properties: **confidentiality** (AEAD encryption with keys from an ephemeral ECDHE exchange), **integrity**, and **server authentication** (an X.509 certificate chain signed by a CA the client trusts, proven by a signature over the handshake). The handshake takes **1 round trip**, or 0-RTT on resumption with replay risk. Certificates come from ACME CAs (Let's Encrypt, ZeroSSL, Google Trust Services) and are now capped at **200 days (since 2026-03-15) → 100 (2027) → 47 (2029)**. Remember: **automate issuance and renewal, monitor expiry independently, and terminate TLS where you control both the cert and the next hop.**

## Concept
| Component | Role |
|---|---|
| **ClientHello / ServerHello** | Negotiate version, cipher suite, key-share group (X25519, X25519MLKEM768), extensions (SNI, ALPN) |
| **ECDHE key share** | Ephemeral Diffie–Hellman gives **forward secrecy**: past traffic stays safe even if the cert key leaks later |
| **Certificate** (X.509) | Binds a public key to names (SAN: `app.example.com`, `*.example.com`). Signed by an intermediate CA |
| **Chain** | Leaf → intermediate(s) → root (in the client trust store). The server must send the intermediates |
| **CertificateVerify** | Server signs the handshake transcript with its private key (proves possession) |
| **Finished** | MACs over the transcript. Detects tampering |
| **SNI** | Hostname in ClientHello so one IP can serve many certificates |
| **ALPN** | Selects `h2` / `http/1.1` (HTTP/3 uses QUIC with ALPN `h3`) |
| **ECH** | Encrypts the inner ClientHello (hides SNI) using keys published in DNS HTTPS records |
| **Certificate Transparency (CT)** | Public append-only logs. Browsers require SCTs. Anyone can monitor issuance for their domains |

- **TLS 1.3 cipher suites** (only 5): `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256`, and two CCM variants. RSA key exchange, CBC, RC4, SHA-1 and static DH are all removed.
- **Key types**: ECDSA P-256 (smaller, faster) or RSA-2048/3072 (compatibility). Many servers serve both.

## How It Works

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server
  C->>S: ClientHello (versions, suites, key_share X25519MLKEM768, SNI, ALPN h2)
  S->>C: ServerHello (chosen suite, key_share) — handshake keys derived (HKDF)
  S->>C: {EncryptedExtensions, Certificate chain, CertificateVerify, Finished}
  C->>C: verify chain → trusted root, SAN matches SNI, validity, SCTs; verify signature
  C->>S: {Finished} + application data (1-RTT total)
  S->>C: NewSessionTicket (for resumption / 0-RTT next time)
```

- **Key schedule**: HKDF derives separate handshake and application traffic keys from the ECDHE shared secret. Keys can be updated mid-connection (KeyUpdate).
- **Resumption**: a PSK from a session ticket allows an abbreviated handshake. **0-RTT early data** can be **replayed** by an attacker, so only allow it for idempotent GETs.
- **Post-quantum hybrid**: `X25519MLKEM768` combines classical X25519 with ML-KEM-768 (FIPS 203). Chrome, Firefox and Cloudflare have negotiated it by default since 2024–25. The larger ClientHello (~1.1 KB key share) breaks some old middleboxes.
- **Revocation**: OCSP/CRL proved unreliable (soft-fail), and Let's Encrypt shut down its OCSP service in 2025. The industry answer is **short-lived certificates** plus CRLs for browsers' aggregated revocation lists.
- **mTLS**: the server also requests a client certificate (CertificateRequest), the basis of zero-trust service auth and Cloudflare Authenticated Origin Pulls.

## Practical Usage

### ACME issuance & renewal
| Challenge | How | Use when |
|---|---|---|
| HTTP-01 | Serve token at `/.well-known/acme-challenge/` on port 80 | Public hosts, single names |
| DNS-01 | Create `_acme-challenge` TXT record via DNS API | **Wildcards**, hosts not reachable on 80, behind Cloudflare proxy |
| TLS-ALPN-01 | Special cert on 443 with ALPN `acme-tls/1` | Port 80 blocked |

```yaml
# Traefik v3 static config — DNS-01 via Cloudflare (scoped API token: Zone.DNS:Edit)
certificatesResolvers:
  le:
    acme:
      email: ops@example.com
      storage: /letsencrypt/acme.json      # chmod 600, persistent volume
      dnsChallenge:
        provider: cloudflare
        resolvers: ["1.1.1.1:53", "8.8.8.8:53"]
```

### Inspect & debug
```bash
openssl s_client -connect app.example.com:443 -servername app.example.com -showcerts </dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
curl -vI https://app.example.com 2>&1 | grep -E "SSL connection|ALPN|expire|issuer"
# Check chain/protocol/cipher config from outside: SSL Labs, or testssl.sh
./testssl.sh --fast app.example.com
```

### Expiry monitoring (independent of the renewer)
```bash
days=$(( ( $(date -d "$(echo | openssl s_client -connect $H:443 -servername $H 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2)" +%s) - $(date +%s) ) / 86400 ))
[ "$days" -lt 14 ] && notify "Cert for $H expires in $days days"
```
- Or use Uptime Kuma / Better Stack cert checks. Subscribe to CT monitoring (crt.sh, Cloudflare CT alerts) for unexpected issuance.

### Cloudflare in front of origin
| Mode | Browser → CF | CF → Origin | Verdict |
|---|---|---|---|
| Off / Flexible | HTTPS | **HTTP** | ❌ Plaintext to origin, redirect loops |
| Full | HTTPS | HTTPS, cert **not validated** | ⚠️ MITM-able |
| **Full (strict)** | HTTPS | HTTPS, valid cert (public CA or Cloudflare Origin CA) | ✅ Default choice |
| + Authenticated Origin Pulls (mTLS) | — | Origin only accepts Cloudflare's client cert | ✅✅ Blocks direct-to-origin bypass |

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| ACME automation + independent expiry alerts | Every domain | Manual 1-year certs + calendar reminders (impossible at 47 days) |
| DNS-01 wildcards for many subdomains (previews) | PaaS/Coolify previews | Hitting Let's Encrypt rate limits with per-preview HTTP-01 certs |
| TLS 1.2+1.3 only, modern suites | Public endpoints | Keeping TLS 1.0/1.1 "for old clients" |
| Full (strict) + origin firewall to CF IPs / mTLS | Behind Cloudflare | Flexible SSL. Origin open to the world |
| HSTS after HTTPS is solid everywhere | Production | HSTS preload before every subdomain supports HTTPS |
| Private keys only on the terminating host, 0600 | Always | Keys in git, in images, in shared folders |

## Performance & Trade-offs
- TLS 1.3 adds 1 RTT on new connections (vs 2 for TLS 1.2). HTTP/3 merges transport and TLS handshakes. Keep-alive and connection reuse amortize the cost.
- ECDSA certs make smaller handshakes and faster signing than RSA. Serve dual certs if you must support very old clients.
- PQ hybrid key exchange adds ~1 KB to ClientHello and negligible CPU.
- Short-lived certs mean more ACME traffic and more renewal failure points, which is why renewal monitoring matters.

## Tips & Reminders
> [!tip]
> - Always serve the **full chain** (leaf + intermediates). Mobile/Java clients fail with incomplete chains while browsers may silently fetch the intermediates.
> - Use CAA DNS records (`0 issue "letsencrypt.org"`) to restrict which CAs may issue for your domains.
> - Let's Encrypt stopped sending expiry emails in 2025, so you need your own monitoring.
> - **In ZP's stack**: [[Traefik]] (via [[Coolify]]) handles ACME automatically. Use DNS-01 with a scoped Cloudflare token for wildcards. Back up `acme.json`. Set Cloudflare SSL to **Full (strict)** on every client zone, and restrict origin 80/443 to Cloudflare IP ranges (or use Cloudflare Tunnel, which needs no inbound ports).

## Version Notes
| Change | When | Impact |
|---|---|---|
| TLS 1.3 (RFC 8446) | 2018-08 | 1-RTT, mandatory forward secrecy, removed legacy crypto |
| TLS 1.0/1.1 deprecated (RFC 8996) | 2021-03 | Disable them |
| ML-KEM hybrid key exchange default in browsers/CDNs | 2024–25 | PQ protection against "harvest now, decrypt later" |
| Let's Encrypt ends OCSP, offers 6-day certs, stops expiry emails | 2025 | Rely on short lifetimes + own monitoring |
| CA/B Forum SC-081v3: 200-day max | 2026-03-15 | Annual renewals no longer possible |
| → 100-day max / 47-day max | 2027-03 / 2029-03 | Full automation mandatory. Shorter domain-validation reuse |

## Critical Issues & Gotchas
> [!danger] Expired certificates = outage
> Recurring incidents at large companies (Microsoft Teams 2020, Starlink 2023, countless SaaS) came from forgotten renewals. Expired intermediates and root transitions also break old clients (Let's Encrypt's DST Root CA X3 expiry in Sep 2021 broke older Android/OpenSSL 1.0.x clients). Automate, monitor and test the chain from old clients you support.

> [!danger] Private key exposure & mis-issuance
> Leaked keys (in repos, images or backups) allow impersonation until the cert expires or is revoked, and revocation checking is weak. CA mis-issuance has happened (DigiNotar 2011, Symantec distrust 2017–18). Use CAA records + CT monitoring, and keep certs short-lived.

> [!warning] Gotchas
> - SNI missing in clients (old Java, some IoT, `openssl s_client` without `-servername`) → wrong/default cert served.
> - Clock skew on servers or devices → "certificate not yet valid".
> - TLS-terminating proxies must forward `X-Forwarded-Proto`, or apps generate `http://` redirect loops.
> - Corporate TLS inspection (MITM proxies) breaks certificate pinning and mTLS.
> - Wildcards cover one level only: `*.example.com` ≠ `a.b.example.com`.

## Related
- [[HTTP & HTTPS]]
- [[Traefik]] — ACME resolvers, TLS termination
- [[Cloudflare]] — edge TLS, Origin CA, Authenticated Origin Pulls
- [[Coolify]] — certificate automation for deployed apps
- [[OAuth 2.0 & OIDC]] — mTLS / sender-constrained tokens

## References
- RFC 8446 (TLS 1.3): https://www.rfc-editor.org/rfc/rfc8446
- Let's Encrypt docs (challenge types, rate limits): https://letsencrypt.org/docs/
- CA/B Forum 47-day schedule (DigiCert): https://www.digicert.com/blog/tls-certificate-lifetimes-will-officially-reduce-to-47-days
- Cloudflare SSL/TLS encryption modes: https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/
- testssl.sh: https://testssl.sh/
