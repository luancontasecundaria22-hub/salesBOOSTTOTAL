# salesBOOSTTOTAL — Requirements Document (v1 / MVP-core)

> Source of truth for designing and building the platform. Assembled from the
> structured discovery interview (see `discovery-notes.md` for reasoning/risks).
> Intended as the handoff spec for the build phase (e.g. continued in Cowork).
>
> Status: **core requirements locked.** A few decisions are marked
> `DECISION` — pick before/early in build. Everything else is agreed.

---

## 1. Vision

An **internal, operator-run automation system** ("done-for-you" / agency model)
that converts inbound leads into ready-to-send, personalized sales replies and
follow-ups — with a human (the client) approving anything risky.

- We operate the system **on behalf of clients**; clients do **not** get a
  self-serve SaaS account (yet). The product sold is the **outcome**, not software.
- First system: **Inbound Lead Conversion Agent Network.**
- Canonical pipeline:
  `Client channels → Make → Supabase → LLM agents → Save results → Discord alert → (client) approval → Send`.
- Design principle: a **chain of single-job specialist agents** (one job, one
  input, one output). Human-in-the-loop before risky actions.

---

## 2. Business Model

- **Agency / done-for-you first** (~12 months). Architect for **one operator
  serving many clients**; keep the SaaS door open but don't pay the
  multi-tenant self-serve tax yet.
- **Pricing:** ~R$/USD **1.5k setup + 1k–2k/mo** retainer per client.
  No pure performance/per-lead pricing at start.
- **Year-1 target:** **15–25 clients** (design for ~30).

---

## 3. Users & Roles

| Role | Who | Can do |
|---|---|---|
| **Operator / Admin** | Founder | Everything: configure clients, agent logic, thresholds; sees escalations + health alerts; manages credentials & billing. |
| **Client Approver** | Each client (1–3 people) | Approve/reject medium-confidence replies for **their own** leads, in **their own** Discord server. Sees what was auto-sent. |
| **Helper / Monitor** *(later)* | Optional VA | Monitor system health & unresponsive clients. **No** credentials/billing access, **no** agent-logic changes. |

- **The approver is the CLIENT, not the operator** — this is what removes the
  solo-operator approval bottleneck.

---

## 4. Data

### Stored per lead
`name`, `contact_handle`, `raw_message`, `source`, `timestamp`, plus AI-derived:
`intent`, `signals`, `confidence_score`, `drafted_reply`, `approval_status`,
`sent_at`, `channel`. **Includes the client's customers' PII.**

### Channels (MVP)
- **Inbound + outbound: WhatsApp Business (Cloud API) + Instagram Messaging.**
- Replies go out on the **same channel** the lead arrived on.

### Retention
- **Purge raw PII after 30 days**; hard-delete on client offboarding.
- **Retain anonymized aggregates** (counts, scores, outcomes) beyond 30 days for
  analytics.

### System of record
- **Supabase (Postgres)** is the system of record. **Make is plumbing only.**

### ⚠️ Hard platform constraint — Meta 24-hour window
- Free-form messages only allowed **within 24h** of the customer's last message.
  Outside 24h: WhatsApp = **approved templates only**; Instagram = **limited
  message tags only**.
- ⇒ Approval must happen **inside the window** → drives the auto-send model (§6).
- ⇒ Per-client onboarding requires connecting **each client's own** WhatsApp
  number + Instagram Professional account.
- ⇒ **Meta App Review + Business Verification** required for production — weeks
  of lead time; start early.
- ⇒ Maintain a set of **pre-approved WhatsApp templates** for outside-window
  re-engagement and follow-ups.

---

## 5. AI Agent Architecture (two layers)

### Layer A — Functional pipeline (HOW a lead is processed)
Single-job sub-agents; only the Copywriter emits final customer-facing text.

| # | Sub-agent | Input → Output |
|---|---|---|
| 1 | **Intake** | raw lead → normalized JSON (deterministic, cheap) |
| 2 | **Extraction** | lead → intent, signals, urgency, buyer type, `confidence_score` |
| 3 | **Enrichment** *(post-MVP)* | + industry/company/use-case/angle |
| 4 | **Strategy** *(post-MVP)* | → tone, hook, value prop, objection handling, CTA |
| 5 | **Copywriter** | strategy → final reply + follow-up sequence |
| 6 | **Sender** | final message → send OR queue for approval |

**MVP = Intake + Extraction + Copywriter** (collapse Strategy/Enrichment into
the Extraction/Copywriter reasoning for now). Split out later only if a step
proves it needs isolation.

### Layer B — Business agents (WHO owns a metric) — mostly post-MVP
Run on a **different cadence** (batch/scheduled), not per-lead.
- **Customer Agent** — the revenue agent; Layer A lives inside it. **(MVP)**
- **Marketing Agent** — top-of-funnel intelligence; messaging angles. *(later)*
- **Analytics Agent** — truth layer; conversion, response time, lead quality. *(later)*
- **Customer Success Agent** — retention/onboarding/upsell. *(later)*

> Guardrail: business agents may **analyze** and recommend, but cross-agent
> **self-modification** of live behavior must pass a human gate and be logged
> (prevents feedback-loop drift).

> Agents do **not** scale with client count — the **same agent definitions run
> per client, scoped by `client_id`.** Fixed small set (3 MVP → ~6–10 later).

---

## 6. Automation — Approval Model (core of the product)

**Tiered approval by confidence ("approve-by-exception"):**

