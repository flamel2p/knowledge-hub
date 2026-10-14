---
title: Design Patterns - Interview Questions
aliases: [OOP interview, design pattern interview, SOLID interview, 设计模式面试题]
type: reference
domain: cs-fundamentals
tags: [domain/cs-fundamentals, type/reference, topic/design-patterns, topic/oop, topic/interview, lang/java, lang/typescript]
status: draft
created: 2026-10-13
updated: 2026-10-14
version_checked: "n/a (concepts) — Java 21+ examples — 2026-10"
parent: "[[Design Patterns]]"
related: ["[[System Design - Interview Questions]]", "[[Spring Boot - Interview Questions]]", "[[Backend Interview Questions]]"]
---

# Design Patterns - Interview Questions

> [!info] Reference for [[Design Patterns]] (overview planned) · index: [[Backend Interview Questions]]

> [!abstract] TL;DR
> Q1: **OOP** — the four pillars with real examples (`Money` invariants, `PaymentMethod` sealed interface), composition over inheritance, SOLID. Q2: **design patterns in your recent projects** — Adapter, Strategy, queue-based load leveling, idempotent consumer, Repository, circuit breaker, told as *problem → pattern → result* on the WhatsApp AI-agent pipeline, with a `[!question]` box for your project numbers.

## Q1. Explain the concept of object-oriented programming (OOP).

**Grading**: can you explain the **why** (managing change and complexity) with a real example, not "Dog extends Animal"? Do you know where inheritance hurts?

**One-liner**: OOP models a system as objects that bundle **state with the behaviour that protects it**, and that collaborate through **interfaces**, so each part can change without breaking the others.

| Pillar | What it really means | Real example |
|---|---|---|
| **Encapsulation** | Hide state and **enforce invariants** behind methods. Not "private fields + getters/setters" | `Money` can't be negative or mix currencies. `Order.cancel()` refuses if already shipped |
| **Abstraction** | Expose *what*, hide *how* | `PaymentGateway.charge()` hides HTTP, signing, retries |
| **Inheritance** | Reuse plus an "is-a" subtype relationship | `abstract class BaseRepository<T>`. Use sparingly |
| **Polymorphism** | One call, behaviour chosen at runtime by the concrete type (dynamic dispatch) | `gateway.charge()` runs Stripe, FPX or a test fake |

```java
public record Money(long minor, Currency currency) {
    public Money {
        if (minor < 0) throw new IllegalArgumentException("negative amount");
        Objects.requireNonNull(currency);
    }

    public Money plus(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException("currency mismatch");
        return new Money(Math.addExact(minor, other.minor), currency);
    }
}

public sealed interface PaymentMethod permits Card, Fpx, EWallet {}
public record Card(String token) implements PaymentMethod {}
public record Fpx(String bankCode) implements PaymentMethod {}
public record EWallet(String provider) implements PaymentMethod {}

String describe(PaymentMethod m) {
    return switch (m) {
        case Card c -> "Card " + c.token();
        case Fpx f -> "FPX via " + f.bankCode();
        case EWallet w -> w.provider();
    };
}
```

- **Composition over inheritance**: deep hierarchies are fragile (a base-class change breaks every subclass). Prefer small interfaces plus composed collaborators (Strategy, Decorator).
- **SOLID** in one breath: Single responsibility, Open/closed (add a new gateway without editing checkout), Liskov (subtypes must honour the contract), Interface segregation (small interfaces), Dependency inversion (depend on `PaymentGateway`, not `StripeClient`; Spring DI does the wiring, see [[Spring Boot - Interview Questions]]).
- **Modern nuance**: languages mix paradigms. Java 21 records + sealed interfaces + pattern matching (above) give data-oriented modelling. [[TypeScript]] and [[Dart]] favour interfaces and composition. Pure data pipelines are often clearer as functions than as objects.
- **Traps**: Animal/Dog examples with no real system; "encapsulation = getters/setters" (a setter for every field breaks encapsulation); claiming inheritance is the main way to reuse code.
- Deep dive: [[Design Patterns]] · [[Java]]

## Q2. Explain the design patterns you used in your recent projects.

