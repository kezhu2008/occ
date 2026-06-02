# Ledger Org — a worked OCC example

This directory is a **One Consultancy (OCC) output**: the corporate-structure artifacts OCC generates when you point it at an existing codebase. It was produced for the real **Ledger** repo at `~/git/ledger` — a private iOS app that turns one self-directed investor's Interactive Brokers activity into portfolio valuations, projections, and correct Australian tax (CGT, franking, FX/Division 775), accurate to the cent, deterministic by replay, on-device through W1–W4 and lifting to an AWS backend in W6.

**It was generated *without touching the real repo.*** Nothing under `~/git/ledger` was created, edited, or moved. OCC read Ledger's own design docs (`architecture.md`, `data-model.md`, `system-design.md`, `requirements.md`, `use_cases.md` + `use-cases/`, `backend-migration-review.md`, `waves.md`, `decisions.md`) and *mirrored their real shape* into a standalone org under this `examples/` tree. Every fact here — the Foundation-only `LedgerCore` seam, `Money = Decimal` (D7), the byte-identical replay property (A6), the R8 tax firewall, the W6 "lift not rebuild" plan — is grounded in those docs. This is a reference example you can read end-to-end, not a live company directory wired into `~/git/ledger`.

---

## What an OCC output is

OCC reuses modern corporate structure because it is a battle-tested answer to the same problem a solo owner has: **how does one owner get a large amount of trustworthy work done without being in every decision?** The answer OCC encodes is layered delegation, an **independent review gate at every layer**, **lateral consultation** so roles resolve ambiguity instead of guessing, and an **HR feedback loop** that sharpens the org over time.

An OCC output for a repo is two things side by side:

```
ledger-org/
├── company_policies/        # PERSISTENCE — the authoritative, founder-facing, versioned facts
└── claude-skills/           # the ORG — the invokable persona skills (ledger-*), each with SKILL.md + MEMORY.md
```

