---
title: AI Agents - Voice Agents & Phone Automation
aliases: [voice AI agent, AI phone agent, AI receptionist phone, speech-to-speech, realtime voice API, STT LLM TTS pipeline, LiveKit Agents, Pipecat, Vapi, Retell, Twilio voice, SIP trunk, barge-in, VAD, call recording consent]
type: deep-dive
domain: ai-engineering
tags: [domain/ai-engineering, type/deep-dive, topic/ai-agents, topic/voice, topic/telephony]
status: draft
created: 2026-11-25
updated: 2026-11-25
version_checked: "Realtime speech-to-speech APIs + cascaded STT→LLM→TTS · LiveKit Agents / Pipecat (open source) · Vapi / Retell (managed) · Twilio/Telnyx SIP — 2026-11"
parent: "[[AI Agents]]"
related: ["[[AI Agents - Customer Support Agents for WhatsApp & Messaging]]", "[[AI Agents - Tool Calling & Agent Loops]]", "[[AI Agents - Human-in-the-Loop & Approval Workflows]]", "[[WebSocket - SSE & Streaming LLM Responses]]", "[[AI Evals & Observability - Cost Tracking & Budgets per Tenant]]", "[[LLM Fundamentals - Model Selection & Routing]]"]
---

# AI Agents - Voice Agents & Phone Automation

> [!info] Deep dive of [[AI Agents]]

> [!abstract] TL;DR
> Voice agents answer or place phone calls for bookings, FAQs, reminders and lead qualification. Two architectures:
> 1. **Cascaded**: STT → LLM (with tools) → TTS. Flexible model choice, easier debugging, more latency.
> 2. **Speech-to-speech realtime models**: lower latency, natural turn-taking, less control/transparency.
>
> Telephony comes from a **SIP trunk/CPaaS** (Twilio, Telnyx, local providers) bridged to an agent framework (**LiveKit Agents, Pipecat**) or a managed platform (**Vapi, Retell**).
> - **Engineering reality**: end-to-end latency must stay **under ~1 s** per turn, **barge-in** (interrupting the bot) must work, tools must be fast, and **handoff to a human** must be one step away.
> - **Malaysian context**: mixed BM/English/Chinese, Manglish and accents challenge STT. Test with real callers.
> - Phone automation needs **recording/consent notices** (PDPA), clear disclosure that it's an AI, and careful cost control (per-minute telephony + STT + LLM + TTS).

## Concept
| Layer | Options | Concerns |
|---|---|---|
| Telephony | Twilio, Telnyx, Vonage, local SIP providers | Malaysian numbers, call quality, per-minute cost |
| Media transport | SIP → WebRTC (LiveKit), WebSocket media streams | Jitter, codecs (8 kHz PSTN audio) |
| VAD / turn detection | Silero VAD, model-based end-of-turn | Cutting users off vs awkward silences |
| STT | Streaming ASR models | Code-switching (BM/EN/ZH), noise, names/numbers |
| LLM | Fast model + tools | Latency, tool timeouts, grounding |
| TTS | Streaming neural voices | Natural BM/EN pronunciation, latency to first audio |
| Speech-to-speech | Realtime voice models | Lower latency. Less control over transcripts/tools |
| Orchestration | LiveKit Agents, Pipecat (self-host) / Vapi, Retell (managed) | Control vs speed-to-market |

**Build vs buy**:

| Approach | Time to launch | Cost/min | Control | Pick it when… |
|---|---|---|---|---|
| Managed (Vapi/Retell) | Days | Higher (platform fee + providers) | Medium | Pilots, SME clients, quick validation |
| Open-source framework (LiveKit/Pipecat) self-hosted | Weeks | Lower at scale | High | Productizing across many clients |
| Telephony IVR + LLM only for intent | Days | Low | Medium | Simple routing ("press 1" replaced by speech) |

## How It Works
```mermaid
sequenceDiagram
  participant C as Caller
  participant T as SIP / CPaaS
  participant A as Agent (LiveKit/Pipecat or managed)
  participant L as LLM + tools (bookings DB)
  C->>T: dials +60 3-xxxx xxxx
  T->>A: media stream (8 kHz audio)
  A->>C: greeting + AI disclosure + recording notice (TTS)
  loop each turn (< ~1 s target)
    C->>A: speech (VAD detects end of turn; barge-in stops TTS)
    A->>A: streaming STT → text
    A->>L: text + context → tool call (check_slots) → reply
    L-->>A: reply text (streamed)
    A->>C: streaming TTS audio
  end
  A->>T: transfer to human (SIP REFER) on request/low confidence
  A->>L: post-call: summary, outcome, recording link → CRM/WhatsApp follow-up
```

