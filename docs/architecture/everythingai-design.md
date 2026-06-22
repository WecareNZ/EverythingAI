# EverythingAI — Architecture & Migration Design

> Status: **Draft for review** · Owner: _TBD_ · Clinical safety owner: _TBD_
> This document captures the agreed direction for consolidating WeCare Health's
> AI assistants into a single platform ("EverythingAI"), and the plan to
> migrate the Senior Clinician AI out of Heidikiller as the reusable clinical core.

## 1. Problem & goal

WeCare Health currently runs narrow assistants:

- **TUI** — junior-nurse clinical advice, bounded to standing orders.
- **KEA** — HR / company policy (billing, annual leave, patient cost, etc.).
- **Heidikiller** — in-consult ambient scribe **plus** a Senior Clinician AI
  (BPAC / HealthPathways evidence, "does this case meet referral criteria",
  referral drafting).

Gaps:

- A senior clinician asking about **non-standing-order** items has no good path
  in TUI (which only covers the standing-order set).
- Staff face multiple disjoint assistants for what feels like "ask the company a
  question".

Goal: **one front door** for all staff, without flattening the very different
risk profiles of clinical advice vs HR policy.

## 2. Core principle — one brain, many skills (not one prompt, not many AIs)

The unit of design is the **skill**, not the persona. A single foundation model
and a single shared context spine serve multiple skills, each with its **own
prompt, retrieval corpus, and guardrails**.

- **Not** one monolithic prompt doing everything → unsafe; the HR safety bar
  silently becomes the clinical bar.
- **Not** several independent AIs → duplicated context, divergent behaviour,
  double maintenance.

"Scribe" and "advice" are therefore **two skills sharing one brain and one
context** — not two AIs.

```
        Staff (junior clinician / senior clinician / any staff)
                              │
                    EverythingAI — one front door
                    router reads VERIFIED claims
                              │
   ┌──────────────┬───────────┼───────────────┬──────────────────┐
 Scribe        Clinical Q&A /        Referral            Non-clinical
 (capture)     evidence skill        drafting skill      skill (= KEA)
               (BPAC, HealthPathways,                    policy / HR /
               standing orders)                          dress code / cost
                              │
        shared CONSULT/SESSION CONTEXT + identity/claims + audit
```

## 3. Heidikiller is the core — extend, don't rebuild

Heidikiller already contains the hard parts of the clinical side: ambient
capture (the context spine), evidence integration (BPAC / HealthPathways), and
referral-criteria checking + drafting. EverythingAI is therefore **Heidikiller
extended**, not a parallel build.

The agreed shape:

1. **Heidikiller** keeps the **scribe** + the **Senior Clinician AI**
   (advice + referral) as its in-consult experience.
2. The **Senior Clinician AI** is extracted as the **reusable clinical core**
   and migrated into EverythingAI to serve all purposes.
3. EverythingAI adds two comparatively cheap pieces around that core:
   - a **router + role/claims layer**, and
   - the **non-clinical floor** (fold in KEA).

### Migration plan for the Senior Clinician AI (pending Heidikiller code read)

> Requires `WecareNZ/Heidikiller` to be added to the working session before the
> code-level steps can be done. Open question: how many models/prompts it uses
> today and whether scribe + advice already share one context.

- [ ] Read Heidikiller; document its model/prompt/retrieval topology.
- [ ] Identify the Senior Clinician AI boundary (prompt + retrieval + tools).
- [ ] Extract it into a shared package/service consumed by both Heidikiller and
      EverythingAI (single source of truth — no fork).
- [ ] Wrap with the EverythingAI router + claims so it can also serve junior
      (framed) and non-consult contexts.

## 4. Personas × capabilities

Capabilities are shared; **claims decide access and framing**, not the
conversation.

| Capability | Junior clinician | Senior clinician | Any staff |
|---|---|---|---|
| Standing-order guidance | ✅ primary | ✅ reference | — |
| Evidence & tools (BPAC, HealthPathways) | ⚠️ *framed*: "outside your standing orders → confirm/escalate" | ✅ primary | — |
| Referral drafting | ✅ (sign-off by tier) | ✅ | — |
| Non-clinical (policy, dress code, billing, leave) | ✅ | ✅ | ✅ (the floor) |

Notes:

- **Non-clinical is the floor**, available to everyone with no clinical claims —
  lowest risk, ship first.
- The junior↔senior line is **framing, not a wall**: a junior gets the evidence
  *with* the boundary made explicit, never a hard refusal.

## 5. Identity, claims & care-relationship

The model must be **told** the user's state by trusted systems — never infer
seniority/scope from the conversation (otherwise anyone unlocks prescriber-level
answers with a sentence).

Layers of "state":

