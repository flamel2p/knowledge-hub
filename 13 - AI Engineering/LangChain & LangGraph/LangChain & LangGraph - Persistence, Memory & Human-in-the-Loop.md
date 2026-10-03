---
title: LangChain & LangGraph - Persistence, Memory & Human-in-the-Loop
aliases: [LangGraph checkpointer, LangGraph memory, LangGraph interrupt, PostgresSaver, LangGraph Store, HITL]
type: deep-dive
domain: ai-engineering
tags: [domain/ai-engineering, type/deep-dive, topic/langgraph, topic/memory, lang/python]
status: draft
created: 2026-10-02
updated: 2026-10-02
version_checked: "langgraph 1.2.12 · langgraph-checkpoint 3.x — 2026-10"
parent: "[[LangChain & LangGraph]]"
related: ["[[LangChain & LangGraph - Graph API, State & Control Flow]]", "[[LangChain & LangGraph - Agents, Tools & Middleware]]", "[[PostgreSQL]]", "[[Supabase]]", "[[Redis]]"]
---

# LangChain & LangGraph - Persistence, Memory & Human-in-the-Loop

> [!info] Deep dive of [[LangChain & LangGraph]]

> [!abstract] TL;DR
> A **checkpointer** snapshots graph state after each super-step, keyed by `thread_id`. That one mechanism gives you conversation memory, crash recovery, time travel and **`interrupt()`**-based human-in-the-loop. A separate **Store** holds cross-thread, long-term memory (user facts, preferences) in namespaced key-value form with optional vector search. Remember: **checkpointer = per-thread, Store = per-user/global; the interrupted node re-runs from the top on resume.**

## Concept
| | Checkpointer | Store |
|---|---|---|
| Scope | One `thread_id` (a conversation / job) | Any namespace, e.g. `("users", user_id, "memories")` |
| Holds | Full graph state history (`StateSnapshot`s) | Arbitrary JSON docs with keys |
| Written | Automatically, per super-step | Explicitly by your nodes/tools (`store.put`) |
| Read | Automatically on `invoke` with the same `thread_id` | Explicitly (`store.get` / `store.search`) |
| Backends | `InMemorySaver`, `SqliteSaver`, `PostgresSaver`, Redis, MongoDB | `InMemoryStore`, `PostgresStore`, Redis |
| Purpose | Short-term memory, HITL, fault tolerance, time travel | Semantic/episodic/procedural long-term memory |

- **Thread**: `config={"configurable": {"thread_id": "..."}}`. Same id = continue. New id = fresh state.
- **`StateSnapshot`** fields: `values`, `next` (nodes pending), `config` (includes `checkpoint_id`), `metadata` (`source`, `step`, `writes`), `created_at`, `parent_config`, `tasks` (with pending interrupts).
- **Pending writes**: if one parallel node fails, the successful siblings' writes are stored, so a resume doesn't redo them.

## How It Works

```mermaid
sequenceDiagram
  participant C as Client
  participant G as Graph
  participant DB as Checkpointer
  C->>G: invoke(input, thread_id=T)
  G->>DB: checkpoint step 0..n
  G->>G: node "approve" calls interrupt(payload)
  G->>DB: save state + pending interrupt
  G-->>C: result["__interrupt__"] = [Interrupt(value=payload, id=…)]
  Note over C: hours later, any process
  C->>G: invoke(Command(resume=decision), thread_id=T)
  G->>DB: load latest checkpoint
  G->>G: re-run "approve" from its first line; interrupt() returns decision
  G->>DB: continue checkpointing → END
```

- **Durability modes** (`graph.invoke(..., durability=...)`):
  - `"exit"` persists only when the run ends or interrupts. Fastest, but a crash loses the progress in between.
  - `"async"` (default) writes while the next step runs. There's a small window where a crash loses the latest step.
  - `"sync"` writes before the next step starts. Safest, slowest.
- **Resume semantics**: on `Command(resume=...)`, the node containing `interrupt()` starts **again from its first line**. Each `interrupt()` call in that node is matched **by order** to resume values. Code before the interrupt runs twice.
- **Time travel**: `get_state_history(config)` lists checkpoints. `invoke(None, {"configurable": {"thread_id": T, "checkpoint_id": X}})` replays from X. `update_state(config, values, as_node="n")` forks a new branch from there.

## Practical Usage