**Grading**: can you tie a pattern to a *problem it solved*, not recite GoF names? Do you know the cost?

**Answer frame**: one system, 3–4 patterns, each with *problem → pattern → result*. Use the WhatsApp AI-agent pipeline, which exercises the most patterns:

| Problem | Pattern | Implementation |
|---|---|---|
| Several inbound channels (WhatsApp Cloud API, web chat, Telegram) with different payloads | **Adapter** | Each channel adapter maps to one internal `InboundMessage`. The agent core never sees Meta's schema |
| Different LLMs per tenant/task (cost vs quality), provider outages | **Strategy** (+ Factory to pick it) | `LlmProvider` interface. Router picks a model by task type and tenant plan, with fallback order on 429/5xx |
| Meta retries webhooks that don't get a quick 200. LLM calls take 2–20 s | **Queue-based load leveling** (Producer–Consumer) | Webhook verifies the signature, enqueues, returns 200 in < 100 ms. Workers ([[n8n - Queue Mode & Scaling\|n8n queue mode]] / BullMQ on [[Redis]]) consume |
| Retries deliver the same message twice | **Idempotent Consumer** | `SET dedupe:{wamid} 1 NX EX 86400`. Skip if the key already exists |
| Data access mixed into business logic, hard to test | **Repository** | `ConversationRepo` over [[Supabase]]/[[PostgreSQL]]. Tests use an in-memory fake |
| Provider outage cascades into piled-up workers | **Retry with backoff + jitter, Circuit Breaker** | Exponential backoff with full jitter. Open the circuit after N failures and fall back to the next provider or a canned reply |

```typescript
interface LlmProvider {
  readonly id: string;
  complete(req: CompletionRequest, signal: AbortSignal): Promise<CompletionResult>;
}

class LlmRouter {
  constructor(private readonly providers: Map<string, LlmProvider>, private readonly breaker: CircuitBreaker) {}

  async complete(req: CompletionRequest, order: string[]): Promise<CompletionResult> {
    for (const id of order) {
      if (this.breaker.isOpen(id)) continue;
      try {
        return await this.breaker.run(id, () => this.providers.get(id)!.complete(req, AbortSignal.timeout(30_000)));
      } catch (err) {
        if (!isRetryable(err)) throw err;
      }
    }
    throw new AllProvidersFailedError(order);
  }
}

export async function handleWebhook(req: Request): Promise<Response> {
  if (!verifyMetaSignature(req)) return new Response(null, { status: 401 });
  for (const msg of parseWhatsAppPayload(await req.json())) {
    const fresh = await redis.set(`dedupe:${msg.id}`, '1', 'EX', 86_400, 'NX');
    if (fresh) await queue.add('inbound', msg, { attempts: 5, backoff: { type: 'exponential', delay: 2_000 } });
  }
  return new Response(null, { status: 200 });
}
```

- **Close with a trade-off**: "The queue added eventual consistency and a Redis dependency. I made Redis non-evicting with AOF and kept Postgres as the source of truth" (see [[Redis - Interview Questions]]).
- **Trap**: listing Singleton/Factory/Observer with no problem attached, or overusing patterns ("I used Abstract Factory for two classes"). Saying *where you chose not to* add a pattern scores well.
- Deep dive: [[Design Patterns]] · [[System Design]] · [[AI Agents - Tool Calling & Agent Loops]]

> [!question] Fill in before the interview
> Project name/client type, message volume (msgs/day), p95 webhook latency, and one number the pattern moved (e.g. duplicate replies → 0, LLM cost −X% via routing).

## Related
- [[Design Patterns]] · [[System Design]] — planned overviews
- [[System Design - Interview Questions]] — patterns applied to payments and ticketing
- [[Spring Boot - Interview Questions]] — DI as dependency inversion
- [[AI Agents - Tool Calling & Agent Loops]] — the agent loop behind Q2's example
- [[Backend Interview Questions]] — interview index

## References
- Refactoring.Guru, design pattern catalogue: https://refactoring.guru/design-patterns
- JEP 409, sealed classes: https://openjdk.org/jeps/409
- JEP 441, pattern matching for switch: https://openjdk.org/jeps/441
