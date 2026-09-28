# Knowledge Hub — Vault Rules

Personal Obsidian vault of ZP (full-stack AI engineer, Malaysia). A scheduled Claude loop grows it **one task per run** from `task.md`. These rules are binding for every run and every manual edit.

## 1. Vault Layout

```
00 - Home.md                      ← vault entry point, links every domain MOC
CLAUDE.md                         ← this file (rules)
task.md                           ← task queue (the loop's only state)
_templates/                       ← Obsidian templates = canonical note structure
_attachments/                     ← images only (png/jpg/svg/webp)
NN - <Domain>/
  <Domain> MOC.md                 ← Map of Content for the domain
  <Topic>/
    <Topic>.md                    ← overview note (type: overview)
    <Topic> - <Subtopic>.md       ← deep-dive notes (type: deep-dive)
```

Domains (fixed — do not add/rename without ZP's approval):

| Folder | MOC | Scope |
|---|---|---|
| `01 - Languages` | `Languages MOC` | JavaScript, TypeScript, Python, Go, Rust, Java, Kotlin, Swift, Dart, PHP |
| `02 - Frontend` | `Frontend MOC` | React, Next.js, Astro, WebGL, Tailwind, browser fundamentals |
| `03 - Mobile` | `Mobile MOC` | Flutter, React Native, native iOS/Android concerns |
| `04 - Backend` | `Backend MOC` | Node.js, Spring Boot, Supabase, testing, backend frameworks |
| `05 - Databases` | `Databases MOC` | PostgreSQL, MySQL, Redis, NoSQL, vector DBs |
| `06 - Messaging & Streaming` | `Messaging & Streaming MOC` | Kafka, queues, pub/sub |
| `07 - DevOps & Infrastructure` | `DevOps & Infrastructure MOC` | Docker, Kubernetes, Coolify, Traefik, Nginx, Cloudflare, Linux, Git, CI-CD |
| `08 - Automation` | `Automation MOC` | n8n, Windmill, workflow engines |
| `09 - Networking & APIs` | `Networking & APIs MOC` | HTTP & HTTPS, REST, WebSocket, GraphQL, gRPC |
| `10 - Security` | `Security MOC` | network security, auth, OWASP |
| `11 - CS Fundamentals` | `CS Fundamentals MOC` | Big O, data structures, algorithms, system design, patterns |
| `12 - Web3` | `Web3 MOC` | blockchain, Ethereum/EVM, Solidity, wallets |
| `13 - AI Engineering` | `AI Engineering MOC` | LLMs, prompting, RAG, agents, MCP, evals, fine-tuning |

## 2. File Naming

- Overview file name = canonical topic name, exactly as in `task.md`: `React.md`, `Next.js.md`, `HTTP & HTTPS.md`.
- Deep dive: `<Parent Topic> - <Subtopic>.md` → `React - Hooks.md`. Keeps every file name vault-unique so `[[wikilinks]]` resolve by name.
- Forbidden characters in names: `/ \ : # ^ [ ] | ? * " < >`. Use `CI-CD`, not `CI/CD`.
- Never create files at the vault root. Never create non-`.md` files except images in `_attachments/`.

## 3. Frontmatter (required on every note)

```yaml
---
title: React
aliases: [ReactJS, React.js]          # other names people search for
type: overview                        # overview | deep-dive | moc | reference
domain: frontend                      # folder slug: languages, frontend, mobile, backend, databases, messaging, devops, automation, networking, security, cs-fundamentals, web3, ai-engineering
tags: [domain/frontend, type/overview, topic/react]
status: draft                         # stub | draft | complete | needs-review
created: 2026-09-28
updated: 2026-09-28
version_checked: "19.2 — 2026-09"     # latest stable + month verified; "n/a" for concepts
parent: "[[Frontend MOC]]"            # deep dives: "[[React]]"
related: ["[[Next.js]]", "[[JavaScript]]", "[[React Native]]"]
---
```

Tags are hierarchical and lowercase-kebab: `domain/*`, `type/*`, `topic/*`, optional `lang/*`. No inline `#tags` in body text.

## 4. Note Structure

Use `_templates/Topic Overview.md` or `_templates/Deep Dive.md` **exactly** — same headings, same order. Delete a section only if truly N/A (e.g. no "Project Structure" for Big O) and write `N/A — <reason>` instead of leaving it empty.

Overview sections (in order): TL;DR callout → Introduction → Core Concepts → Architecture / How It Works → Project Structure → Use Cases → Pros & Cons → Alternatives & Peers → Tips & Reminders → Versions & Breaking Changes → Critical Issues & Gotchas → Deep Dives → Related → References.

`[[Redis]]` (`05 - Databases/Redis/Redis.md`) is the **reference exemplar** — read it only if unsure about depth/tone.

## 5. Writing Style

- Audience: senior developer. No beginner hand-holding, no filler, no marketing tone. Dense bullets and tables over prose.
- Every claim must be concrete: numbers, versions, commands, config keys, code. Code blocks always have a language tag.
- **Pros & Cons**: table, honest. **Alternatives & Peers**: comparison table with a "Pick it when…" column; link peers with `[[ ]]`.
- **Versions & Breaking Changes**: table `Version | Released | Key changes | Breaking / migration notes`. Verify the latest stable via web search. If you cannot verify, add `> [!warning] Unverified — check before relying on this`.
- **Critical Issues**: real incidents, CVEs, license changes, data-loss footguns, platform/vendor risks. Use `> [!danger]` callouts.
- Callouts: `[!abstract]` TL;DR, `[!tip]`, `[!warning]`, `[!danger]`, `[!example]`, `[!info]`, `[!question]`.
- Diagrams: Mermaid (` ```mermaid `) — max one or two per note, only if they clarify architecture or flow.
- Context: ZP's stack is Next.js, Flutter, Supabase/PostgreSQL, Redis, n8n (self-hosted, queue mode), Docker, Coolify, Traefik, Cloudflare, Hostinger KVM. Where relevant, add a short "In ZP's stack" tip. Malaysian regulatory notes (PDPA) only when genuinely relevant.
- Language: English. Technical terms untranslated.

## 6. Linking Rules (Obsidian graph)

- Link the **first** mention of any topic that has (or is planned in `task.md` to have) a note: `[[Kubernetes]]`, `[[React - Hooks|hooks]]`. Planned-but-unwritten links are fine — they show as future nodes.
- Do not over-link: first mention per section at most.
- Every new note must be linked **from** its domain MOC, and overview notes must list their deep dives under `## Deep Dives`.
- Deep dives link back to the parent overview in frontmatter `parent` and in the first line.
- Cross-domain links are encouraged (e.g. [[Redis]] ↔ [[n8n]] queue mode, [[PostgreSQL]] ↔ [[Supabase]]).
- Never put `[[wikilinks]]` inside `task.md` (keeps the graph clean).

## 7. Size Budget (Claude Pro — keep runs cheap)

| Item | Limit |
|---|---|
| Tasks per run | **exactly 1** |
| Overview note | 180–350 lines |
| Deep dive note | 100–250 lines |
| Web searches per run | ≤ 3 (versions, breaking changes, recent critical issues only) |
| Files modified per run | ≤ 5 (the note, its MOC, `task.md`, ≤ 2 related notes for backlinks) |
| Files read per run | Only what the task needs: this file, `task.md`, the template, the target MOC, the parent overview (for deep dives) |

Do not refactor, reformat or "improve" unrelated notes during a run.

## 8. Automation Loop Protocol (scheduled runs)

1. **Sync**: `git fetch origin main && git checkout -B main origin/main`. Work only on `main`. ZP has explicitly authorized direct pushes to `main` for this vault.
2. **Pick**: in `task.md`, the first line matching `- [ ] T` in the Queue (top to bottom). Skip lines containing `⚠️`. Do nothing else.
3. **Execute** by task type:
   - `overview` / `deep-dive` → write the note at the given path from the template. If the file already exists, improve it in place rather than replacing good content.
   - `moc` → create or refresh the named MOC.
   - `maint` → do only the maintenance described (link audit, MOC sync, stale `version_checked` refresh on ≤ 3 notes). Fix, don't report.
   - `replenish` → append the next 12 tasks to the Queue (see §9). No notes written.
4. **Link**: add the new note to its domain MOC (move the entry from "Planned" to "Notes" if listed there); add a backlink in ≤ 2 closely related existing notes' `## Related` section.
5. **Tick**: change `- [ ]` → `- [x]` and append ` | ✅ YYYY-MM-DD` (Asia/Kuching date). Update `Last run:` at the top of `task.md`.
6. **Commit & push**: stage only the files you changed, then
   `git commit -m "kb(T###): <type> <Topic>"` and `git push origin main`.
   If the push is rejected: `git pull --rebase origin main`, then push again (max 3 attempts).
7. **Stop.** One task per run, even if time remains.

**If blocked** (e.g. unverifiable topic, repeated push failure): do not tick; append ` | ⚠️ <reason, 1 line>` to the task line, commit only `task.md`, push, stop. ZP reviews ⚠️ lines manually.

**Never**: create reports, summaries, logs, plans or any file outside the vault layout; edit `CLAUDE.md` or `.gitignore`; touch `.obsidian/`; rename/move existing notes; force-push.

## 9. Backlog Replenishment Rules

When the Queue has no unchecked tasks except a `replenish` task (or the replenish task is reached):
- Add 12 tasks, continuing the `T###` numbering.
- Priority: (1) deep dives for existing overviews that have none yet, in the order ZP's stack cares about (see §5 Context); (2) missing overviews for peers already linked from notes but not yet written (unresolved links); (3) one `maint` task per 12.
- Deep-dive subtopics must be substantial enough for 100+ lines; don't split trivially.
- End the new batch with another `replenish` task so the loop never dries up.
