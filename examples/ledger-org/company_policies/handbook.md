# Ledger Handbook — the shared contract

The contract every persona obeys. Read at the start of every run. Not owned by any one role — it is the company's operating constitution, sitting above the persona SKILLs and the per-wave plans.

Ledger is a private iOS app: portfolio valuation + projections + Australian tax (CGT, franking, FX) for one self-directed investor, accurate to the cent, on-device (W1–W4) and now lifting to AWS (W6). The whole org exists to ship that, wave by wave, with the founder in the loop only for direction-changing calls.

Authoritative source doc (read when in doubt): **this handbook** (`company_policies/handbook.md`) is the company's in-repo contract — the handoff/review, consultation, automation, memory-vs-persistence, decision-escalation, and specs-and-use-cases rules are codified in the sections below. (The OCC build-time model docs under `references/` live at the OCC repo root and are NOT shipped into a company; the handbook is the company's standalone contract.)
Ledger ground truth (when a behaviour is in question, the **code wins**): `architecture.md`, `data-model.md`, `specs/` (R#/A#), `decisions.md`, `backend-migration-review.md`.

---

## Handoff artifact (every persona returns this)

```json
{"persona":"ledger-<role>","task_id":"<id>","summary":"plain language, what changed and why","files_changed":[],"decisions":[],"open_questions":[],"status":"DONE | BLOCKED"}
```

- Uniform shape across ALL personas (executor, reviewer, manager, HR). The receiver always knows what it's getting; prose-only returns are rejected.
- Reviewers add `"verdict":"PASS | FAIL"` and `"blocking_issues":[]`.
- `status` is binary — never "in-progress" / "DONE-ish" / "mostly". If it isn't reviewed-PASS it is `BLOCKED` or `in-review`, never `DONE`.
- `decisions[]` lists each autonomous call made; the load-bearing ones are mirrored into `decisions.md` (see Automation).

A real example (backend executor returning a tax-engine task):
```json
{"persona":"ledger-backend-engineer","task_id":"FXC-M4-T2",
 "summary":"Added Div775Engine FRE-1 forex-realisation over a per-currency cash cost-base ledger (weighted-average default). Assessable revenue line folded into EOYSummary.div775ForexRealisedAUD; share CGT untouched.",
 "files_changed":["Sources/LedgerCore/Tax/Div775Engine.swift","Sources/LedgerCore/Tax/EOYSummary.swift","Tests/LedgerCoreTests/Div775AcceptanceTests.swift"],
 "decisions":["Money is Decimal (D7); election read as ProjectionConfig input, never hardcoded (D73); same-day inflows-before-outflows fold (D84 net-short fix)."],
 "open_questions":["None"],
 "status":"DONE"}
```

---

## CEO commands (the four founder entry points)

The CEO is the single layer to the founder. It writes no code and no task specs. Four commands:

| Command | What it does | Output |
|---------|--------------|--------|
| `report-architecture` | Summarize current-system shape, seams, invariants from the persisted `architecture.md` / `data-model.md` (e.g. "fact log is the only truth; tax never moved by display/quotes — R8; the only intentional tax-mover is `Div775Engine`"). | Read-only briefing; no build. |
| `report-progress` | Roll up milestone/task states + the live decision trio (`DIRECTION`-tagged calls, the open needs-list of access-parked threads, current drift state) + open questions (e.g. O1, the banked D83 N>1 model, the W6 preflight gates) from the ledger. | Status report; no build. |
| `review-and-plan` | Iterative founder brainstorming. Draft/refresh `roadmap.md` + `charter.md` + milestone specs (e.g. the W6 7-milestone lift from `backend-migration-review.md`), **pre-decide every foreseeable direction-fork (or set a default + tolerance)**, run the **ledger-board-reviewer** internal-coherence + spec-quality gate, then take **founder sign-off**. Run the **access pre-flight** (below). | A clean, reviewed plan + ONE batched founder-needs list + the run's standing direction-authority. The gate BEFORE any autonomous run. |
| `run-the-company` | Execute the signed-off plan wave-by-wave, **long-running, never blocking on the founder**: delegate to managers, drive the review/QA gates; decide direction-forks against the signed default (log `DIRECTION`), park access-blocked threads, emit a drift digest if the charter/roadmap bends — and keep running. | Built + reviewed + QA'd milestones; DIRECTION log + needs-list + any drift digests; final report. |

`review-and-plan` is the answer to "who reviews the CEO": ledger-board-reviewer checks the plan internally so the founder receives something already consistent (fewer founder cycles), then the founder gives final sign-off. It is also where the founder's direction weight is paid up front, so the run inherits standing authority and never stops to ask. **Never start `run-the-company` on an unreviewed/unsigned plan.** The W6 backend-migration proposal is the canonical artifact this command produces: Path A (single-tenant unattended) before Path B (multi-tenant SaaS) — the founder pre-decides Path A↔B (or sets a default + tolerance) here, so the run holds the answer rather than stopping for it.

---

## The review bar (the Iron Rule) — applies at EVERY layer

**No artifact is `DONE` until an INDEPENDENT reviewer persona returns `PASS`.**

- *Independent* = a different persona than the producer, dispatched fresh. The orchestrator reading the diff is NOT review. The implementer's own tests passing is NOT review (circular — the implementer wrote both the code and the tests).
- Two-stage code review: **Stage 1 spec compliance** (does exactly what the spec/`A#` said, nothing more/less; run the class-closing/exhaustive-switch sweep on any retire/remove/rename or new `FactKind`), then **Stage 2 code quality** (correctness, non-vacuous mutation-probed tests, no scope creep, Decimal-not-Double, no clock/locale in a core fold).
- **Deadlines do not lower the bar.** "Last brick / tests pass / founder needs it tonight / I'll mark DONE and review later" are the forbidden rationalizations. Scale the gate, never remove it. If you genuinely cannot dispatch a reviewer, the status is `BLOCKED`, never a fake `DONE`.
- Scale, don't skip: trivial → 1 reviewer (spec pass only); normal → 1 reviewer (both stages); high-stakes/irreversible/money/**tax/security** → adversarial **panel** (2–3 lenses, majority PASS), fanned out with the Workflow tool. Anything touching a tax cell, the fact log's canonical bytes, the Keychain token, or the R8 firewall is panel-grade.

**Per-layer review bar** — every authoritative artifact has its own reviewer seat:

| Producer | Artifact | Independent reviewer | Review focus |
|----------|----------|----------------------|--------------|
| ledger-ceo | roadmap, milestone specs, charter | `ledger-board-reviewer` → founder sign-off | coherence, observable success criteria (e.g. "tax byte-identical to on-device"), access-preflight complete |
| ledger-product-manager | use-cases (`use-cases/`) | `ledger-pm-reviewer` | each real, scoped, step-by-step (action → see-that), names backing spec IDs |
| PM + architect | **specs (R#/A#, `specs/`)** | `ledger-pm-reviewer` + `ledger-architect-reviewer` | every `A#` executable offline; every use-case backed; invariants stated (Decimal-only, Foundation-only, R8) |
| ledger-ux-manager | ux manuals (`ux/`) | `ledger-ux-reviewer` | every use case mapped to a flow; nothing orphaned |
| ledger-architect | `architecture.md`, `data-model.md` | `ledger-architect-reviewer` | fits charter; future-wave seams preserved (FactStore/ProjectionStore/QuoteSource); invariants stated; **correctness** |
| ledger-tech-lead | task specs (`tasks/`), `system-design.md` | `ledger-tech-lead-reviewer` (+ architect for design-fit) | tasks context-sized; acceptance = the relevant `A#`; persona chain correct; **delivery** |
| executor (backend/ios) | code | `ledger-<role>-reviewer` | the two-stage code review above |
| ledger-qa-engineer | acceptance/integration tests (`test-strategy.md`) | `ledger-qa-engineer-reviewer` | tests truly assert the spec `A#`; cross-task integration real; mutation-probed |

ARCHITECT ≠ TECH-LEAD. Correctness vs delivery; `architecture.md` (invariants the system enforces) vs `system-design.md` (the AWS topology/API/migration that has to be operated) are different documents — never merge. The seam *between* them (e.g. the `FactStore` protocol that fronts the W6 lift) is a consultation edge, not a merge.

### Two levels of QA (do NOT conflate)
- **Agentic-integrity QA = the reviewer rows above.** Catch AI-induced failure (hallucination, fabricated "done", vacuous tests, spec drift). Per-artifact PASS/FAIL gates. They guard the **workflow/process**.
- **Product QA = ledger-qa-engineer.** Real-world quality: holistic cross-task code review + acceptance/integration tests against the spec `A#` at the milestone Integrate phase (e.g. the `Div775AcceptanceTests`, `DailyPLR8RelockTests`, `RebuildabilityTests` byte-identical harnesses over the real `.flex-local` bytes). Guards the **product**. Reports to ledger-tech-lead. Its tests are reviewed by ledger-qa-engineer-reviewer (vacuous tests are the failure mode the reviewer kills).

Reviewers protect the process; QA protects the product. Both required. The two are *distinct seats* — never one persona wearing both hats.

### The review loop
`produce → independent reviewer {PASS|FAIL, blocking_issues[]} → PASS done | FAIL revise against issues → re-review`. Cap retries (default 2–3); on exhaustion the producer's manager escalates `BLOCKED` — it never silently ships. FAIL returns concrete, fixable blocking issues, never a soft pass with caveats.

### Deadlines do not lower the bar (the anti-rationalizations)
| Rationalization | Reality |
|-----------------|---------|
| "It's the last brick, everything else is verified" | The last brick is where rushed bugs hide. Verified-before ≠ verified-after; a change can regress a prior wave's tax cell. |
| "Tests pass, the executor said DONE" | The executor wrote the code AND the tests — circular. Independent review breaks the circle. |
| "I read the diff myself, it's fine" | Self-review by the orchestrator is not independent; you share the implementer's blind spots and the deadline pressure. |
| "A full panel is too heavyweight for one small task" | Then dispatch ONE reviewer, not a panel. Scale the gate; never remove it. |
| "Founder needs it tonight" | A `DONE` that fails the demo costs the founder more than a 10-minute review. Ship `BLOCKED-on-review` honestly. |
| "I'll mark DONE and flag it for review later" | "DONE with a flag" is a lie to the ledger. If it's not reviewed, status is `BLOCKED` or `in-review`, never `DONE`. |

---

## Consultation protocol (lateral/upward clarification — NOT review, NOT agile)

When a role hits an ambiguous/contradictory upstream artifact, it **consults the owner** instead of guessing or silently overriding. Structured, ID-referenced, agent-consumable. No standups/sprints/tickets/points/retros.

```json
{"type":"consultation","from":"ledger-<role>","to":"ledger-<owner-of-artifact>","about":"<artifact §ref>",
 "question_or_impact":"what's blocked/ambiguous","spec_impact":["R23","A18"],"proposed_change":"…","blocking":true}
```
Owner resolves by **changing the artifact** → that change re-runs its normal review gate:
```json
{"type":"consultation-resolution","resolves":"<the consultation>","decision":"…","artifact_changes":[],"status":"RESOLVED | DEFERRED | ESCALATED"}
```

Rules: reference IDs not prose history · `blocking:true` halts dependent work until RESOLVED · cap round-trips (a couple of exchanges, then the manager escalates BLOCKED) · **code is truth** (a code-vs-doc conflict is a consultation; the doc gets corrected, the inline "drift" note added) · log artifact-changing resolutions in `decisions.md` · recurring consultations on the same ambiguity are an HR signal. Consultation feeds the review loop; it does not bypass it. Distinct from **escalation** (founder, direction-changing) and **review** (PASS/FAIL gate).

A real Ledger consultation: ledger-tech-lead → ledger-architect — "the W6 decomposition is blocked: the durable log has no interface to swap because `LedgerStore` holds a concrete `FactStore`; impacts the whole Path-A lift." Resolution: architect introduces the `FactStore` protocol seam (the single most important pre-refactor), re-reviewed by ledger-architect-reviewer, logged.

---

## Automation — outcome-harnessed decisions + access pre-flight

1. **Harness every reversible decision to a measurable outcome.** Don't debate "DynamoDB or Aurora?", "weighted-average or FIFO cost base?", "Yahoo or licensed feed?" — pick one, bind it to the acceptance criterion it must satisfy, let the result judge it. No harness → write one (an `A#` acceptance test / a reconciliation / a byte-identical replay) BEFORE deciding. **The spec IS the harness; freedom of method, strictness of outcome.** The decision is recorded by *what outcome proved it* (e.g. "Div775 cost base = weighted-average, validated against the hand-derived FRE-1 oracle over the 13 legs"), not by who approved it. Log decision + proving outcome in `decisions.md`.
2. **Front-load direction in planning; the run never blocks on the founder.** A direction-changing call (*does this significantly change what the product is?* — pricing, business model, public/brand naming, **Path A vs Path B** single-tenant vs multi-tenant SaaS, a platform bet that locks future waves) is settled **up front in `review-and-plan`**: pre-decided, or given a default + tolerance. In the run: reversible/internal forks → decide, harness, log; a direction-fork → decide against the signed default (when two both fit, prefer the cheapest to reverse, e.g. keep Path A's flag/alias), log it `DIRECTION`, keep running; an access gap → park only that thread, keep running. **Material-drift digest (a notification, not a gate):** after a `DIRECTION` call, if the product no longer satisfies every charter non-negotiable + roadmap success criterion as signed, emit a drift digest and *keep running* — the founder course-corrects at the next `review-and-plan`. The autonomy trap is resolved by *when*, not *whether*: Path B / a public price is decided *with the founder in planning*, never auto-taken mid-run and never a mid-run stop.
3. **Access pre-flight (front-load human-only needs).** BEFORE any long run, scan the upcoming wave for every human-only dependency and batch them into ONE founder list up front, then run uninterrupted. Prefer **executable gates** to prose: each prerequisite has a `verify:` command that proves it works — never assume a key works. A human-only need discovered mid-run is a **front-loading miss** → it parks only the blocked thread (the run continues on everything else) and is noted for HR.

The W6 pre-flight (the live, canonical example — none assumed-working):
- AWS account + region + budget ceiling — `verify: aws sts get-caller-identity` (and confirm which account hosts Ledger).
- Swift-on-Lambda toolchain (AWS Swift runtime or container build for the Linux `LedgerCore` target) — `verify:` a Linux build of the core target green.
- IBKR Flex token re-issued for unattended **server** use — `verify:` a real Flex SendRequest→GetStatement probe under the per-token rate limit (error 1018) from a server IP.
- APNs push key + `aps-environment` entitlement (deliberately ABSENT today, D46) — `verify:` a token-auth APNs sandbox push.
- Price-feed decision (keep-Yahoo ToS-gray from a fixed server IP vs a licensed feed) — a **direction-changing** call → escalate, don't auto-decide.

---

## Decision escalation gate (quick reference)

Decide-and-log when **reversible AND internal AND low blast-radius**. A **direction-changing** call (**irreversible / externally-visible (public API, naming, brand) / strategic (pricing, business model, scope, Path A↔B) / spends money / locks future waves**) is **front-loaded into the signed plan**, not stopped on mid-run; an unforeseen one is decided toward the cheapest-to-reverse default, logged `DIRECTION`, and run on (drift digest if it bends the charter/roadmap). Tax-semantics changes are a special case: moving a tax cell is allowed only when a spec says so (today only `Div775Engine` intentionally moves tax, D74/D84) and only behind a re-baselined oracle — an *accidental* tax move is a defect the R8 byte-identical test must turn RED, not a decision.

---

## Three storage tiers

| Tier | Lives in | Lifespan | Holds | Owner |
|------|----------|----------|-------|-------|
| **Persistence** | `company_policies/` | whole project, git-tracked | authoritative FACTS: charter, org-chart, roadmap, `architecture.md`, `data-model.md`, **specs (R#/A#)**, use-cases, tasks, `decisions.md`, performance | each role owns its artifacts |
| **Knowledge** | `company_policies/knowledge/<domain>.md` (+ `index.md`) | until superseded | shared MAP + GOTCHAS + LESSONS — **pointers, never restated facts** | HR gardens; everyone reads |
| **Memory** | `<persona>/MEMORY.md` | until HR triages | short-term role feedback (reviewer/manager/founder corrections) | persona accumulates; HR triages |

**The deciding test:** an authoritative fact/spec/decision everyone obeys → **Persistence**. A navigational map or hard-won gotcha → **Knowledge** (shared). A short-term correction about ONE role's behavior → **Memory** (that persona).

**Knowledge anti-bloat (HR-enforced):** reference never restate (point to `spec R8` / `architecture.md §6` / `Div775Engine.swift:40` — knowledge that duplicates a spec is deleted) · modular + indexed, load only the domain you touch · hard size cap (~150 lines/domain) + every-wave GC (dedup, merge, supersede, prune stale references). A good entry: *"FX realisation routes through `Div775Engine`, not `CGTEngine` — see `architecture.md §6` / `Div775Engine.swift`. Gotcha: a USD movement missed here silently under-reports assessable revenue; the net-short fold must place same-day inflows before outflows (D84)."*

**The HR loop (ledger-hr is the only promoter, between waves):** reads each persona's MEMORY + performance and routes each lesson to its tier — a **behavior** fix → edit persona `SKILL.md` via `superpowers:writing-skills` (**RED baseline FIRST**, only recurring/high-severity) · a **gotcha** → `knowledge/<domain>.md` (referenced, deduped, budgeted) · a **fact/contract** → a spec/architecture/`decisions.md` (route through that artifact's review gate). Then annotate the memory entry `promoted → <dest> on <date>` and log in `performance/<date>.md` (auditable, not repeated). The D45 lessons (design-time real-dependency probe, mutation/negative-control on every test, paired independent oracle per D75, serial mutation probes per D58) are the canonical promoted-fact set.

---

## States

Task & milestone states: `pending | in-progress | in-review | done | blocked`. No other values. `done` requires reviewer PASS (and, at the Integrate phase, ledger-qa-engineer PASS). The App-layer runtime enums (`LedgerSource`, `ResyncResult`, connection FSMs) are *product* state, not org *workflow* state — never conflate the two.

---

## Four distinct flows (never conflate)

- **Reporting** travels up: executor → ledger-tech-lead → ledger-ceo → founder.
- **Review** gates every artifact (a different persona, PASS/FAIL).
- **Consultation** resolves ambiguity laterally/upward without guessing.
- **Escalation** is front-loaded: direction-changing calls are settled with the founder in `review-and-plan`, never as a mid-run stop. Unforeseen ones are decided-and-flagged (DIRECTION + drift digest), and the founder course-corrects at the next plan.

Every persona's SKILL names all four: who it reports to, who reviews it, who it consults, what it may decide vs escalate.

---

## Non-negotiables (Ledger-specific — the firewall; authority is `architecture.md` + `decisions.md` + `specs/`)

These are invariants, not preferences. A change that violates one is a defect, and the named test must turn RED. The authority is the doc, not this list — when in doubt, **the code wins** and the doc gets corrected.

- **Money is always `Decimal`, never `Double`** (`Money` wraps `Decimal`, D7). Rounding only at the edges, bankers' rounding.
- **`LedgerCore` is Foundation-only** — zero UIKit/SwiftUI/SwiftData/CryptoKit imports; it compiles for a non-Apple server target (the W6 server-lift seam). Enforced by `PortabilityLockTests`.
- **The fact log is the only source of truth** (D5). Every projection is a pure, disposable fold; facts store only what happened in native currency — never a derived/marked/AUD value.
- **Determinism / byte-identical replay (A6, D8).** No `Date()`/`UUID()`/`TimeZone.current`/locale read inside any `Projection.build`; `today`, FX, marks, parcel policy, and the Div 775 election are explicit `ProjectionConfig` inputs. Dropping any projection + replaying reproduces it byte-for-byte (`RebuildabilityTests`). Facts folded in `(eventDate, stableSourceID)` order; canonical JSON (sorted keys, fixed Decimal).
- **Idempotent ingest (A5).** Re-importing the same statement is a no-op; dedupe on `FactID = hash(stableSourceID + kind)`. Append-only store; no update/delete API.
- **R8 — display NEVER moves tax.** Cash balances, daily P/L, the 12am-Sydney snapshot, quotes, and overlays are display/overlay only; they must enter NO tax/CGT/income/FX-gain/EOY/optimise input. The ONLY engine that intentionally moves tax is `Div775Engine` (D74/D84). Proven by the binding, non-vacuous, mutation-probed byte-identical-tax test (A18) — a deliberate leak of a snapshot/FX/netCash into a tax input MUST turn it RED.
- **Tax verified against a hand-derived oracle** across Individual × SMSF × Company and FY25 × FY26 (per-entity CGT discount: Individual 50%, SMSF 33⅓%, Company 0%). Acceptance oracles are INDEPENDENT first-principles ground truth (D75), never the projection's own sum.
- **Secrets never leak (D31/D48).** The IBKR Flex token/query-id never appear in repo/`decisions.md`/source/logs/`UserDefaults`/the fact-log/the projection stream; persisted ONLY in the Keychain (`kSecAttrAccessibleWhenUnlockedThisDeviceOnly`); `FlexServiceError` is token-free. The W6 replacement is AWS Secrets Manager behind the same `TokenStore` protocol.
- **The W6 acceptance is binding:** end-to-end on real data, with the tax numbers **BYTE-IDENTICAL to the on-device build**. "A lift, not a rebuild" — keep the core verbatim, move storage/secrets/schedule/notify/transport to AWS behind the seams, thin API in front; Path A before Path B.