| Confidence | Action |
|---|---|
| **High** (≥ ~0.85, per-client tunable) | **Auto-send** immediately; log; notify client *after*. |
| **Medium** (~0.6–0.85) | **Hold for client approval** in their Discord server, with a **timeout** → auto-send or drop (protects the 24h window). |
| **Low** (< ~0.6) | **Escalate to operator**; never auto-send. |

- **Auto-send high-confidence: YES**, with a per-client **kill-switch**
  ("approve everything" mode for nervous clients).
- **Threshold is per-client** (sensible default, adjustable).
- **Permanent human hard-stops** regardless of confidence:
  **price/quotes, refunds, legal/contractual commitments, complaints.** A
  guardrail classifier must detect these and force human routing.
- **Follow-ups: auto-respond** on schedule via pre-approved sequence templates
  (respect 24h window → templates outside it).
- **NOT automated in v1:** closing/negotiation, pricing, contracts — AI hands
  warm leads to the human.

---

## 7. Security & Compliance

- **Layered client isolation:**
  - Discord: **one server per client.**
  - Supabase: `client_id` on every row + **Row-Level Security** (a query can
    never return another client's rows, even with an app bug).
  - **AI context scoped to one `client_id`** — no cross-client data ever in a
    single LLM prompt.
- **Credentials encrypted at rest** (Supabase Vault / secrets manager). Never
  plaintext in config or Make fields. Per-client WhatsApp/IG tokens.
- **Compliance: LGPD + Meta Platform Terms** (Brazil default). GDPR layers on
  if any client serves EU customers.
- **Consent & opt-out tracking from day one** — honor WhatsApp opt-in + "STOP"
  opt-out; system tracks consent state.
- **Immutable audit trail** — every AI-sent message, approval, and auto-send
  (who/what/when, human vs. AI).

---

## 8. Infrastructure / Stack Mapping

| Component | Responsibility |
|---|---|
| **Make** | Ingestion & routing plumbing (webhooks → normalize → write Supabase → trigger agent run → deliver outbound). No AI logic/storage. |
| **Supabase** | System of record + auth + secrets (Postgres, RLS, Vault, Edge Functions). |
| **Claude** | Reasoning across agent steps (extraction, strategy, copywriting). |
| **OpenAI** | `DECISION` — see below. |
| **Discord** | Per-client approval & notification surface (one server/client). |
| **MCP servers** | The agents' typed tools (read/write Supabase, send WhatsApp/IG, post Discord). |
| **Orchestrator** | Runs the agent chain per lead, scoped by `client_id`; calls LLM + MCP tools; manages state. (Thin service or Supabase Edge Functions — keep it controlled, not a black box.) |
| **Dashboard** | Deferred post-MVP; Discord is the interface for now. |

> `DECISION` — **LLM vendor.** Recommended: **Claude-only, model-tiered**
> (Haiku for cheap steps: intake, extraction, hard-stop classifier; Sonnet/Opus
> for strategy + copywriting). Add OpenAI later only if a step measurably wins or
> for cross-vendor fallback. Alternative: OpenAI for cheap high-volume steps +
> Claude for reasoning (more plumbing). **Default to Claude-only for MVP.**

---

## 9. Scale (design targets)

- **Clients:** design for ~**30** (year-1 actual 15–25).
- **Volume:** ~200–1,000 leads/mo per client → **~25k leads/mo aggregate**,
  campaign-day peaks 2–3×. ~3 LLM calls/lead → **~75k calls/mo**.
- **Required:** LLM **rate-limit handling + retry/queue** (not optional).
- **Users:** small (~25 clients × 1–3 approvers + operator). No heavy
  user-management system.
- **Real bottleneck = onboarding** (Meta connection + Discord server + config +
  template approval per client), not compute → needs a **repeatable onboarding
  runbook** as a v1 concern.

---

## 10. Roadmap

### MVP (build first)
1. One Supabase project: schema (leads, results, audit, consent, client config)
   + **RLS** by `client_id`; Vault for tokens.
2. WhatsApp + Instagram inbound via Make → write lead → trigger agent run.
3. Orchestrator runs **Intake → Extraction → Copywriter** (Claude), scoped per
   client.
4. **Tiered approval**: auto-send high-confidence; medium → Discord approval
   (per-client server) with timeout; low → operator escalation.
5. **Hard-stop classifier** (price/refund/legal/complaint → human).
6. Outbound send on same channel; **audit log** + post-hoc client notification.
7. **Consent/opt-out** tracking; pre-approved WhatsApp templates for follow-ups.
8. **Onboarding runbook** for connecting a new client.

### V2
- Enrichment + Strategy sub-agents split out; follow-up sequences matured.
- **Analytics Agent** (conversion, response time, lead quality, approval latency).
- Per-client tuning UI (still operator-side).

### V3
- **Marketing Agent** + **Customer Success Agent** (full business-agent loop with
  human-gated self-tuning).
- **Client-facing dashboard** / read-only reporting.
- Path toward optional **self-serve SaaS** (real multi-tenancy) if validated.

---

## Open Decisions (resolve early)
1. **LLM vendor** — default **Claude-only model-tiered**; confirm or switch to
   OpenAI+Claude. (§8)
2. **Client/customer countries** — confirm to finalize LGPD-only vs. +GDPR. (§7)
3. **Orchestrator implementation** — thin custom service vs. Supabase Edge
   Functions. (§8)

## Top Risks (carry into build)
- **Meta 24h window + App Review timeline** — start verification immediately.
- **Client approval latency** — mitigated by auto-send + timeouts; track per client.
- **Cross-client data leakage** — RLS + per-`client_id` prompt scoping are
  non-negotiable.
- **Confidence calibration** — must validate before trusting auto-send in prod.
- **Scope creep** — hold MVP at 3 sub-agents / Customer Agent only.