## Practical Usage
**Latency budget per turn** (cascaded):
```text
VAD end-of-turn detection   150–300 ms
STT final transcript        100–300 ms (streaming)
LLM first token             200–500 ms (fast model, short prompt, cached system prompt)
TTS first audio             100–250 ms
Network/telephony           100–200 ms
Target total                ≈ 0.7–1.2 s  → tools must answer in < 300 ms or say "let me check…"
```
**Agent instructions skeleton**:
```text
You are the phone receptionist for {{business}} in {{city}}. Speak briefly (1–2 sentences).
Language: reply in the caller's language (Malay, English or Mandarin); keep names and numbers clear.
Start: "Hi, this is {{business}}'s AI assistant. This call may be recorded for quality."
You can: answer hours/prices/location, check and book slots (confirm date, time, name, phone back to caller).
Always offer: "I can connect you to our staff" — call transfer_to_human when asked, unsure, or upset caller.
Never take card numbers or IC numbers by voice.
```
**Tool design for voice** (fast, confirmable):
```ts
const tools = {
  check_slots: async ({ service, date }) => db.slots(service, date).then((s) => s.slice(0, 3)),   // offer max 3 options
  create_booking: async ({ slotId, name, phone }) => db.book({ slotId, name, phone, source: "voice" }),
  transfer_to_human: async ({ reason }) => telephony.transfer(env.STAFF_NUMBER, reason),
};
```
**Post-call automation** (n8n webhook):
```text
call.ended → {transcript, summary, outcome, duration, recording_url}
  → Postgres call_logs → CRM activity → WhatsApp confirmation template to caller → Telegram alert if outcome=complaint
```
**Wrong way**:
```text
❌ Long LLM answers read aloud                 → callers interrupt or hang up
❌ No barge-in                                  → bot talks over the caller
❌ Collecting card/IC numbers by voice          → PCI/PDPA exposure in recordings and transcripts
❌ No human transfer path                       → angry callers, reputational damage
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Short turns, confirm key details back | Bookings | Monologues |
| Max 3 options per question | Slot selection | Reading a full schedule |
| Filler phrases while tools run ("let me check") | Slow tools | Dead air |
| Human transfer + after-hours voicemail | All deployments | Bot-only lines |
| Post-call summary to CRM/WhatsApp | Lead capture | Calls with no record |
| Pilot with real callers, review transcripts weekly | Launch | Testing only with the team's accents |
| Per-call duration caps | Cost control | Unlimited loops with spam callers |

## Performance & Trade-offs
- **Speech-to-speech vs cascaded**: realtime models feel more natural and faster, but cost more per minute and give less control over tool calls and transcripts. Cascaded allows choosing the best STT for Malaysian accents and cheaper LLMs.
- **Cost per minute** stacks telephony + STT + LLM + TTS (+ platform fee on managed services). Track per call and per tenant ([[AI Evals & Observability - Cost Tracking & Budgets per Tenant]]).
- **PSTN audio** is 8 kHz narrowband, so STT accuracy drops vs app/WebRTC audio. Names, emails and numbers are the hardest parts. Confirm them, or send a WhatsApp form link for complex details.
- **Self-hosting** media servers (LiveKit) on a VPS needs UDP ports, TURN and bandwidth. Managed platforms remove that ops burden.

## Tips & Reminders
> [!tip] In ZP's stack
> - **Start with WhatsApp, add voice where calls dominate**: clinics, salons, property agents and F&B take many phone bookings. An "AI receptionist" that books slots and sends a WhatsApp confirmation is a strong upsell to the WhatsApp agent ([[AI Agents - Customer Support Agents for WhatsApp & Messaging]]).
> - **Pilot on a managed platform** (fast, measurable), and move to LiveKit/Pipecat only if volume justifies it.
> - **Same tools/backends** as the WhatsApp agent (slots, bookings, FAQ RAG). Only the channel layer changes.

- Disclose AI and recording at call start. Store recordings/transcripts with retention limits (PDPA).
- Test numbers (`012…`, `+60…`) and dates spoken in BM/EN, and normalize before tool calls.
- Monitor: answer rate, transfer rate, booking conversion, average handle time, cost per call.
- Provide staff with a dashboard of calls needing callback.

## Version Notes
| Date | Change relevant here |
|---|---|
| 2024 | Realtime speech-to-speech APIs from major providers. LiveKit Agents and Pipecat frameworks mature |
| 2025 | Managed voice-agent platforms grow (Vapi, Retell). Better turn-detection models and multilingual TTS |
| 2025–26 | Lower realtime pricing and latency. SIP integration in agent frameworks |

> [!warning] Unverified — check before relying on this
> Provider pricing per minute, Malaysian number availability on specific CPaaS providers, local regulatory requirements for AI calls/recordings (MCMC, PDPA guidance) and BM/Chinese STT quality per vendor weren't verified this run. Test vendors with real Malaysian call samples before committing.

## Critical Issues & Gotchas
> [!danger] Consent, disclosure and sensitive data in recordings
> Recording calls and processing them with third-party AI providers involves personal data (and possibly health or financial information for clinics/insurers). Without a disclosure, consent, a retention policy and vendor DPAs, the client carries PDPA risk. Announce AI and recording, avoid collecting payment/IC data by voice, restrict recording access, and delete per retention policy.

> [!danger] Hallucinated commitments over the phone
> A voice agent that confirms non-existent slots, quotes wrong prices or promises refunds creates real obligations and angry customers. Spoken errors are harder to audit than chat. Ground every factual answer in tools/knowledge, confirm bookings via tools (not model memory), send written confirmation (WhatsApp/SMS), and transfer when unsure.

- **Spam/robocalls** can burn minutes. Add caller rate limits and max call duration.
- **Telephony outages** need a fallback (forward to a staff phone).
- **Accent/language mismatches** frustrate callers. Offer "press 1 for staff" or a language switch early.
- **Echo and noise** on mobile calls break VAD. Tune thresholds with real recordings.

## Related
- [[AI Agents]]
- [[AI Agents - Customer Support Agents for WhatsApp & Messaging]] — shared tools and handoff design
- [[AI Agents - Tool Calling & Agent Loops]] — fast, reliable tools
- [[AI Agents - Human-in-the-Loop & Approval Workflows]] — transfers and approvals
- [[WebSocket - SSE & Streaming LLM Responses]] — streaming media/text transport
- [[AI Evals & Observability - Cost Tracking & Budgets per Tenant]] — per-minute cost tracking

## References
- https://docs.livekit.io/agents/
- https://docs.pipecat.ai/
- https://docs.vapi.ai/
- https://docs.retellai.com/
- https://www.twilio.com/docs/voice/media-streams
