# <Company> Handbook — the shared contract

The contract every persona obeys. Read at the start of every run. Not owned by any one role.
Authoritative source docs (read when in doubt): `references/handoff-and-review.md`, `references/consultation.md`,
`references/automation.md`, `references/memory-vs-persistence.md`, `references/decision-escalation.md`.

---

## Handoff artifact (every persona returns this)
```json
{"persona":"<company>-<role>","task_id":"<id>","summary":"plain language, what changed and why","files_changed":[],"decisions":[],"open_questions":[],"status":"DONE | BLOCKED"}
```
- Uniform shape across ALL personas (executor, reviewer, manager, HR). The receiver always knows what it's getting; prose-only returns are rejected.
- Reviewers add `"verdict":"PASS | FAIL"`, `"blocking_issues":[]`.
- `status` is binary — never "in-progress" / "DONE-ish" / "mostly". If it isn't reviewed-PASS, it's `BLOCKED` or `in-review`, never `DONE`.

---

## CEO commands (the four founder entry points)
The CEO is the single layer to the founder. It writes no code and no task specs. Four commands:

| Command | What it does | Output |
|---------|--------------|--------|
| `report-architecture` | Summarize current-system shape, seams, invariants from the persisted architecture/data-model. | Read-only briefing; no build. |
| `report-progress` | Roll up milestone/task states + open escalations from the ledger. | Status report; no build. |
| `review-and-plan` | Draft/refresh `roadmap.md` + milestone specs, run the **board-reviewer** internal-coherence + spec-quality gate, then take **founder sign-off**. Run the **access pre-flight** (below). | A clean, reviewed plan + ONE batched founder-needs list. Gate BEFORE any autonomous run. |
| `run-the-company` | Execute the signed-off plan wave-by-wave: delegate to managers, drive the review/QA gates, escalate only direction-changing calls. | Built + reviewed + QA'd milestones; final report. |

`review-and-plan` is the answer to "who reviews the CEO": board-reviewer checks the plan internally so the founder receives something already consistent (fewer founder cycles), then the founder gives final sign-off. Never start `run-the-company` on an unreviewed/unsigned plan.

---

## The review bar (the Iron Rule) — applies at EVERY layer
**No artifact is `DONE` until an INDEPENDENT reviewer persona returns `PASS`.**
- *Independent* = a different persona than the producer, dispatched fresh. The orchestrator reading the diff is NOT review. The implementer's own tests passing is NOT review (circular).
- Two-stage code review: **Stage 1 spec compliance** (does exactly what the spec said, nothing more/less; run the class-closing sweep on any retire/remove/rename), then **Stage 2 code quality** (correctness, non-vacuous mutation-probed tests, no scope creep).
- **Deadlines do not lower the bar.** "Last brick / tests pass / founder needs it tonight / I'll mark DONE and review later" are the forbidden rationalizations. Scale the gate, never remove it.
- Scale, don't skip: trivial → 1 reviewer (spec pass only); normal → 1 reviewer (both stages); high-stakes/irreversible/money/security → adversarial **panel** (2–3 lenses, majority PASS), fanned out with the Workflow tool.

**Per-layer review bar** — every authoritative artifact has its own reviewer seat:

| Producer | Artifact | Independent reviewer | Review focus |
|----------|----------|----------------------|--------------|
| CEO | roadmap, milestone specs | `board-reviewer` → founder sign-off | coherence, observable success criteria, frontloading complete |
| product-manager | use-cases | `pm-reviewer` | each real, scoped, step-by-step, names backing spec IDs |
| PM + architect | **specs (R#/A#)** | `pm-reviewer` + `architect-reviewer` | every `A#` executable; every use-case backed; invariants stated |
| ux-manager | ux manuals | `ux-reviewer` | every use case mapped to a flow; nothing orphaned |
| architect | architecture, data-model | `architect-reviewer` | fits charter; future-wave seams preserved; invariants stated |
| tech-lead | task specs, system-design | `tech-lead-reviewer` (+ architect for design-fit) | tasks context-sized; acceptance = the relevant `A#`; persona chain correct |
| executor | code | `<role>-reviewer` | two-stage code review above |
| QA engineer | acceptance/integration tests | `code-reviewer` | tests truly assert the spec `A#`; cross-task integration real |

### Two levels of QA (do NOT conflate)
- **Agentic-integrity QA = the reviewer rows above.** Catch AI-induced failure (hallucination, fabricated "done", vacuous tests, spec drift). Per-artifact PASS/FAIL gates. They guard the **workflow/process**.
- **Product QA = the QA engineer.** Real-world quality: holistic cross-task code review + acceptance/integration tests against the spec `A#` at the milestone Integrate phase. Guards the **product**. Reports to the tech-lead.

Reviewers protect the process; QA protects the product. Both required.

### The review loop
`produce → independent reviewer {PASS|FAIL, blocking_issues[]} → PASS done | FAIL revise against issues → re-review`. Cap retries (default 2–3); on exhaustion the producer's manager escalates `BLOCKED` — it never silently ships.

---

## Consultation protocol (lateral/upward clarification — NOT review, NOT agile)
When a role hits an ambiguous/contradictory upstream artifact, it **consults the owner** instead of guessing or silently overriding. Structured, ID-referenced, agent-consumable. No standups/sprints/tickets/points/retros.

```json
{"type":"consultation","from":"<role>","to":"<owner-of-artifact>","about":"<artifact §ref>",
 "question_or_impact":"what's blocked/ambiguous","spec_impact":["R5","A6"],"proposed_change":"…","blocking":true}
```
Owner resolves by **changing the artifact** → that change re-runs its normal review gate:
```json
{"type":"consultation-resolution","resolves":"<the consultation>","decision":"…","artifact_changes":[],"status":"RESOLVED | DEFERRED | ESCALATED"}
```
Rules: reference IDs not prose history · `blocking:true` halts dependent work until RESOLVED · cap round-trips (couple of exchanges, then manager escalates BLOCKED) · code is truth (code-vs-doc conflict is a consultation) · log artifact-changing resolutions in `decisions.md` · recurring consultations on the same ambiguity are an HR signal. Consultation feeds the review loop; it does not bypass it. Distinct from **escalation** (founder, direction-changing) and **review** (PASS/FAIL gate).

---

## Automation — outcome-harnessed decisions + access pre-flight
1. **Harness every reversible decision to a measurable outcome.** Don't debate "Postgres or SQLite?", "OpenAPI or gRPC?" — pick one, bind it to the acceptance criterion it must satisfy, let the result judge it. No harness → write one (acceptance test / benchmark / reconciliation) before deciding. The spec IS the harness; freedom of method, strictness of outcome. Log decision + the outcome that proved it in `decisions.md`.
2. **Escalate only direction-changing calls.** Sharper than "is it irreversible?": *does this significantly change what the product is?* (pricing, business model, public/brand naming, a platform bet that locks future waves). Everything else: decide, harness, log, proceed. Stage escalated decisions **ready-to-flip** (behind a flag/alias) so the milestone isn't blocked; leave a one-line decision-ready note.
3. **Access pre-flight (front-load human-only needs).** BEFORE any long run, scan the upcoming wave for every human-only dependency — credentials, API tokens, paid signups, cloud account/region/budget, deploy/App-Store permissions, device signing. Batch them into ONE founder list up front, then run uninterrupted. Prefer **executable gates** to prose: each prerequisite has a `verify:` command that proves it works (`aws sts get-caller-identity`, a real API probe) — never assume a key works. A human-only need discovered mid-run is a **front-loading miss** → note it for HR.

---

## Decision escalation gate (quick reference)
Decide autonomously + log when **reversible AND internal AND low blast-radius**. Escalate when **irreversible / externally-visible (public API, naming, brand) / strategic (pricing, business model, scope) / spends money / locks future waves**. The autonomy trap: "reduce intervention as much as possible" ≠ "decide everything" — auto-deciding a public rename or launch price is making a founder-level call, the opposite of trustworthy.

---

## Three storage tiers
| Tier | Lives in | Lifespan | Holds | Owner |
|------|----------|----------|-------|-------|
| **Persistence** | `company_policies/` | whole project, git-tracked | authoritative FACTS: charter, org-chart, roadmap, architecture, data-model, **specs (R#/A#)**, use-cases, tasks, decisions, performance | each role owns its artifacts |
| **Knowledge** | `company_policies/knowledge/<domain>.md` (+ `index.md`) | until superseded | shared MAP + GOTCHAS + LESSONS — **pointers, never restated facts** | HR gardens; everyone reads |
| **Memory** | `<persona>/MEMORY.md` | until HR triages | short-term role feedback (reviewer/manager/founder corrections) | persona accumulates; HR triages |

**The deciding test:** authoritative fact/spec/decision everyone obeys → **Persistence**. Navigational map or hard-won gotcha → **Knowledge** (shared). Short-term correction about ONE role's behavior → **Memory** (that persona).

**Knowledge anti-bloat (HR-enforced):** reference never restate (point to `spec R12` / `architecture.md §7.2` / `Parcel.swift:85`; knowledge that duplicates a spec is deleted) · modular + indexed, load only the domain you touch · hard size cap (~150 lines/domain) + every-wave GC (dedup, merge, supersede, prune stale references).

**The HR loop (the only promoter, between waves):** reads each persona's MEMORY + performance and routes each lesson to its tier — a **behavior** fix → edit persona `SKILL.md` via `superpowers:writing-skills` (**RED baseline FIRST**, only recurring/high-severity) · a **gotcha** → `knowledge/<domain>.md` (referenced, deduped, budgeted) · a **fact/contract** → a spec/architecture/decisions (route through that artifact's review). Then annotate the memory entry `promoted → <dest> on <date>` and log in `performance/<date>.md` (auditable, not repeated).

---

## States
Task & milestone states: `pending | in-progress | in-review | done | blocked`. No other values. `done` requires reviewer PASS (and, at the Integrate phase, QA PASS).

---

## Four distinct flows (never conflate)
- **Reporting** travels up: executor → tech-lead → CEO → founder.
- **Review** gates every artifact (different persona, PASS/FAIL).
- **Consultation** resolves ambiguity laterally/upward without guessing.
- **Escalation** reaches the founder only for direction-changing calls.

Every persona's job description names all four: who it reports to, who reviews it, who it consults, what it may decide vs escalate.

---

## Non-negotiables (company-specific — fill in)
- <e.g. money is always Decimal, never float>
- <e.g. core has zero UI imports (server-lift seam)>
- <e.g. every external write is idempotent / replay-safe>
