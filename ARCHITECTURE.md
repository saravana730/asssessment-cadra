# Solution Architecture

Serverless container based, BAA-signed end to end, shared agent definitions across voice and chat,
Redis for session state, Postgres as source of truth, Quadrant for policy RAG, object storage for audio.

Diagrams are Mermaid - they render on GitHub and most Markdown viewers (VS Code preview, Obsidian, etc.).

---

## 1. System context (container view)

```mermaid
flowchart TB
  subgraph CLIENTS["Clients"]
    PAT["Patient (phone)"]
    STAFF["Clinic staff (React portal)"]
  end

  subgraph EDGE["Edge"]
    TEL["Twilio Programmable Voice<br/>Media Streams WebSocket"]
    WAF["API Gateway + WAF<br/>TLS termination"]
  end

  subgraph CORE["Core services - serverless containers, autoscaled"]
    WSS["Voice WS Gateway<br/>FastAPI WebSocket"]
    SSES["Chat + Portal API<br/>FastAPI + SSE"]
    ORCH["Agent Orchestrator<br/>LLM + tools + memory"]
    AGENTS["Reusable Agent Definitions<br/>prompts, tools, guardrails"]
    GUARD["Guardrail + Policy Engine"]
    TOOLAPI["Restricted Data API<br/>per-call scoped tokens"]
  end

  subgraph SPEECH["Speech + AI pools"]
    VAD["VAD"]
    STT["STT / ASR"]
    TTS["TTS"]
    LLM["LLM"]
  end

  subgraph WORK["Async workers"]
    CEL["Celery workers"]
    BEAT["Celery beat"]
  end

  subgraph DATA["Data - BAA covered"]
    REDIS[("Redis<br/>session state, turns,<br/>broker, cache")]
    PG[("Postgres<br/>source of truth")]
    VDB[("Quadrant vector DB<br/>policy documents")]
    S3[("Object storage<br/>call audio")]
  end

  subgraph EXT["External systems"]
    FHIR["EHR - FHIR / Epic"]
    SCHED["Scheduling / PMS"]
    ELIG["Payer eligibility 270/271"]
    MSG["SMS / email follow-up"]
  end

  PAT --> TEL --> WAF --> WSS
  STAFF --> WAF --> SSES
  WSS --> ORCH
  SSES --> ORCH
  ORCH --- AGENTS
  ORCH --> GUARD --> TOOLAPI
  TOOLAPI --> PG
  TOOLAPI --> FHIR
  TOOLAPI --> SCHED
  TOOLAPI --> ELIG
  ORCH --> VDB
  ORCH --> LLM
  WSS --> VAD --> STT --> ORCH
  ORCH --> TTS --> WSS
  WSS --> REDIS
  SSES --> REDIS
  ORCH --> REDIS
  WSS --> S3
  REDIS --> CEL
  CEL --> PG
  CEL --> VDB
  CEL --> MSG
  BEAT --> CEL
```

Notes:

- Gateways are stateless; all conversation state lives in Redis, so any container can serve any turn.
- Websockets are long lived, so the voice pool needs a warm minimum and graceful drain on scale-in.
- The orchestrator never touches Postgres directly - only the Restricted Data API does (see section 5).

---

## 2. Voice call pipeline

```mermaid
sequenceDiagram
  autonumber
  participant P as Patient
  participant TW as Twilio Media Streams
  participant WS as Voice WS Gateway
  participant VAD
  participant STT
  participant OR as Agent Orchestrator
  participant TOOL as Tools + Restricted API
  participant TTS
  participant R as Redis
  participant CEL as Celery worker
  participant DB as Postgres

  P->>TW: speech
  TW->>WS: audio frames
  WS->>VAD: frames
  VAD-->>WS: speech start / stop
  WS->>STT: utterance audio
  STT-->>OR: transcript plus partials
  Note over WS,R: first turn - identify patient, pick preferred language, then continue in it
  WS->>R: load or create session
  OR->>R: load short term context
  OR->>TOOL: tool calls - patient, schedule, payer check, policy RAG
  TOOL-->>OR: results plus citations
  TOOL->>R: append action log with reason plus citations
  OR->>TTS: response text streamed
  TTS-->>TW: audio deltas
  TW-->>P: spoken answer
  OR->>R: append turn
  R-->>CEL: session turns queued
  CEL->>DB: batched write, not per turn
```

Low latency approach:

- VAD detects speech stop, the utterance is sent to STT while the LLM prompt and tool plan are prepared in parallel.
- TTS consumes the LLM stream sentence by sentence, so the caller hears the first sentence before generation finishes.
- Barge-in: if VAD fires while TTS is playing, playback is cut and the new utterance wins.
- All N turns are kept in Redis for the session; Postgres is written in batches by Celery to avoid per turn DB hits.

---

## 3. Chat pipeline (SSE)

```mermaid
sequenceDiagram
  autonumber
  participant U as User - React portal
  participant API as FastAPI SSE endpoint
  participant R as Redis memory
  participant RW as Query rewriter
  participant RET as Retrieval - Quadrant + reranker
  participant OR as Agent Orchestrator
  participant TOOL as Tools + Restricted API

  U->>API: POST chat - stream
  API->>R: load window plus rolling summary
  API-->>U: SSE - thinking
  API->>RW: rewrite query with history
  RW-->>API: standalone query
  API->>RET: retrieve policy chunks
  RET-->>API: chunks plus citations
  API-->>U: SSE - tool and retrieval events
  API->>OR: prompt - rewritten query, context, tool specs
  OR->>TOOL: tool calls if needed
  TOOL-->>OR: result plus citations
  OR-->>API: token stream
  API-->>U: SSE - answer tokens plus citations
  API->>R: append turn
  Note over R: every N turns, oldest turns summarized and dropped from the window
```

