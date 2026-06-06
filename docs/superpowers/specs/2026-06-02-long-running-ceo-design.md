# Long-running CEO — refine-then-run

**Date:** 2026-06-02
**Topic:** Make the recruited CEO long-running. Push all founder-decision weight forward into
planning/review; make `run-the-company` non-blocking by construction.

## Problem

The recruited `<company>-ceo` keeps pausing mid-run. The pauses come from three places:

1. **Escalation** — "escalate only direction-changing calls" is a *mid-run hard stop*.
2. **Access pre-flight miss** — a human-only dependency discovered mid-run stalls the run.
3. **Turn-ending questions** — the agent stops its turn to ask the founder.

(Internal **consultation** between roles does not pause the founder — it is the CEO doing more
work in-session — so it is out of scope.)

Founder intent: *refine as much as possible during planning and review, then keep running.*
Implementation decisions belong to the CEO / PM / tech-lead, not the founder.

## The reframe

Trust moves from **mid-run sign-off** to **front-loaded sign-off**. Planning carries the
escalation weight; the run never blocks on the founder.

### 1. Planning carries the escalation weight
`review-and-plan` gains a required step: **enumerate the foreseeable direction-forks** for the
upcoming run and, for each, get the founder to pre-decide it OR set a **default + tolerance**.
The `board-reviewer` gate adds one check: *are the run's foreseeable forks pre-decided with
defaults?* A plan is not signed until they are. This is where the founder and CEO refine hard
(pricing posture, naming rules, platform bets, spend ceilings) so the run inherits standing
authority.

### 2. `run-the-company` never blocks on the founder
Three behaviors replace the hard stop:

- **Reversible / internal fork** → decide, harness to acceptance, log. *(unchanged)*
- **Direction-changing fork** → decide against the signed plan's pre-committed default (or, if
  unforeseen, the CEO's best-judgment default), log to `decisions.md` tagged `DIRECTION`, keep
  running. When two options both satisfy the plan, prefer the one cheapest to reverse — costs
  nothing, keeps the founder's later course-correction a one-line flip.
- **Human-only access gap** → park *only* that thread, append it to a running needs-list, keep
  executing every thread that does not depend on it. Still logged as a front-loading-miss
  signal for HR.

### 3. Material-drift checkpoint (a notification, not a gate)
The drift harness is concrete, not a vibe: **the charter non-negotiables + the roadmap success
criteria.** After each `DIRECTION` decision the CEO asks: *does the product, as now decided,
still satisfy every charter non-negotiable and every roadmap success criterion as signed?*

- **Yes** → log quietly, run on.
- **No / bends one** → emit a **drift digest** (what drifted, why, the decisions behind it) and
  **keep running**. The founder may re-plan to course-correct; the run never waits.

`report-progress` surfaces the live trio anytime: `decisions.md` (DIRECTION-tagged), the
needs-list, and the current drift state.

## What this deliberately keeps

- The **review bar (Iron Rule)** is untouched. Independent review at every layer still gates
  every artifact; QA still verifies the product. Autonomy is about *founder* interruptions, not
  about lowering the review gate.
- The **outcome-harness** philosophy is extended, not replaced: the charter non-negotiables and
  roadmap success criteria become the *drift* harness, the same way `A#` acceptance is the
  *decision* harness.
- "Prefer the reversible option when adequate" replaces the old "stage ready-to-flip *and wait*"
  — the staging hygiene survives; the waiting does not.

## File surface

**OCC model (durable fix):**
- `references/automation.md` — rule 2 reframe + long-running/drift section
- `references/decision-escalation.md` — reframe: nothing hard-stops mid-run
- `templates/handbook.md` — Automation + Decision-escalation + run-the-company row
- `templates/job-description.md` — escalate bullet note
- `SKILL.md` — `run-the-company` description + core discipline 2

**Worked example + live company (identical files, same edits):**
- `examples/ledger-org/claude-skills/ledger-ceo/SKILL.md` ↔ `/Users/kezhu/git/ledger/.claude/skills/ledger-ceo/SKILL.md`
- `examples/ledger-org/company_policies/job-descriptions/ceo.md` ↔ `/Users/kezhu/git/ledger/company_policies/job-descriptions/ceo.md`
- `examples/ledger-org/company_policies/handbook.md` ↔ `/Users/kezhu/git/ledger/company_policies/handbook.md`

**Then:** install the OCC skill for the user (`~/.claude/skills/occ`).
