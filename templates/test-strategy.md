# <Company> <Milestone> Test Strategy

> Owned by the **QA engineer** (`<company>-qa`). The acceptance + integration test plan for this milestone: which `A#` each test asserts, the fixtures and oracle/byte-identical harnesses, cross-task integration coverage, and the Integrate-phase gate. Reviewed by `code-reviewer` (tests truly assert the `A#`, non-vacuous, mutation-probed; cross-task integration real). Version-stamped per milestone.
>
> **Two-level QA — read this first.** This is **product QA**: holistic, against-the-spec, at the milestone Integrate phase — it guards the *product*. It is **distinct from the reviewers** (`<role>-reviewer`), who are agentic-integrity QA: per-artifact PASS/FAIL gates that guard the *workflow* (catching hallucination, fabricated "done", vacuous tests, spec drift). Reviewers protect the process; I protect the product. Both run; neither replaces the other. The spec's acceptance is the shared harness.

## Scope
- **Milestone:** `<id>` — `<title>`.
- **Specs under test:** `<links to specs/ — the subsystem specs whose A# this milestone closes>`.
- **Tasks integrated:** `<T1..Tn — links to tasks/<id>.md>`.
- **Use cases that must demonstrably work end-to-end:** `<UC ids from use-cases/>`.

## A#-to-test coverage matrix
> The core of this plan: **every `A#` in the milestone's specs maps to at least one test here.** An `A#` with no test is a coverage gap the reviewer FAILs; a test that asserts no `A#` is scope creep. Each `A#` is executable by definition (a test asserts it without a human judging) — see `automation.md`: the spec is the harness.

| A# | what it asserts (one line) | test id / name | kind | fixtures | harness | task(s) covered |
|----|----------------------------|----------------|------|----------|---------|-----------------|
| A7 | `<drop any projection + replay ⇒ byte-identical>` | `<ReplayByteIdenticalTest>` | acceptance | `<fixture set>` | byte-identical | `<T2>` |
| A8 | `<reconciled cash == statement closing balance, Decimal exact; 1¢ delta FAILs>` | `<ReconcileToCentTest>` | acceptance | `<broker statement fixture>` | oracle (exact equality) | `<T3>` |
| `<A#>` | `<…>` | `<…>` | acceptance \| integration | `<…>` | `<…>` | `<…>` |

- **kind:** `acceptance` = asserts one `A#` directly against the spec. `integration` = asserts an `A#` (or a use-case end-to-end) **across task boundaries** (see the integration section).
- Aim for one row per `A#` minimum; high-stakes `A#` (money, security, irreversible) get a positive case **and** a failure case (the boundary that must FAIL).

## Fixtures
> Inputs and expected outputs that make each `A#` provable offline, with no human and no network. Deterministic, version-pinned, committed.

| fixture | shape / source | feeds tests | owner / provenance |
|---------|----------------|-------------|--------------------|
| `<broker-statement-2024Q1.xml>` | `<sanitized real-shaped record>` | `<A8 tests>` | `<captured + scrubbed; no PII>` |
| `<golden/replay-baseline/>` | `<expected byte-identical output snapshot>` | `<A7 tests>` | `<regenerated only on an intentional, reviewed format change>` |

- Fixtures are **deterministic and offline** — pinned, no live calls, no clock/locale/random leakage. A fixture that needs the network or a wall clock is not a fixture.
- **Golden/baseline files are reviewed artifacts.** A diff to a golden file is a behavior change: it gets re-reviewed, never silently re-blessed to make a test pass.
- Scrub real-shaped fixtures of secrets/PII before committing.

## Harnesses (how the outcome is judged)
> Per `automation.md`, every acceptance is outcome-harnessed: the result, not a human, is the arbiter. Name the harness each `A#` is bound to.

- **Byte-identical / replay harness.** `<Drop derived state, replay from the source of truth, assert the rebuilt output is byte-for-byte equal to the golden baseline. Catches any non-determinism (unsorted keys, float drift, locale/clock leakage).>`
- **Oracle / exact-equality harness.** `<An independently-computed expected value (e.g. broker closing balance) the system must match by Decimal exact equality. A 1-unit delta FAILs — no tolerance on money.>`
- **Property / invariant harness.** `<Spec invariants asserted over generated inputs (e.g. "sum of splits == total for all inputs"), not just hand-picked examples.>`
- **Boundary/failure harness.** `<The negative case for high-stakes A#: prove the system FAILs the way the spec says it must — the 1¢ delta, the malformed record, the unauthorized call.>`
- No suitable harness exists for an `A#` → **write one before testing**; an acceptance with no falsifiable outcome is not ready (escalate to PM+Architect as a spec gap).

