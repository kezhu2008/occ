# Corporate Structure — the OCC org model

OCC reuses modern corporate structure because it is a battle-tested answer to the same problem we have: **how does one owner get a large amount of trustworthy work done without being in every decision?** Answer: layered delegation, an **independent review gate at every layer**, **lateral consultation** so roles resolve ambiguity instead of guessing, and a **feedback loop** (HR) that makes the org sharper over time.

## The layers

```
                          ┌─────────────┐
                          │   FOUNDER   │  (the human owner)
                          └──────┬──────┘
                       sign-off ▲│ escalations (direction-changing only)
                          ┌──────┴──────┐        reviews CEO's plans
                          │     CEO     │◄──────► board-reviewer
                          └──────┬──────┘
        ┌──────────────┬────────┼────────┬──────────────┐
   ┌────▼────┐   ┌─────▼────┐ ┌─▼──────┐ ┌────▼────┐
   │ Product │◄─►│   UX     │ │Architect│◄►│  Tech   │   mid-level managers
   │ Manager │   │ Manager  │ │ design  │ │  Lead   │   (◄► = consultation edges)
   └────┬────┘   └────┬─────┘ └─┬──────┘ └────┬────┘   each + a peer reviewer
        │ use-cases   │ ux       │ arch +      │ tasks + delivery +
        │  (+co-own   │          │ data-model  │ system-design + migration
        │   specs) ───┼──────────┤ + co-own    │  (Workflow tool)
        └─────────────┴────specs─┴─────┬───────┘
                              ┌─────────▼─────────┐   ┌──────────┐
                              │     executors     │◄─►│    QA     │
                              │  each + reviewer  │   │ engineer  │  product-level
                              └───────────────────┘   └──────────┘  acceptance+integration
   ┌─────┐  HR: promotes persona MEMORY → SKILL.md (behavior) and → knowledge/ (gotchas).
   │ HR  │  Gardens the shared knowledge tier. OCC = structural; HR = incremental.
   └─────┘
```

**Every producing box has an independent reviewer** (the review spine). **Consultation edges (◄►)** let roles push feedback laterally/upward — a tech-lead tells the architect a decomposition is blocked; a dev asks the tech-lead about an ambiguous spec — see `consultation.md`.

### CEO — strategic layer (one)
Owns vision execution + priorities; the single layer to the founder. Owns `roadmap.md`, milestone specs, `decisions.md`; co-owns `charter.md`. **Runs the automation pre-flight** (access audit, frontload founder-only needs) and escalates only **direction-changing** calls — everything else is decided and judged by outcome (`automation.md`). Commands: `report-architecture`, `report-progress`, `run-the-company`. Reviewed by **board-reviewer** (internal coherence/spec-quality) then founder sign-off. Writes no code or task specs.

### Mid-level managers — four distinct seats (do NOT merge Architect and Tech Lead)
- **Product Manager** — owns `use-cases/` (step-by-step, user-facing). **Co-owns `specs/`** (the demand side). Reviewed by `pm-reviewer`.
- **UX Manager** — maps UI/design to use-cases; owns `ux/`. Reviewed by `ux-reviewer`.
- **Architect** — owns **correctness**: `architecture.md` (current-system shape, seams, invariants) + `data-model.md`, and **co-owns `specs/`** (the behavioural contract R#/A# + invariants). Answers *"is it right?"* Designs; does not decompose tasks or write code. Reviewed by `architect-reviewer`.
- **Tech Lead** — owns **delivery**: turns the spec + architect's design into `tasks/<id>.md`, owns the **deployment/next-phase design** (`system-design.md`), API contracts and migration plan, and runs the milestone-delivery Workflow. Answers *"can we build & run it?"* Reviewed by `tech-lead-reviewer`. Writes no product code.

Boundary (from the real Ledger docs, which keep `architecture.md` and `system-design.md` as different documents): Architect = invariants enforced by the system itself; Tech Lead = topology/API/migration that has to be operated. The seam *between* them (e.g. a `FactStore` protocol) is exactly a consultation edge.

### Executors — the builders
backend-engineer, ios-engineer, infra-engineer, etc. Implement one task against its spec, TDD-first (the test is the outcome-harness for their method choice). Each paired with an independent `<role>-reviewer`.

### QA Engineer — real-world product quality (the second level of QA)
There are **two levels of QA**, and they are different jobs:
- **Agentic-integrity QA = the reviewer personas** (board/pm/ux/architect/tech-lead/code reviewers). Their job is to catch **AI-induced failure** — hallucination, fabricated "done", vacuous tests, spec drift — and keep the *agentic workflow* honest. Per-artifact gates.
- **Product QA = the QA Engineer persona.** A real-world QA: performs holistic **code review across tasks** and writes/runs **acceptance + integration tests against the spec's `A#`** at the milestone Integrate phase. Its job is "does the actual software work for the user?", not "did the agent behave." Reports to the Tech Lead; its test results are the product-level outcome-harness.

Keep both: reviewers protect the *process*; QA protects the *product*.

### HR — the training function (special, internal)
Not in the reporting chain. Between waves, reads MEMORY + performance and **promotes to the right tier**: behavior → SKILL.md (writing-skills, RED baseline), gotcha → `knowledge/` (referenced, deduped, budgeted), fact → a spec/architecture. **Gardens the shared knowledge tier** (GC, supersede, cap). HR = incremental within-role; OCC strategic-review = structural cross-role.

## Reporting, review, consultation, escalation — four distinct flows
- **Reporting** travels up: executor → tech-lead → CEO → founder.
- **Review** gates every artifact (a different persona, PASS/FAIL) — `handoff-and-review.md`.
- **Consultation** resolves ambiguity laterally/upward without guessing — `consultation.md`.
- **Escalation** reaches the founder only for direction-changing calls — `automation.md` / `decision-escalation.md`.

Every persona's job description names all four: who it reports to, who reviews it, who it consults, what it may decide vs escalate.