| Layer | Example | Source | AI may infer? |
|---|---|---|---|
| Identity | employee #4821 | SSO | No — authenticated |
| Credential / scope | EN vs RN vs Nurse Practitioner (prescriber); endorsements | HR + credentialing system of record | **No — looked up** |
| Situational | on shift, ward, supervised vs independent | roster / session | Partly |
| **Care relationship** | authorised to act on *this* patient's record | EHR / care-team | **No — looked up** |

Rules:

- Claims are attached to the session as **trusted, non-overridable context**.
- **Fail safe:** if a system of record is unavailable, default to **lowest
  privilege + escalate** — never assume up.
- Role/scope changes (gain/lose prescribing) must propagate same-day.

## 6. Referral workflow (no PMS API)

There is **no API link to the PMS**. This is handled by using the **consult
itself as the context source** — the scribe already holds the consultation, so
the referral skill drafts from *captured consult + conversation*, not a PMS
pull. The typical flow:

> clinician asks → AI answers → clinician says "write a referral" → AI drafts
> from the context it already holds.

Non-negotiable sequence: **AI drafts → clinician reviews & edits → clinician
signs → system sends.** Never auto-send. The clinician remains the accountable
author.

"Linked" means three connections; with no API we get inbound for free via
capture, and outbound stays manual:

| Direction | With no PMS API |
|---|---|
| Inbound (read patient/encounter context) | **Solved by ambient capture** |
| Outbound (write referral back) | **Manual** — clinician pastes into PMS, or referral goes via HealthLink / ERMS as a separate transport |
| Audit linkage (patient + encounter + clinician) | Maintained in EverythingAI's audit log |

## 7. Safety, regulatory & data residency — PRESENT TENSE

"Does this case meet criteria for referral" is **active clinical decision
support**. These obligations already apply to Heidikiller today; consolidation
makes them organisation-wide, not new.

- **Medical-device classification.** Likely Software as a Medical Device
  territory (Medsafe; incoming Therapeutic Products Act). Get a regulatory read
  **before** widening clinical skills. Resolve first — it can change hosting and
  architecture.
- **Clinical risk management.** Clinical safety case, governance group, and a
  **named clinical safety officer** who signs off clinical skills and owns
  incidents.
- **Patient data privacy & residency.** Ambient capture = recording patient
  consultations (audio/transcript). Under the Health Information Privacy Code
  2020: per-consult **patient consent** to record; defined **processing/storage
  location** and whether data leaves NZ; **retention/deletion** policy. This is
  a hard gate for any PHI-handling skill.

## 8. Content lifecycle

The system is only as good as its corpora.

- Named **owners** per corpus (standing orders, policy/HR, evidence sources).
- **Version control, review dates, expiry** — a confidently-cited *lapsed*
  standing order is worse than no AI.
- **Approval gate** for what may enter each corpus.
- BPAC / HealthPathways must be **live retrieval / synced index with citations**
  — never the model's memory. Refuse/escalate when grounding is missing.
  - Open question: do BPAC / HealthPathways expose APIs/feeds, or is it
    scrape/manual-sync?

## 9. Evaluation & monitoring

- Clinical **eval set** + sign-off thresholds before launch.
- Ongoing monitoring in production.
- A **feedback / incident loop** ("this answer was wrong" → reaches the safety
  officer).

## 10. Failure modes to design for

- **Emergency detection** — deteriorating-patient intent short-circuits to
  "escalate / call now", not an evidence summary.
- **Warm human escalation** — a real handoff to a prescriber/senior, not just
  text. This is the safety net for the non-standing-order gap.
- **Claims unavailable** — fail to lowest privilege + escalate.
- **Cross-domain queries** — e.g. "I'm exhausted and worried I'll make a med
  error" is clinical *and* wellbeing/HR; the router must handle blended intent.
- **Dual-role / role-change** people — same-day claim propagation.

## 11. Phased rollout

1. **Non-clinical floor (KEA) behind the EverythingAI front door.** No PHI, no
   PMS dependency. Proves the front door + router + claims + audit plumbing.
2. **Referral drafting as a reviewable draft.** Value without deep integration;
   output pasted/sent manually.
3. **Migrate the Senior Clinician AI** from Heidikiller as the shared clinical
   core; add junior (framed) access.
4. **Full clinical decision-support skills** — gated behind the regulatory,
   clinical-safety, and data-residency workstream (run in parallel from day one).

## 12. Open questions

- [ ] Heidikiller internal topology — 1 model or a pipeline? Do scribe + advice
      share context today? (needs repo access)
- [ ] Which PMS(es) are clinicians on? (shapes outbound/referral transport)
- [ ] Capture mode — true ambient (mic) or clinician-typed/dictated?
- [ ] BPAC / HealthPathways integration surface — API vs sync vs scrape?
- [ ] Systems of record for credential/scope and care-relationship claims — exist
      and queryable in real time?
- [ ] Medical-device / data-residency determination — who owns it, by when?