## Cross-task integration coverage
> Per-task review proved each task in isolation. Integration proves the tasks **compose** — the seams between them and the use-cases that span more than one task. This is where milestone-level bugs hide.

| use case / flow | tasks it spans | A# asserted end-to-end | integration test | seam exercised |
|-----------------|----------------|------------------------|------------------|----------------|
| `<UC-RECONCILE>` | `<T2 → T3 → T5>` | `<A7, A8>` | `<ReconcileFlowIntegrationTest>` | `<RawRecordSource → ledger → report>` |
| `<…>` | `<…>` | `<…>` | `<…>` | `<…>` |

- Every milestone **use case** has at least one integration test walking it end-to-end against its backing `A#`.
- Exercise the **seams** from `architecture.md` (the swappable boundaries) — a contract honored by each side alone but broken across the seam is exactly what unit tests miss.
- Integration tests assert `A#`, same as acceptance — never a vague "it ran." If it can't name the `A#`/use-case it proves, it doesn't belong.

## Non-vacuity (mutation probe)
> A test that always passes proves nothing — and `code-reviewer` will mutation-probe this plan. Pre-empt it: every test in the matrix must be shown to fail when the behavior it guards is broken.

- For each `A#`, record the **mutation that turns its test RED** (e.g. "swap Decimal→Double ⇒ A8 FAILs", "shuffle key order in serialization ⇒ A7 FAILs", "off-by-one the closing balance ⇒ A8 FAILs").
- A test that survives its mutation is vacuous — fix the test, don't widen the tolerance.

## The Integrate-phase gate
> The milestone-level product gate. The CEO/manager does not call the milestone `done` until this gate is green. Distinct from the per-task reviewer gates that already passed.

Gate is GREEN only when **all** hold:
- [ ] **Coverage complete** — every `A#` in the milestone's specs has at least one passing test in the matrix; no orphan `A#`, no orphan test.
- [ ] **Acceptance green** — every acceptance test passes against committed fixtures, offline, deterministically (re-run twice ⇒ identical result).
- [ ] **Integration green** — every milestone use case passes end-to-end; every named seam exercised.
- [ ] **Non-vacuity shown** — each test's mutation probe documented and confirmed RED-on-break.
- [ ] **Reviewer sign-off** — `code-reviewer` returned `PASS` on this test plan (tests truly assert the `A#`; integration is real). This is the agentic-integrity check on the product-QA work.

Then I emit the standard handoff (`verdict` form), and on any failure the gate is **not** waived for a deadline:

```json
{"persona":"<company>-qa","task_id":"<milestone-id>","summary":"Integrate gate: A# coverage, acceptance + integration results.","files_changed":["Tests/Acceptance/…","Tests/Integration/…","Tests/Fixtures/…"],"decisions":["<harness/fixture choices, each bound to the A# it proves>"],"open_questions":["None"],"verdict":"PASS | FAIL","blocking_issues":["<concrete, per-A# on FAIL>"],"status":"DONE | BLOCKED"}
```

- **FAIL is per-`A#` and concrete** — "A8 fails: reconciled cash off by 3¢ on `broker-statement-2024Q1.xml`" — never a soft pass with caveats.
- A milestone with an uncovered or failing `A#` ships `BLOCKED`, not a fake `DONE`. Deadlines do not lower the bar (see `handoff-and-review.md`).
- When a defect traces to a recurring producer behavior, append a note to that persona's `MEMORY.md` for HR to promote.

## Definition of done
- [ ] A#-to-test matrix complete — every milestone `A#` mapped, every test names its `A#`.
- [ ] Fixtures committed, deterministic, offline, scrubbed; goldens are reviewed artifacts.
- [ ] Each `A#` bound to a named harness (byte-identical / oracle / property / boundary).
- [ ] Cross-task integration covers every milestone use case and exercises the architecture seams.
- [ ] Non-vacuity mutation probe documented per test.
- [ ] Integrate-phase gate run; `code-reviewer` PASS; handoff emitted with binary `status`.
