# Handoff & the Independent Review Gate

Two contracts hold the org together: the **handoff artifact** (how work moves between personas) and the **review gate** (why work is trustworthy). The review gate is the single most-violated discipline — it gets its own anti-rationalization section.

## The handoff artifact (uniform across all personas)

Every persona — executor, reviewer, manager, HR — returns the SAME shape. Uniformity is the mechanic: the receiver always knows what it's getting. Prose-only returns are rejected.

```json
{
  "persona": "<company>-backend-engineer",
  "task_id": "M4-T2",
  "summary": "One paragraph, plain language, what changed and why.",
  "files_changed": ["Sources/Core/Tax/FXGain.swift", "Tests/FXGainTests.swift"],
  "decisions": ["Chose Decimal over Double for FX gain — money must not use float."],
  "open_questions": ["None"],
  "status": "DONE | BLOCKED"
}
```

Reviewer handoffs add `verdict: PASS | FAIL` and `blocking_issues: [...]`. `status` is binary — never "DONE-ish", "in-progress", "mostly".

## The review gate (the Iron Rule)

**No task is `DONE` until an INDEPENDENT reviewer persona returns `PASS`.**

- *Independent* = a different persona than the implementer, dispatched as its own fresh subagent.
- The orchestrator (manager) reading the diff themselves is **not** review. The implementer's own tests passing is **not** review.
- Reviews are two-stage (from v1, proven): **Stage 1 — spec compliance** (does it do exactly what the task spec said, nothing more, nothing less; run the class-closing sweep for any retire/remove/rename task). **Stage 2 — code quality** (correctness, tests are non-vacuous via mutation probe, no scope creep).

## Review applies to EVERY layer, not just code

A wrong task spec is as expensive as a bug; a wrong architecture is worse than both. So every authoritative artifact passes an independent reviewer of the appropriate layer before it is treated as done. The reviewer is a real seat in `org-chart.md`.

| producer | artifact | independent reviewer | review focus |
|----------|----------|----------------------|--------------|
| CEO | roadmap, milestone specs | `board-reviewer` (then founder sign-off) | coherence, spec quality, success criteria are observable, frontloading complete |
| product-manager | use-cases | `pm-reviewer` | each use-case is real, scoped, step-by-step, names its backing spec IDs |
| product-manager + architect | **specs (R#/A#)** | `pm-reviewer` + `architect-reviewer` | every `A#` executable; every use-case backed; invariants stated |
| ux-manager | ux manuals | `ux-reviewer` | every use case has a mapped flow; nothing orphaned |
| architect | architecture, data-model | `architect-reviewer` | fits charter non-negotiables; future-wave seams preserved; invariants stated |
| tech-lead | task specs, system-design | `tech-lead-reviewer` (+ architect for design-fit) | tasks context-sized; acceptance = the relevant `A#`; persona chain correct |
| executor | code | `<role>-reviewer` | the two-stage code review above |
| QA engineer | acceptance/integration tests | `code-reviewer` (non-vacuous, mutation-probed) | tests truly assert the spec `A#`; cross-task integration real |

The board-reviewer is the answer to "who reviews the CEO": it checks the CEO's plan internally so the founder receives a clean, consistent plan (fewer founder cycles — consistent with the CEO mandate), then the founder gives final sign-off.

### Two levels of QA (don't conflate them)
- **Agentic-integrity QA = the reviewer rows above.** They catch AI-induced failure (hallucination, fabricated "done", vacuous tests, spec drift) and guard the *workflow*. Per-artifact PASS/FAIL gates.
- **Product QA = the QA engineer row.** Real-world quality: holistic cross-task code review + acceptance/integration tests against the spec, at the milestone Integrate phase. Guards the *product*.

Reviewers protect the process; QA protects the product. Both are required.

### Consultation is not review
A reviewer returns a PASS/FAIL verdict on a finished artifact. **Consultation** (see `consultation.md`) is a producer pushing a question or impact *back up the design chain* before/while building — e.g. tech-lead → architect "this decomposition is blocked by invariant X; spec R5/R7 impacted." It feeds the review loop (the owner revises, the revision is re-reviewed); it does not replace the gate.

## The review loop (produce → review → revise → converge)

Every gate runs the same loop:

```
producer emits artifact + handoff
        │
        ▼
independent reviewer returns {verdict: PASS|FAIL, blocking_issues[]}
        │
   ┌────┴────┐
 PASS       FAIL
   │          │
 done    producer revises against blocking_issues ──► re-review
            (retry cap N; on exhaustion escalate BLOCKED up the line)
```

- The reviewer is always a **different persona**, dispatched fresh. Independence is structural.
- FAIL returns concrete, fixable blocking issues — never a soft pass with caveats.
- Cap retries (default 2–3); if it still FAILs, the producer's manager escalates `BLOCKED`, it does not silently ship.
- High-stakes artifacts (money, security, public API, load-bearing architecture) get an **adversarial panel** instead of one reviewer (see below).

### Deadlines do not lower the bar

Baseline finding: under "founder wants it tonight, it's the last brick, tests pass," the orchestrator self-reviewed and skipped the independent gate, reasoning *"the cost of a heavyweight gate exceeds the residual risk."* That is the exact rationalization this rule forbids.

| Rationalization | Reality |
|-----------------|---------|
| "It's the last brick, everything else is verified" | The last brick is where rushed bugs hide. Verified-before ≠ verified-after; a change can regress prior work. |
| "Tests pass, the executor said DONE" | The executor wrote the code AND the tests — that's circular. Independent review breaks the circle. |
| "I read the diff myself, it's fine" | Self-review by the orchestrator is not independent. You share the implementer's blind spots and your deadline pressure. |
| "A full reviewer panel is too heavyweight for one small task" | Then dispatch ONE reviewer, not a panel. Scale the gate; never remove it. The gate is cheap relative to a bad demo. |
| "Founder needs it tonight" | A `DONE` that fails the demo costs the founder more than a 10-minute review. Ship `BLOCKED-on-review` honestly instead of a fake `DONE`. |
| "I'll mark DONE and flag it for review later" | "DONE with a flag" is a lie to the ledger. If it's not reviewed, status is `BLOCKED` or `IN-REVIEW`, never `DONE`. |

### Red flags — STOP, you are about to skip the gate

- "I'll just verify it myself this once"
- "No time to spin up a reviewer"
- "It's too simple to need review"
- "Tests pass so it's done"
- "Mark it done, review can happen after the demo"

All of these mean: **dispatch the independent reviewer now.** If you cannot (no reviewer available, environment blocked), the status is `BLOCKED`, not `DONE`.

### Scaling the gate (not removing it)

- Trivial/mechanical task → one reviewer, spec-compliance pass only.
- Normal task → one reviewer, both stages.
- High-stakes/irreversible/money/security task → adversarial **panel**: 2–3 reviewers with distinct lenses (correctness, does-it-reproduce, security), majority must PASS. Use the Workflow tool to fan them out (see `dynamic-workflows.md`).

The choice is *how much* review, never *whether*.
