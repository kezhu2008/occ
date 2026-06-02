# Specs & Use-Cases — the implement-and-test-against chain

A developer (UI or backend) must be able to **implement and write tests against** a written artifact, with no guessing. That requires two linked artifacts, mirroring how the real Ledger team actually works:

1. **Use-case** — *what the user does*, step-by-step, UI/behaviour-facing.
2. **Backend specification** — *what the system must do*, the behavioural contract with offline-testable acceptance.

Use-cases and specs are **separate documents joined by IDs**. A use-case is the demand; the spec is the contract that satisfies it; the tests are written against the spec's acceptance.

## Ownership

- **Use-cases: Product Manager owns** (`use-cases/`). What/why, the user-facing flow + acceptance in user terms.
- **Backend spec: Product Manager + Architect CO-OWN** (`specs/`). PM frames the required behaviour from the use-case; Architect makes it a precise, invariant-bearing, testable contract. pm-reviewer + architect-reviewer both gate it.
- **Tests: written against the spec's `A#` acceptance** — by the developer (TDD unit) and by QA (acceptance/integration). Reviewers mutation-probe them.

## Use-case format (step-by-step — mirror the real Ledger manuals)

Two tiers. An **index** (`use-cases/index.md`) and per-area **manuals**.

Index row: `| ID | Use case | Entry point | Manual | Spec | Source |` — ID (e.g. `UC-RECONCILE`), one-line title, entry path, link to the manual section, **link to the backing spec IDs**, and source `file:line` once implemented.

Each manual entry is concrete enough to build and test from:
```md
## UC-RECONCILE
**Confirm the ledger ties out to the broker before trusting any report**
- **Goal:** <one sentence — what the user achieves>
- **Actor / preconditions:** <who, and what must be true first>
- **Entry point:** <the exact tap/call path to reach it>
- **Steps (action → see-that):**
  1. <do this> → <observe exactly this — real labels/values/states>
  2. ...
- **Outcome:** <the system state after success — persistence, flags, data visible>
- **Variations & edge states:** <error paths, boundaries, empty/loading/failure>
- **Backed by spec:** R12, R13, A7, A8   ← the contract the steps rely on
- **Source:** <file:line once implemented; "(to be implemented)" while proposed>
```
For a not-yet-built feature, steps are *intended flows* (no exact labels yet) but the **spec + acceptance are still binding**.

## Backend specification format (R# / A# — the testable contract)

Organized by subsystem, not by use-case (one spec item backs many use-cases).
```md
## <Subsystem> spec
R12. <requirement — a behaviour/invariant the system MUST hold, architecture-agnostic>.
R13. <requirement ...>.

### Acceptance (offline-testable; this is what tests assert)
A7. <a provable fact, with fixtures if needed> — e.g. "drop any projection + replay ⇒ byte-identical (test)".
A8. <a provable fact> — e.g. "reconciled cash == statement closing balance by Decimal exact equality; a 1¢ delta FAILs".
```
Rules:
- Every `A#` is **executable** — a test can assert it without a human judging. (Outcome-harnessed; see `automation.md`.)
- Every use-case names the `R#`/`A#` it is backed by; every `A#` should trace to at least one use-case or invariant. A use-case with no backing spec, or a spec item nothing uses, is a gap the reviewers FAIL.
- The spec is where **technical invariants** live (Decimal-only, Foundation-only, determinism) — NOT the charter (business vision) and NOT the use-case (user flow).

## The chain (who produces what, what tests what)

```
PM use-case (what user does, step-by-step)
        │ names
        ▼
PM+Architect spec  R#/A#  (behavioural contract, executable acceptance)
        │ decomposed by tech-lead into
        ▼
tasks/<id>.md  (acceptance = the relevant A#)
        │ implemented by developer, TDD against A#  ──► code-reviewer (non-vacuous, spec-compliance)
        ▼
QA: acceptance + integration tests assert the A# across tasks  ──► milestone Integrate gate
```

The outcome of every layer is judged by the spec's acceptance — that is what makes the whole thing automatable.
