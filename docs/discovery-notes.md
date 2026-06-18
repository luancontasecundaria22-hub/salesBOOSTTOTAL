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

## Sections still to cover
3. Users · 4. Data · 6. Automation · 7. Security · 8. Infrastructure ·
9. Scale · 10. Roadmap