Notes:

- SSE carries `thinking`, `tool_call`, `retrieval` and `token` events so the user sees progress before the final answer.
- Only recent turns are sent to the LLM; older turns become a rolling summary.
- RAG answers always carry citations; full policy files are available to the reader view, not dumped into the prompt.

---

## 4. Reusable agent layer - shared by voice and chat

```mermaid
flowchart LR
  V["Voice channel<br/>WebSocket + STT/TTS"] --> RUN["Agent Runtime"]
  C["Chat channel<br/>SSE"] --> RUN

  subgraph DEF["Agent definition - shared, versioned"]
    PR["Prompt templates"]
    TLD["Tool / function specs"]
    GRD["Guardrail policies"]
    SCH["I/O + session schema"]
    MDC["Model / voice config"]
    FLOW["Payer flows<br/>Medicare vs Medicare Advantage"]
  end

  DEF --> RUN
  RUN --> O1["Voice out - text to TTS"]
  RUN --> O2["Chat out - SSE events + citations"]
```

- Same prompts, tools and guardrails serve both channels - only the transport and the input/output codec differ.
- New agents are added as definitions, not new services.
- Payer specific behaviour (Original Medicare vs Medicare Advantage) is a flow config, not branching code.

---

## 5. Data access + guardrails

```mermaid
flowchart TB
  OR["Agent Orchestrator"] --> IG["Input guardrails<br/>prompt injection, abuse, intent"]
  IG --> TL["Tool layer<br/>allow-listed actions only"]
  TL --> REG["Action registry<br/>schedule, patient lookup,<br/>payer check, policy RAG"]
  REG --> API["Restricted Data API<br/>scoped token per call / chat user"]
  API --> RBAC["RBAC + field level filters"]
  RBAC --> PG[("Postgres<br/>views + stored procs")]
  RBAC --> FH[["EHR / FHIR"]]
  RBAC --> EL[["Payer eligibility"]]
  TL --> OG["Output guardrails<br/>PHI leak, scope, tone, no steering"]
  OG --> OR
  TL --> AUD["Audit log<br/>action + reason + citations"]
  AUD --> DEVL["Dev - full logs and citations"]
  AUD --> PRODL["Prod - minimized, PHI redacted"]
```

- The agent has no direct DB or API credentials; every action goes through the registry and a token scoped to the current call or chat user.
- Dev keeps full action logs with reasoning and citations for debugging; prod keeps a minimized, redacted version.
- Output guardrails block PHI leakage and CMS-prohibited Medicare Advantage steering.

---

## 6. Async jobs - Celery

```mermaid
flowchart LR
  R[("Redis<br/>queues / streams")] --> CEL["Celery workers"]
  BEAT["Celery beat"] --> CEL

  CEL --> J1["Flush session turns<br/>Redis to Postgres in batches"]
  CEL --> J2["Ingest policy docs<br/>chunk, embed, upsert to Quadrant"]
  CEL --> J3["Rolling chat summaries<br/>after N turns"]
  CEL --> J4["Post call tasks<br/>notes, follow-up SMS / email"]
  CEL --> J5["Archive audio<br/>stream to object storage"]
  CEL --> J6["Eligibility + prior auth<br/>retries and status checks"]

  J1 --> PG[("Postgres")]
  J3 --> PG
  J2 --> VDB[("Quadrant")]
  J4 --> MSG["SMS / email"]
  J5 --> S3[("Object storage")]
  J6 --> EL[["Payer systems"]]
```

---

## 7. Data + compliance

| Store | Holds | Notes |
| --- | --- | --- |
| Redis | session state, turns, broker, cache, summaries | TTL per session, AOF on, not the system of record |
| Postgres | patients, appointments, actions, batch written transcripts | source of truth, encrypted at rest, row level access |
| Quadrant | embedded policy documents | separate collection per tenant / clinic, no PHI in the index unless required |
| Object storage | call audio, exports | BAA signed, retention policy, lifecycle expiry |

- BAA signed with every subprocessor: telecom, STT/TTS, LLM, vector DB, object storage, Postgres hosting.
- Two party consent: recording notice played before capture, consent flag stored per call.
- No direct DB access for agents, all access via the restricted API with per-call scoped tokens.
- Audit logging split: full in dev, minimized and redacted in prod.

---

## 8. Scaling + deployment

- Serverless containers per service, autoscale on CPU plus queue depth plus active websocket count.
- Voice pool keeps a warm minimum so calls never wait on a cold start; scale-in drains connections first.
- Sizing from the open questions: ~2,800 calls/day overall lands in the 3,000-3,500 peak target; if it is per clinic the target is ~39,200/day (2,800 x 14) and the speech and websocket pools scale accordingly.
- Annual Enrollment Oct 15 - Dec 7 is the peak window; payer rules and capacity must be sized for it.
- Celery workers scale independently of the gateways so ingestion and transcription never compete with live calls.

---

## 9. Open points / risks

- Batch writes: a crash before flush can lose the last turns - mitigate with AOF plus a quick flush on session end and idempotent upserts in Postgres.
- Websocket affinity: long lived connections need sticky routing or Redis backed handoff; verify drain behaviour under scale-in.
- Quadrant vendor BAA and whether policy documents may contain PHI - scope collections and access accordingly.
- Audio retention and consent rules vary by state; make retention configurable per clinic.
- Latency budget: set explicit p50 / p95 targets for VAD, STT, LLM first token and TTS first byte, and test with the 65+ caller population.
- Clinical safety: RAG answers must cite, low confidence must escalate to staff, and Medicare Advantage scripts need a compliance review.
