# Discovery Notes — salesBOOSTTOTAL

> Living document. Captures answers from the structured discovery interview.
> The final requirements document will be assembled from this once all 10
> sections are complete. **No architecture is committed yet — these are inputs.**

Status: **in progress** — Section 1 locked; agent model previewed (Section 5).

---

## Section 1 — Vision (LOCKED)

**What this is:** An internal automation cockpit operated *by us* (done-for-you /
agency model). Clients do **not** log in initially. The product we sell is the
**workflow outcome**, not the software.

**Canonical processing pipeline:**

```
Client systems → Make → Supabase → OpenClaw + Claude → Agent analysis
  → Save results → Discord alert → Human approval → Action
```

**First system to build:** "Inbound Lead Conversion Agent Network."
Takes raw inbound leads → qualified intent → sales-ready insight →
ready-to-send personalized reply + follow-up sequence.

**Design principle:** chain of single-job specialist agents. One job, one input,
one output. Human-in-the-loop approval before any outbound action.

---

## Section 5 (PREVIEW) — Two-Layer Agent Model

### Layer A — Functional pipeline (HOW work is processed)
Sub-agents, each does ONE cognitive step. Only the Copywriter produces final text.

1. Intake (normalize raw lead → JSON)
2. Extraction (intent + sales signals + confidence_score)
3. Enrichment (industry/company/use-case/angle)
4. Strategy (tone, hook, value prop, objection handling, CTA)
5. Copywriter (final reply + follow-up sequence)
6. Sender (send OR queue for approval)

### Layer B — Business agents (WHO owns an outcome/metric)
Not steps — owners of goals. Run on different cadences than the pipeline.

- **Marketing Agent** — top-of-funnel intelligence; messaging angles, lead
  quality signals, channel performance. Does NOT reply to customers.
- **Customer Agent** — THE revenue agent. Converts leads → conversations → deals.
  The Layer-A pipeline lives *inside* this agent.
- **Analytics Agent** — truth layer; measures conversion, response time, lead
  quality, agent performance. Explains reality, does NOT act.
- **Customer Success Agent** — retention/onboarding/upsell after conversion.

**Key reframe:** building a "business simulation where each agent owns a metric,"
not "a bunch of agents doing tasks."

### MVP agent scope (per founder's own note)
Build only: Intake + Extraction + Copywriter (3 sub-agents, Customer Agent only).
Defer: Enrichment, Strategy, Sender automation, and all other business agents.

---

## OPEN RISKS / FLAGS (to resolve during interview)

- **Niche not yet chosen.** Pipeline is horizontal; the value lives in *which
  industry's* leads and *what action*. Must pick a first niche.
- **Cadence mismatch.** Customer Agent is event-driven (per lead); Marketing/
  Analytics/Success are batch/scheduled. Don't build them on the same trigger.
- **Scope creep.** 4 business agents + 6 sub-agents = 10 agents is too much for
  MVP. Hold the line at 3 sub-agents.
- **Feedback-loop stability.** Analytics → Marketing → Customer auto-tuning can
  oscillate or drift; needs human gate + change logging early.
- **Confidence scores need calibration** before they gate any automated action.
- **Discord-as-approval** lacks audit trail / client visibility — revisit in
  Infrastructure & Security sections.

---

## Section 2 — Business Model (LOCKED)

1. **Agency first, SaaS later.** Pure done-for-you for ~12 months. Architect for
   ONE operator serving MANY clients. Per-client data separation YES; client
   self-serve logins NO (yet). Keep SaaS door open, don't pay multi-tenancy tax now.
2. **Pricing:** ~$1.5k setup + ~$1k–$2k/mo retainer per client. No pure
   performance/per-lead pricing at start (attribution + billing complexity).
3. **Year-1 target: 15–25 clients.** Assume ~200–1,000 leads/mo each →
   **up to ~25k leads/month aggregate** at the top end.
4. **Delivery: solo operator** (founder). Cockpit must make ONE person fast.

