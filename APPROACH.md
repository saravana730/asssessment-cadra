# Approach

Delivery approach for a BAA-covered, voice-and-chat patient access agent for clinics.
Phase 1 targets a single clinic, English and Spanish, read-only operations (no writes).
Medicare payers are the first vertical: they represent roughly 40% of call volume.

The companion document [`ARCHITECTURE.md`](./ARCHITECTURE.md) holds the system diagrams
and component-level detail referenced throughout this document.

---

## 1. Scope and Objectives

- Conversational assistant reachable by **phone (voice)** and **chat**, sharing one agent definition.
- **Identity verification** before any PHI is released.
- **Policy and benefit Q&A** via retrieval-augmented generation (RAG) with citations.
- **Scheduling and eligibility** support, with write operations gated and confirmed.
- **Escalation to staff** whenever the agent is not authorized or confident.
- **No query goes unanswered**: anything the agent cannot resolve becomes a tracked follow-up task.

Phase 1 deliberately limits scope to make the agent architecture and the chat/call pipeline
reliable before broadening functionality:

1. One clinic.
2. English and Spanish, read-only operations only (no updates in this phase).
3. Validate and benchmark call accuracy, especially in Spanish.
4. After-hours handling deferred to a later phase.

---

## 2. Open Questions for the Client

### 2.1 Integrations

1. **Do we have FHIR or Epic API integration?** These vendors charge separately for API access.
   - If an API is available, use it to read and update the application directly.
   - If not, use a screen-based / image-based agent to read information; any update or creation
     becomes a task for clinic employees to complete.

### 2.2 Volume and Capacity

2. **Is the 2,800-call figure per clinic or overall?**
   - If overall: size the architecture for 3,000–3,500 calls per day.
   - If per clinic: size for approximately 39,200 calls per day (2,800 × 14 clinics).

### 2.3 Call Handling

3. **Missed calls**: how are they handled? Is there a call-back, how is it recorded, and how is the
   resulting action verified?
4. **Wait handling**: are callers placed on hold/queue? Is there an average-wait-time announcement?
5. **Per-contact output**: is a summary and resolution required for every call/chat contact?

### 2.4 Commercial

6. **Cost per call/chat per minute**: what is the target? This drives the choice of LLM, ASR, TTS,
   and STT models against the customer's cost expectations.

> Payer-specific questions are listed in section 3.4.

---

## 3. Medicare Payer Flows

Medicare is the largest payer group by far (~40% of call volume) and is **not a single flow**. It
splits into two distinct payers with different rules:

- **Original Medicare (Parts A & B)** — federal program. No network, no referrals, prior auth is rare
  (mainly DME and certain items). Claims go to the MAC.
- **Medicare Advantage (Part C)** — a private plan. Has network, referral, and prior auth
  requirements; benefits change per plan and per year.

### 3.1 Flow A — Original Medicare (Parts A & B)

Payer is Medicare. No network limits, no referrals, and prior auth is rare (mostly DME / certain
items). The patient pays 20% coinsurance unless a Medigap plan covers it.

Agent steps:

1. Verify member — name, DOB, Medicare number, Part A/B effective dates.
2. Confirm the clinic accepts Medicare assignment / is taking new Medicare patients.
3. Capture secondary coverage — Medigap or Medicaid — since billing must know who pays the 20%.
4. Schedule / route; no referral required.
5. Drugs fall under a separate Part D plan — route drug questions to pharmacy or staff.

Escalate to staff: no secondary coverage on a high-cost service, Medicare + Medicaid (dual eligible),
DME / prior auth items.

### 3.2 Flow B — Medicare Advantage (Part C)

Payer is a private plan. The member card is not the red-white-blue card. Network, referrals, and
prior auth all apply.

Agent steps:

1. Capture plan name, plan ID, member ID, group, effective date.
2. Verify eligibility — real-time 270/271 if we have API or portal access; otherwise capture the data
   and create a task.
3. Check in-network status for this clinic — out of network means stop and route to staff.
4. Check whether a referral is required (usually a PCP referral for a specialist) and whether prior
   auth is required for the service.
5. Start prior auth, or create a task for clinic employees.
6. Schedule / route with the plan recorded on the appointment.

Escalate to staff: out of network, prior auth required, plan not in our payer list, member ID
mismatch, DME.

### 3.3 Both Flows

- The **same fields** are written to the EHR for both payers: payer, plan, member ID, group, effective
  date, secondary payer. One data model, two rule sets.
- Annual enrollment runs **Oct 15 – Dec 7**, and benefits change every year. Payer rules must therefore
  be a **configuration table**, not hardcoded logic.
- Medicare Advantage is subject to **CMS rules** on what we may say (no steering, no plan comparison).
  Scripts require compliance review per flow.

### 3.4 Payer-Specific Questions for the Client