### Postgres checkpointer + store (production)
```python
from psycopg_pool import AsyncConnectionPool
from psycopg.rows import dict_row
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.store.postgres.aio import AsyncPostgresStore

pool = AsyncConnectionPool(DB_URI, max_size=20, open=False,
                           kwargs={"autocommit": True, "row_factory": dict_row, "prepare_threshold": None})
await pool.open()
checkpointer = AsyncPostgresSaver(pool)
store = AsyncPostgresStore(pool, index={"embed": "openai:text-embedding-3-small", "dims": 1536, "fields": ["text"]})
await checkpointer.setup(); await store.setup()        # run once / in deploy migration step

graph = builder.compile(checkpointer=checkpointer, store=store)
```
- `autocommit=True` and `row_factory=dict_row` are **required**. Without them you get silent non-persistence or `TypeError: tuple indices`.
- `prepare_threshold=None` makes it safe behind PgBouncer/Supavisor transaction pooling.
- Tables created: `checkpoints`, `checkpoint_blobs`, `checkpoint_writes`, `checkpoint_migrations` (+ `store`, `store_vectors` with pgvector).

### Long-term memory via Store
```python
from langgraph.runtime import Runtime

async def remember(state: State, runtime: Runtime[Ctx]):
    ns = ("users", runtime.context.user_id, "memories")
    hits = await runtime.store.asearch(ns, query=state["messages"][-1].content, limit=5)
    facts = "\n".join(h.value["text"] for h in hits)
    ...
    await runtime.store.aput(ns, str(uuid4()), {"text": "Prefers replies in Malay"})
```
- Memory types: **semantic** (facts about the user), **episodic** (past successful interactions as few-shot examples), **procedural** (self-updated system prompt).
- Writing memories in the hot path adds latency. A background "memory extraction" node/job after the conversation is usually better (see the LangMem SDK).

### Managing short-term history
```python
from langchain_core.messages import RemoveMessage
from langchain_core.messages.utils import trim_messages, count_tokens_approximately

def trim(state):
    keep = trim_messages(state["messages"], strategy="last", max_tokens=6000,
                         token_counter=count_tokens_approximately, start_on="human", include_system=True)
    drop = {m.id for m in state["messages"]} - {m.id for m in keep}
    return {"messages": [RemoveMessage(id=i) for i in drop]}
```
- With `create_agent`, use `SummarizationMiddleware` instead of hand-rolling this.

### `interrupt()`: approval with edit
```python
from langgraph.types import interrupt, Command

def approve_refund(state: State):
    decision = interrupt({"action": "refund", "order": state["order_id"], "amount": state["amount"]})
    if decision["type"] == "reject":
        return Command(goto="explain_rejection")
    amount = decision.get("amount", state["amount"])      # human may edit
    return {"approved_amount": amount}

out = graph.invoke({"messages": [...]}, cfg)
if "__interrupt__" in out:
    payload = out["__interrupt__"][0].value               # send to UI / WhatsApp admin
graph.invoke(Command(resume={"type": "approve", "amount": 50}), cfg)
```

### HITL in `create_agent`
```python
HumanInTheLoopMiddleware(interrupt_on={
    "send_whatsapp": {"allowed_decisions": ["approve", "edit", "reject"]},
    "search_orders": False,
})
# resume: Command(resume={"decisions": [{"type": "approve"}]})
```