- **`company_policies/`** holds the strategic and meta artifacts — charter, org-chart, roadmap, milestones, the co-owned `specs/` (R#/A#), step-by-step `use-cases/`, `tasks/`, `architecture.md`/`data-model.md`/`system-design.md`, `backend-migration-review.md`, the shared `knowledge/` tier, and `ux/`. These are the facts everyone obeys.
- **`claude-skills/`** holds the recruited org: one `ledger-<role>/` skill per seat (e.g. `ledger-ceo/`, `ledger-architect/`, `ledger-backend-engineer/`), each a `SKILL.md` (persistent role instructions) plus a co-located `MEMORY.md` (short-term, HR-triaged role feedback). The registry of which skills exist and who reviews whom is `company_policies/org-chart.md`. `ledger-hr` is the only function that recruits and edits these.

This example ships the full `company_policies/` set; the `claude-skills/` personas are the org that `org-chart.md` defines and HR recruits against it. Every row in `org-chart.md` names exactly one `claude-skills/<skill>/SKILL.md` + `MEMORY.md`.

---

## The org (from `company_policies/org-chart.md`)

Layered delegation with an independent reviewer on **every producing seat**. `◄►` = a consultation edge.

```
                          FOUNDER  (the human owner, user-zero — runs their real IBKR portfolio on it)
                       sign-off ▲│ escalations (direction-changing only)
                          ledger-ceo ◄──────► ledger-board-reviewer   (reviews the CEO's plans)
        ┌──────────────┬────────┼────────┬──────────────┐
   ledger-product-mgr ◄► ledger-ux-mgr  ledger-architect ◄► ledger-tech-lead
        │ use-cases/        │ ux/         │ architecture.md   │ tasks/ + system-design.md
        │ (+co-own specs/)  │             │ + data-model.md   │ + W6 migration plan
        │                   │             │ (+co-own specs/)  │ runs the milestone Workflow
        └───────────────────┴──── specs/ ─┴────────┬──────────┘
                              ledger-backend-engineer / ledger-ios-engineer  ◄►  ledger-qa-engineer
                              (each + a -reviewer)                                (product QA — 2nd level)
   ledger-hr (special, not in the chain): promotes MEMORY → SKILL.md (behavior) / → knowledge/ (gotcha) / → a spec (fact); gardens knowledge/.
```

**Architect ≠ Tech Lead — never merge them.** This is the load-bearing distinction OCC carries straight from Ledger's own docs, which keep `architecture.md` and `system-design.md` as two different files:
- **`ledger-architect`** owns **correctness** — `architecture.md` (current-system shape, seams, invariants the system *enforces*) + `data-model.md`, and co-owns `specs/`. Answers *"is it right?"*
- **`ledger-tech-lead`** owns **delivery** — `tasks/`, `system-design.md` (the W6 AWS topology, API contracts, Path A→B migration that has to be *operated*), and runs the milestone-delivery Workflow. Answers *"can we build & run it?"*
- The seam *between* them — the W6 `FactStore` protocol the architect designs and the tech-lead operates against DynamoDB — is a consultation edge, not a merge.

**Two levels of QA, both kept:** the **reviewer personas** (board / pm / ux / architect / tech-lead / code reviewers) are *agentic-integrity QA* — they catch AI-induced failure (hallucination, fabricated "done", vacuous tests, spec drift) per artifact. `ledger-qa-engineer` is *product QA* — holistic cross-task code review + acceptance/integration tests against the spec `A#` (A2 tax oracle, A6 rebuildability, R8 byte-identical-tax re-lock) at the milestone Integrate phase. Reviewers protect the *process*; QA protects the *product*. QA is the one producer whose output (tests) is reviewed by the code-reviewer for non-vacuousness.

---

## Read order

1. **`company_policies/charter.md`** — the business vision only (why Ledger exists, who user-zero is, success = the founder filing on it; tax-to-the-cent and same-inputs-same-answer as non-negotiables). No technical design here.
2. **`company_policies/org-chart.md`** — the persona registry: every role, its reviewer, what it owns, its stack adapters. The single source of truth for the org.
3. **`company_policies/handbook.md`** — the shared contract every persona obeys: the uniform handoff artifact, the CEO's four commands, the per-layer review bar (the Iron Rule), the escalation gate.
4. **`company_policies/roadmap.md`** — W1 (done) → W2 (in progress) → W3/W4/W5 → **W6 backend lift**, each with an *observable* success criterion, plus the locked decisions (Decimal-only money, event-sourced/append-only, the named protocol seams, "lift not rebuild", Path A before Path B).
5. **`company_policies/use-cases/`** — the *demand* side: step-by-step "action → see-that" manuals (`onboarding-and-sync.md`, `positions-and-holdings.md`, `tax.md`) and `index.md` (the canonical UC catalog) linking each use-case to its backing `R#/A#`. Genuinely-future areas (watchlist/alerts, the W6 backend backlog) carry `(planned — Wn)` in `index.md` with no manual until one ships.
6. **`company_policies/specs/`** — the *contract*: `domain-core.md` and `backend-lift.md`, the R#/A# behavioural specs with offline-testable acceptance. Co-owned by PM (demand) + architect (invariants).
7. **`company_policies/architecture.md`** + **`data-model.md`** — architect-owned correctness: the Foundation-only core, the Fact/FactKind/projection schema, CanonicalJSON serialization, the invariants (R8 firewall, A6 replay, D7/D8).
8. **`company_policies/system-design.md`** + **`backend-migration-review.md`** — tech-lead-owned delivery: the W6 AWS deploy topology and the Path A vs Path B / 7-milestone migration proposal.
9. **`company_policies/milestones/`** (`W2.md`, `W6.md`) and **`tasks/`** (`W2-T1/T2/T3`, `W6-M1`) — how a milestone is specced and decomposed; **`ux/positions.md`** for a UX manual; **`knowledge/`** (`index.md` + the per-domain files `ledgercore.md`, `tax.md`, `ingest.md`, `ui.md`) for the shared gotcha tier.

Then read any `claude-skills/ledger-<role>/SKILL.md` to see how a single seat is recruited from all of the above.

> **Note — `references/` and `templates/` are NOT shipped into a company.** OCC's model docs (`references/`) and artifact scaffolds (`templates/`) are **build-time** documents that live at the **OCC repo root**, not under this `ledger-org/` tree. They are what OCC reads while generating an org; a generated company's standalone in-repo contract is **`company_policies/handbook.md`** (which codifies the handoff/review, consultation, automation, memory-vs-persistence, and escalation rules a persona needs at runtime). Do not expect a `references/` or `templates/` directory inside a shipped company.

---

## What's new in v3

This example is OCC v3. The deltas from v2:

- **Specs + step-by-step use-cases, joined by IDs (the implement-and-test-against chain).** Use-cases (`use-cases/`, PM-owned) are *what the user does*, step-by-step "action → see-that". Specs (`specs/`, **co-owned by PM + architect**) are *what the system must do* — the `R#/A#` behavioural contract with offline-testable acceptance. A use-case names its backing `R#/A#`; tests are written against that `A#`; QA asserts the use-case's "see-that" against it. A use-case with no spec, or an `A#` nothing uses, is a gap the reviewers FAIL. This mirrors how the real Ledger team actually works (`use_cases.md` + `use-cases/` ↔ `requirements.md`).
- **The knowledge tier — a shared, deliberately-small map of gotchas.** `company_policies/knowledge/` is the third storage tier between persistence (facts everyone obeys) and per-persona MEMORY (short-term role feedback). It holds only the *map* and the *gotcha* and **references** the authoritative artifact (`spec R8`, `architecture.md §6`, `Div775Engine.swift:40`) — never restates a fact. Per-domain files (`ledgercore`, `tax`, `ingest`, `ui`) under a budget, indexed by `index.md`, GC'd every wave by HR.
- **A real QA engineer (`ledger-qa-engineer`) — the second level of QA.** Distinct from the reviewer personas. It owns `test-strategy.md` and writes/runs acceptance + integration tests against the spec `A#` at the milestone Integrate phase, plus holistic cross-task code review. Its tests are themselves reviewed (by the code-reviewer) for non-vacuousness.
- **The CEO has four commands, including `review-and-plan`.** `report-architecture` (read-only seam/invariant briefing), `report-progress` (milestone/task/escalation roll-up), `run-the-company` (execute the signed-off plan wave-by-wave), and the new **`review-and-plan`** — iterative founder brainstorming that drafts/refreshes `roadmap.md` + `charter.md` + milestone specs, runs the `ledger-board-reviewer` internal-coherence + spec-quality gate, then takes founder sign-off and runs the access pre-flight. It is the answer to *"who reviews the CEO"* and the gate that must clear before any autonomous run.
- **W6 — the backend lift, fully specced.** The roadmap, `specs/backend-lift.md`, `system-design.md`, and `backend-migration-review.md` carry the W6 plan: move `LedgerCore` *verbatim* into AWS Lambda behind a `FactStore` protocol, a multi-tenant DynamoDB fact store (`tenant#connection`/`factId`, idempotent conditional put), Secrets Manager tokens, EventBridge scheduled + triggered sync, a thin read API + Cognito, and SNS→APNs push via the `FiredAlert` seam. **"A lift, not a rebuild"** — keep the core, move storage/secrets/schedule/notify/transport to AWS behind the existing seams, put a thin API in front. **Path A (single-tenant unattended sync) before Path B (multi-tenant SaaS)**, the staging surfaced to the founder rather than auto-taken. Acceptance: end-to-end on real data with **tax numbers byte-identical to the on-device build** — the same hand-derived oracle (IND × SMSF, FY25 × FY26) that gated W1 gates the lift.

---

## How this example was produced

OCC ran against `~/git/ledger` read-only. It read Ledger's authoritative docs to learn the product's true shape and constraints, then generated this org as a fresh, self-contained tree under `examples/ledger-org/` — **never writing into the source repo.** The OCC references that define the model it instantiated (the corporate structure, the handoff/review gates, the specs↔use-cases chain, the three storage tiers, outcome-harnessed automation, consultation, escalation, and the dynamic milestone workflow) live under `references/` at the repo root. The generation itself used OCC's own dynamic-workflow pattern: fan out exploration of the source docs, draft each artifact, and run each generated artifact through an independent adversarial review before it was accepted — the same produce → review → revise → converge loop the recruited org uses to deliver a milestone.