a. The split — how much Original Medicare versus Medicare Advantage, and the top MA plans by call volume.
b. Do we get real-time eligibility (API / portal), or do we only capture information for staff?
c. Who owns prior auth and referral — the agent or staff?
d. The top 10 payers by volume, so payer rules can be built and tested.

---

## 4. Assumptions

1. **Language and identity pre-load.** Patient phone number and preferred language are stored, so an
   inbound call configures the agent language from that record. To control cost, use small models that
   are strong in English and Spanish. If no database record exists, default to small bilingual models
   with good accuracy in both languages.
2. **Every contact is logged and resolved.** All calls/chats are logged with the resolution provided.
   If human intervention is needed and staff are unavailable, the system creates a follow-up task; it
   later checks staff availability, calls the patient, and connects them, so no call or query goes
   unnoticed.
3. **No PHI before verification.** No patient-related information is provided over call, chat, or SMS
   until identity is verified: phone number match, DOB, and one additional personal question from the
   database.
4. **Compliant LLM.** Use a walled (walled-garden) LLM under a signed BAA, or a self-hosted LLM.
5. **PHI is not passed to the LLM directly.**
6. **Data protection.** Stored data (name, contact, DOB) is encrypted at rest; users can request
   deletion of their information at any time via call or chat.
7. **Disclosure.** The agent always states that it is an automated assistant and that the call is
   recorded.
8. **Emergencies (code red).** The agent must not attempt to handle emergencies itself; it always
   transfers or instructs the patient to call 911.
9. **Confirm before changing.** The agent always summarizes and obtains confirmation before performing
   any change operation.
10. **Appointment authorization.** Any appointment creation requires authorization and an eligibility
    check.

---

## 5. Solution Design

Serverless, container-based cloud architecture with a BAA signed with the provider, sized to scale to
large call volumes.

### 5.1 Tech Stack

| Layer | Choice |
| --- | --- |
| Infrastructure | Serverless containers on a cloud with a signed BAA; horizontal scaling |
| Backend | Python, FastAPI (WebSocket for voice, SSE for chat) |
| Async processing | Celery (queues and jobs) |
| Agent framework | Haystack / LangChain / custom agent builder |
| Session storage | Redis |
| Source of truth | Postgres |
| Portal | React |
| Vector / RAG | Quadrant vector DB (policy documents ingested, answers read from the full file) |
| Object storage | S3 (or same server) for call audio, BAA-signed |
| Telephony | Twilio or equivalent |

### 5.2 Reusable Agent Layer

- **Tools and prompts are shared between chat and voice**, so agents are reusable for future purposes.
- New agents are added as definitions (prompt, tools, guardrails, flow config) rather than new services.
- Payer-specific behaviour (Original Medicare vs Medicare Advantage) is a flow configuration, not
  branching code.

### 5.3 Chat Pipeline (SSE)

- Delivered over **SSE** so the API can emit thinking/processing events before the final answer.
- On a question: the LLM generates the answer while tool calls and thinking events are emitted to keep
  the user engaged.
- Chat turns are limited — not all turns are sent to the model.
- Queries are **rewritten** before processing.
- After N turns, a **summary** is generated for older turns.
- **Citations** are added for RAG-based Q&A.

### 5.4 Voice Pipeline (WebSocket)

- Calls are WebSocket-based. On each call, check first whether the patient is in the database; if so,
  pick the preferred language and converse in it.
- Pipeline consists of **LLM** (large language model), **TTS** (text to speech), and **STT** (speech to text).
- Audio is processed in a parallel WebSocket connected to **VAD**:
  - When speech starts and stops (a full stop), the utterance is sent to STT and appended.
  - It is sent to the LLM while the system keeps listening once the user stops speaking.
  - Everything is processed together and sent to TTS, so low latency is achieved.
- All turns are stored in **Redis per session** rather than updating Postgres every turn, to avoid added
  latency and minimize database hits. Postgres is updated in batches instead.

### 5.5 Agent-Level Governance

- All actions are logged with reason and citations for debugging (**dev only**); production keeps a
  minimized version, not full logs.
- **Guardrails** are applied to any data fetched from the database or an API.
- The agent is never given direct database access. Access is via an API with permissions restricted to
  the particular user in the call.

### 5.6 Architecture Reference

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the system context, voice and chat sequence diagrams,
data/compliance tables, and scaling notes.

---

## 6. Delivery Plan — Eight-Week Scope

1. Set up the system for one clinic.
2. English and Spanish, **read-only** operations (no updates yet).
3. Check and validate calls and accuracy in Spanish.
4. Defer after-hours calls to a later phase.

Rationale: focus the first phase on standing up the agent architecture and the chat/call pipeline,
benchmark Spanish accuracy, and run two to three iterations to reach a solid baseline.