### Encrypted checkpoints
```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
serde = EncryptedSerializer.from_pycryptodome_aes()       # reads LANGGRAPH_AES_KEY
checkpointer = AsyncPostgresSaver(pool, serde=serde)
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| `thread_id` = conversation/job id, Store namespace = user id | Chat products | Reusing one `thread_id` per user forever (unbounded history + checkpoints) |
| Interrupt as the **first** statement of a dedicated node | Approvals | Side effects (API call, DB insert) before `interrupt()` in the same node |
| Idempotency key on side-effecting nodes | Payments, messaging | Assuming each node runs exactly once |
| Background memory extraction | Long-term memory | Writing to Store on every turn in the hot path |
| Encrypted serializer + DB-level access control | PII in state | Plain checkpoints holding phone numbers/ICs readable by any DB role |
| Retention job per thread age | Prod Postgres | Letting `checkpoint_blobs` grow to 100s of GB |

## Performance & Trade-offs
- Each super-step writes ≥ 1 row in `checkpoints` + changed blobs. A 10-turn agent chat with tools easily produces 50–100 checkpoints per thread.
- Blob size ≈ size of changed channels. Message lists dominate, which is another reason to trim or summarize.
- `durability="sync"` adds one DB round-trip of latency per step. `"async"` hides it.
- Store vector search uses pgvector. Index `fields` deliberately, because embedding every value costs tokens and storage.
- `InMemorySaver` is per-process. Behind multiple Coolify replicas, threads "forget" randomly.

## Tips & Reminders
> [!tip]
> - Inspect a stuck thread with `graph.get_state(cfg).next` and `.tasks[0].interrupts`.
> - Interrupts can be resumed by **any** process that shares the checkpointer, so web-hook-driven approval (e.g. a WhatsApp admin button → n8n → `Command(resume=...)`) works.
> - Don't wrap `interrupt()` in a bare `try/except Exception`. It raises `GraphInterrupt` internally, and swallowing it breaks the pause.
> - **In ZP's stack**: put checkpointer and store in a dedicated `langgraph` schema on [[Supabase]]/[[PostgreSQL]], with RLS-free access from a service role only. Connect via the session pooler (5432) or a direct connection. Back up with the normal PG backups.

> [!question] Self-check
> - A node calls an API, then `interrupt()`. How many times does the API get called for one approval? (Twice.)
> - Which durability mode can lose the last step on a crash?
> - Checkpointer or Store for "user prefers Malay across all chats"?

## Version Notes
| Version | Change |
|---|---|
| 0.2 (2024-08) | Checkpointers moved to `langgraph-checkpoint-*` packages, new schema |
| 0.3 (2025-02) | `interrupt()` + `Command(resume=)` replace static breakpoints as the main HITL API |
| 0.4+ (2025) | Multiple interrupts per node matched by id. `durability` param replaces `checkpoint_during` |
| checkpoint 3.0 | Hardened msgpack deserialization (CVE-2026-28277). Upgrade required |
| langchain 1.0 | `HumanInTheLoopMiddleware` with approve/edit/reject decisions |

> [!warning] Unverified — check before relying on this
> The exact minor that introduced `durability` and id-matched multi-interrupts wasn't verified this run.

## Critical Issues & Gotchas
> [!danger] Checkpointer RCE chain (CVE-2025-67644 + CVE-2026-28277)
> SQL injection in the SQLite checkpointer's metadata filter + unsafe msgpack deserialization of checkpoint blobs = RCE. Fixed in `langgraph-checkpoint` ≥ 3.0 and `langgraph-checkpoint-sqlite` ≥ 3.0.1. Never expose `list(filter=...)` to user input. Treat write access to the checkpoint DB as code execution.

> [!danger] Double side effects on resume
> Code before `interrupt()` re-executes, so you can charge a card twice or send a WhatsApp message twice. Split it into separate nodes, or guard with an idempotency key stored in state.

> [!warning] `langgraph dev` ignores your checkpointer
> The local Agent Server and LangSmith Deployment inject their own persistence. A custom `checkpointer=` passed at compile time is ignored there, which confuses people testing Postgres locally (GitHub issue #5790).

> [!warning] Unbounded growth
> Nothing prunes by default. Add a scheduled job: delete threads older than N days from `checkpoints`/`checkpoint_writes`/`checkpoint_blobs` (or use the Agent Server's TTL config).

## Related
- [[LangChain & LangGraph]]
- [[LangChain & LangGraph - Graph API, State & Control Flow]] — super-steps define checkpoint boundaries
- [[LangChain & LangGraph - Agents, Tools & Middleware]] — `HumanInTheLoopMiddleware`, `SummarizationMiddleware`
- [[PostgreSQL]] · [[Supabase]] · [[Redis]]

## References
- Persistence: https://docs.langchain.com/oss/python/langgraph/persistence
- Checkpointers: https://docs.langchain.com/oss/python/langgraph/checkpointers
- Interrupts: https://docs.langchain.com/oss/python/langgraph/interrupts
- Memory: https://docs.langchain.com/oss/python/langgraph/add-memory
- Durable execution: https://docs.langchain.com/oss/python/langgraph/durable-execution
- Checkpointer SQLi → RCE: https://research.checkpoint.com/2026/from-sqli-to-rce-exploiting-langgraphs-checkpointer/
- Issue #5790: https://github.com/langchain-ai/langgraph/issues/5790
