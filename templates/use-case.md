# Use-case templates — index + per-area manual

A use-case is the **demand**: *what the user does*, step-by-step, UI/behaviour-facing. The PM owns it (`use-cases/`). Every use-case names the backing spec `R#/A#` that satisfies it — the spec is the contract, the tests are written against its acceptance. A use-case with no backing spec is a gap reviewers FAIL.

Two tiers: an **index** (`use-cases/index.md`) and per-area **manuals** (`use-cases/<area>.md`). Mirror the real Ledger manuals — concrete enough that a developer builds from it and QA tests from it with no guessing.

---

## `use-cases/index.md` (PM-owned)

One row per use-case. ID is stable and SCREAMING-KEBAB (`UC-RECONCILE`). `Manual` links the manual section; `Spec` lists the backing `R#/A#`; `Source` is `file:line` once implemented, `(to be implemented)` while proposed.

```md
# <Company> Use-cases

| ID | Use case | Entry point | Manual | Spec | Source |
|----|----------|-------------|--------|------|--------|
| UC-RECONCILE | Confirm ledger ties out to the broker | Reports → Reconcile | [manual](reconcile.md#uc-reconcile) | R12, R13, A7, A8 | Reconcile.swift:88 |
| UC-IMPORT | Import a broker CSV statement | Accounts → Import | [manual](import.md#uc-import) | R4, A2, A3 | (to be implemented) |
```

Guidance:
- **ID** — verb-or-noun, one per user goal. Never renumber; deprecate instead.
- **Entry point** — the human path (`Reports → Reconcile`), not a function name. Detail goes in the manual.
- **Spec** — the exact `R#/A#` the steps rely on. If you can't name one, the spec is missing.
- **Source** — fill `file:line` at the entry view/handler once built; keep it current.

---

## `use-cases/<area>.md` — per-entry manual (PM-owned)

One `## UC-…` section per use-case. Steps are **action → see-that**: each step is one user action and the *exact* thing they then observe — real labels, values, states. For a not-yet-built feature, steps are *intended flows* (no exact labels yet) but the spec + acceptance are still binding.

```md
## UC-RECONCILE
**Confirm the ledger ties out to the broker before trusting any report**

- **Goal:** <one sentence — what the user achieves, in their terms>
- **Actor / preconditions:** <who is doing this, and what must already be true — e.g. "Account owner; at least one imported statement exists">
- **Entry point:** <exact tap/call path to reach it — "Reports tab → Reconcile → pick account">
- **Steps (action → see-that):**
  1. <do this> → <observe exactly this — real label / value / state, e.g. "see 'Closing balance £12,430.18' matching the statement">
  2. <do this> → <observe this>
  3. ...
- **Outcome:** <system state after success — what persisted, what flag flipped, what is now visible, e.g. "Account marked Reconciled as-of <date>; reports unlocked">
- **Variations & edge states:** <error paths, boundaries, empty / loading / failure — e.g. "1p mismatch → 'Out by £0.01', Reconcile disabled", "no statement imported → empty state with Import CTA">
- **Backed by spec:** R12, R13, A7, A8   ← the contract these steps rely on
- **Source:** Reconcile.swift:88   ← file:line once implemented; "(to be implemented)" while proposed
```

Rules:
- **action → see-that, always.** No step without an observable result. The see-that is what QA asserts; it must be specific (a label, a number, a state), never "it works".
- **Outcome is the persisted truth** — what a test can read back after the flow, traceable to an `A#`.
- **Variations cover empty / loading / failure / boundary** — these are where defects hide; name them or QA can't test them.
- **Backed-by must be real** — every listed `R#/A#` exists in `specs/`, and at least one use-case backs each `A#`. Mismatch either way = a gap reviewers FAIL.
- Keep it user-facing: no architecture, no Decimal/Foundation invariants here — those live in the spec.