### Scale implications of 15–25 clients (NEW — must address later)
- **Approval bottleneck is the #1 risk.** Solo + human-approval on every lead +
  up to 25k leads/mo = thousands of manual approvals. Pure 1-by-1 Discord
  approval will NOT scale. Need: auto-approve for high-confidence leads, batch
  approvals, and approve-by-exception. Revisit in Automation (Sec 6).
- Volume now justifies **a real queue + Claude rate-limit handling**, not just
  Make-triggered synchronous calls.
- Per-client isolation via `client_id` on every row is still fine (one Supabase
  project), but RLS / strict scoping becomes mandatory, not optional.
- **Unit economics check (later):** at $1–2k/mo × 20 clients = $20–40k MRR;
  must verify Claude+Make+infra cost per lead stays well under price.

---

## Section 3 — Users (LOCKED)

1. **The approver is the CLIENT, not the operator.** Each client approves/rejects
   replies for their OWN leads. This dissolves the solo-operator approval
   bottleneck — but introduces **client approval latency** as the new top risk.
2. **Clients ARE users now** (at least at the approval step) — they interact with
   the system, so they need a per-client interface.
3. **Strict per-client isolation required** ("personalized chats, not one server
   for everyone"). Decision: **one dedicated Discord server per client**
   (Option A) — strongest isolation, branded, leak-proof at the Discord layer.
   Data layer mirrors this: `client_id` on every row + RLS.
4. **Operators:** founder is admin/builder. A VA/helper may be added to monitor
   unresponsive clients & system health — NOT to approve (client does that).
   Roles: founder = full admin; helper (later) = monitoring only, no creds/billing.

### NEW top risk — client approval latency
- Inbound leads go cold fast; client-as-approver can stall the pipeline.
- Required mitigations (design in Sec 6):
  - reminder nudges to the client in their Discord server
  - timeout policy: auto-send high-confidence replies after N minutes, OR escalate
  - "client unresponsive" alert to the founder
  - track approval latency per client as a health metric (Analytics Agent)

---

## Section 4 — Data (LOCKED)

1. **Per-lead data stored:** name, contact handle, raw inbound message, source,
   timestamp + AI-derived fields (intent, signals, confidence, drafted reply,
   approval status). **Includes the client's customers' PII** → triggers
   privacy/compliance (Sec 7).
2. **Inbound channels (MVP): WhatsApp + Instagram** (both Meta platforms).
3. **Outbound:** reply in the SAME channel the lead arrived on.
4. **Retention:** purge raw lead data after **30 days**; hard-delete on client
   offboarding. KEEP anonymized/aggregated metrics beyond 30 days for Analytics.
5. **System of record: Supabase.** Make = plumbing/routing only, not storage.

### ⚠️ CRITICAL CONSTRAINT — Meta 24-hour messaging window
- WhatsApp Cloud API & Instagram Messaging only allow **free-form** messages
  within **24h of the customer's last message**. After 24h: WhatsApp = approved
  templates only; Instagram = limited message tags only. **No free-form AI reply.**
- Therefore client approval MUST happen inside the 24h window, ideally minutes.
  Reinforces **auto-send high-confidence + escalate-by-exception** (Sec 6).
- **Per-client connection onboarding:** each client connects their OWN WhatsApp
  Business number + Instagram Professional account (linked to a FB Page).
  This is the bulk of onboarding engineering.
- **Meta App Review + Business Verification** required for production multi-client
  use — takes weeks, gates launch. Budget timeline now.
- Pre-approved **WhatsApp message templates** needed for re-engagement outside 24h.

### Retention nuance
- 30-day purge of raw PII conflicts with Analytics needing history.
  Resolution: purge raw PII at 30d, retain anonymized aggregates (counts,
  scores, conversion outcomes) for trend analysis.

---

## Section 6 — Automation (LOCKED)

**Core model: tiered approval by confidence (approve-by-exception).**

| Confidence | Action |
|---|---|
| High (≥ ~0.85, per-client tunable) | **Auto-send** immediately, log, notify client after |
| Medium (~0.6–0.85) | Hold for **client approval** in their Discord server, with **timeout** → auto-send or drop (protects 24h window) |
| Low (< ~0.6) | **Escalate to operator (founder)**, never auto-send |

1. **Auto-send high-confidence: YES.** Each client gets a **kill-switch** to flip
   their account to "approve everything" if nervous.
2. **Confidence threshold: per-client** (sensible default, adjustable).
3. **Permanent human hard-stops** (regardless of confidence): price/quotes,
   refunds, legal/contractual commitments, complaints.
4. **Follow-ups: auto-respond** on a schedule using pre-approved sequence
   templates (must respect 24h window → use WhatsApp templates outside it).
5. **NOT automated in v1:** closing/negotiation, pricing, contracts. AI hands
   warm leads to the human for these.

### Implications
- Need a **confidence calibration** process before auto-send is trusted in prod.
- Need a **content classifier / guardrail** to detect hard-stop topics (price,
  refund, legal, complaint) and force human routing even on high confidence.
- Need a **timeout/scheduler** for medium-tier holds and follow-up sequences.
- Every auto-sent message must be **logged + surfaced to the client** after the
  fact (trust + audit).

---

## Section 7 — Security (LOCKED)

1. **Client isolation (layered):**
   - Discord: one server per client.
   - Supabase: single project, `client_id` on every row, **RLS enforced** —
     a query can never return another client's rows even with an app bug.
   - AI context: every agent run scoped to ONE `client_id`; **no cross-client
     data ever enters a single Claude prompt.** (Biggest real-world leak vector.)
2. **Credentials encrypted at rest** (Supabase Vault / secrets manager). Never
   plaintext in config or Make scenario fields. Per-client WhatsApp/IG tokens.
3. **Compliance: LGPD + Meta Platform Terms** (Brazil default). If any client
   serves EU customers → GDPR layers on top. (Confirm client/customer countries
   when known.)
4. **Consent & opt-out tracking built in from day one.** Honor WhatsApp opt-in
   requirement + "STOP" opt-out across all flows. System tracks consent state.
5. **Immutable audit trail** for every AI-sent message, approval, and auto-send
   (who/what/when, human vs AI). In-scope for v1. Needed for client trust +
   Meta/legal disputes.

---

## Section 8 — Infrastructure (LOCKED except LLM-vendor decision)

1. **Make** = ingestion & routing plumbing (WhatsApp/IG webhooks → normalize →
   write to Supabase → trigger agent run → deliver outbound). No AI logic/storage.
2. **Supabase** = system of record + auth + secrets (Postgres for leads/results/
   audit/consent/per-client config; RLS isolation; Vault for client tokens;
   Edge Functions for server-side logic).
3. **Claude** = reasoning in agent steps (extraction, strategy, copywriting).
4. **Discord** = per-client approval & notification surface (one server/client).
5. **MCP servers** = the agents' typed tools (read/write Supabase, send
   WhatsApp/IG, post to Discord).
6. **Dashboard** = deferred post-MVP; Discord is the interface for now.

### "OpenClaw" clarified = OpenAI
Founder's stack note "OpenClaw plus Claude" meant **OpenAI + Claude**.
**OPEN DECISION:**
- Option 1 (RECOMMENDED): **Claude-only, model-tiered** — Haiku for cheap steps
  (intake, extraction, hard-stop classifier), Sonnet/Opus for strategy +
  copywriting. OpenAI deferred. Less operational overhead for a solo operator.
- Option 2: **OpenAI + Claude** — OpenAI for cheap high-volume steps, Claude for
  reasoning/copywriting. More capability, more plumbing (2 keys/SDKs/rate limits).
→ Awaiting founder choice.

---

## Sections 9 (Scale) & 10 (Roadmap)
Locked with designed-to defaults directly in `requirements.md` (founder opted to
finalize the core spec and continue the build in Cowork).

---

## STATUS: discovery complete → see `requirements.md` for the final spec.
