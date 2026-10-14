---
title: "AI Engineering MOC"
aliases: ["AI Engineering"]
type: moc
domain: ai-engineering
tags: [domain/ai-engineering, type/moc]
status: draft
created: 2026-09-28
updated: 2026-10-14
parent: "[[00 - Home]]"
---

# AI Engineering MOC

> [!abstract] Scope
> LLMs, prompting, RAG, agents, MCP, evals, fine-tuning.

## Notes
- [[Prompt Engineering]] — context engineering, structure, examples, structured outputs, evals
  - [[Build with AI]] — 11-part prompt playbook series for business tasks
- [[LLM Fundamentals]] — tokens, context, sampling, reasoning, caching, model landscape
  - [[LLM Fundamentals - Tokens, Context & Sampling]]
- [[RAG]] — hybrid retrieval, reranking, contextual retrieval, agentic RAG, tenant isolation
  - [[RAG - Chunking & Retrieval Strategies]]
- [[AI Agents]] — workflows vs agents, tool loops, harness, lethal trifecta
  - [[AI Agents - Tool Calling & Agent Loops]]
- [[Model Context Protocol]] — MCP 2026-07-28 stateless spec, OAuth 2.1, server security
- [[LangChain & LangGraph]] — agent framework + stateful graph runtime (1.x)
  - [[LangChain & LangGraph - Graph API, State & Control Flow]]
  - [[LangChain & LangGraph - Agents, Tools & Middleware]]
  - [[LangChain & LangGraph - Persistence, Memory & Human-in-the-Loop]]
  - [[LangChain & LangGraph - Multi-Agent Systems & LangSmith]]
- [[AI Evals & Observability]] — graders (code, LLM-judge, human), RAG/agent metrics, tracing, Langfuse (ClickHouse 2026), PII in traces
- [[Fine-tuning LLMs]] — when (not) to fine-tune, LoRA/QLoRA SFT, DPO, GRPO/RFT, open-weight serving; safety erosion + deprecation risks
- [[LLM APIs & SDKs]] — Claude Messages API vs OpenAI vs Gemini: tools, structured outputs, caching, batches; key hygiene + deprecation risk

## Planned
_None — all planned overviews are written; queued deep dives are listed under Notes as (planned)._

## Cross-domain Links
- [[00 - Home]]
