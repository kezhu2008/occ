# Automation — maximize autonomy, harness decisions to outcomes

The company exists to run with minimal founder intervention. Automation here is not "do things without asking" — it is **binding every decision to a measurable outcome so the outcome, not a human, is the arbiter.** Three rules.

## 1. Outcome-harnessed decisions (the core principle)

A technical decision is made autonomously and **validated by its outcome**, never by debate or sign-off.

- Don't ask "Postgres or SQLite?", "OpenAPI or gRPC?", "this algorithm or that one?" — **pick one, bind it to the acceptance criterion it must satisfy, and let the result judge it.** If it passes the spec's acceptance (tests green, reconciliation to-the-cent, byte-identical replay, latency budget met), it stands. If not, the outcome rejected it — try the alternative.
- Every reversible technical choice must have a **harness**: the measurable outcome that proves it right or wrong. No harness → write one before deciding (an acceptance test, a benchmark, a reconciliation check). A decision with no outcome to judge it is not ready to make.
- This is why the org is spec-driven and review-at-every-layer: the spec *is* the harness. The agent is free to choose any implementation because the acceptance criterion will catch a wrong one. Freedom of method, strictness of outcome.
- Log the decision + its harness in `decisions.md`. The log records *what outcome proved it*, not *who approved it*.

> Founder framing: "technical decisions are to be harnessed by OUTCOMES." Maximize the decisions the agent makes; make each one falsifiable by a result.

## 2. Front-load direction in planning; the run never blocks on the founder

The company is **long-running by construction**. The founder's decision weight is paid *up
front* in `review-and-plan`, not *mid-run*. So `run-the-company` has no hard stop — it decides,
logs, and keeps going.

**In planning (`review-and-plan`), refine as much as possible.** Enumerate the foreseeable
direction-forks for the upcoming run and, for each, get the founder to pre-decide it OR set a
**default + tolerance**. The board-reviewer gate checks that the run's foreseeable forks are
pre-decided with defaults; a plan is not signed until they are. This is where pricing posture,
naming rules, platform bets, and spend ceilings get settled — so the run inherits *standing
authority* and needs no fresh sign-off.

**In the run, three behaviors — none of them a hard stop:**

- **Reversible / internal fork** → decide, bind to its acceptance harness, log. *(rule 1)*
- **Direction-changing fork** → decide against the signed plan's pre-committed default (or, if
  unforeseen, the CEO's best-judgment default), log to `decisions.md` tagged `DIRECTION`, keep
  running. When two options both satisfy the plan, prefer the one cheapest to reverse — that
  costs nothing and keeps the founder's later course-correction a one-line flip.
- **Human-only access gap** → park only that thread, add it to the running needs-list, keep
  executing everything that doesn't depend on it (see rule 3).

The founder is not a code reviewer or an architect; outcomes are. The founder is also not a
mid-run approval gate; the *signed plan* is.

## 2b. Material-drift checkpoint (a notification, not a gate)

The drift harness is concrete, not a vibe: **the charter non-negotiables + the roadmap success
criteria.** After each `DIRECTION` decision the CEO asks: *does the product, as now decided,
still satisfy every charter non-negotiable and every roadmap success criterion as signed?*

- **Yes** → log quietly, run on.
- **No / bends one** → emit a **drift digest** (what drifted, why, the decisions behind it) and
  **keep running**. The founder may re-plan to course-correct; the run never waits on the answer.

`report-progress` surfaces the live trio anytime: `decisions.md` (DIRECTION-tagged), the
needs-list, and the current drift state. (See `decision-escalation.md` for the decision gate
this replaces.)

## 3. Front-load human-granted access before any long run

A long autonomous process must not stall halfway waiting for a human to grant something. So **before** starting, audit everything only a human can provide and batch-ask once:

- **Access audit (pre-flight):** scan the upcoming wave/milestone for every human-only dependency — credentials, API tokens, paid signups, cloud account/region/budget, deploy/App-Store permissions, device signing, anything requiring the founder's identity or wallet.
- **Batch + front-load:** hand the founder ONE list up front ("to run W6 uninterrupted I need: an AWS account+region+budget, an IBKR token re-issued for server use, an APNs key"). Then run to completion without further human gating.
- A human-only need discovered mid-run is a **front-loading miss** — but it does **not** stall the run: park only the blocked thread, append the need to the running needs-list, and keep executing every thread that doesn't depend on it (the founder grants access whenever; the parked thread resumes). Still note the miss for HR — it should have been caught in the pre-flight.
- Prefer **executable gates** to prose: a frontloaded prerequisite has a `verify:` command that proves it works (`aws sts get-caller-identity`, a real API probe) before the run depends on it — never assume a key/permission works.

## How this shows up across the org

| Role | Automation behavior |
|------|---------------------|
| CEO | runs the access pre-flight; front-loads every foreseeable direction-fork into the signed plan; in the run decides everything (logging `DIRECTION` calls), defers blocked threads, emits a drift digest but never stops |
| Architect / Tech Lead | choose methods freely; bind each choice to an acceptance/benchmark harness |
| Executors | TDD — the test IS the harness for their implementation choice |
| QA | owns the product-level harnesses (acceptance + integration against the spec) |
| Reviewers | verify the harness is real (non-vacuous, mutation-probed), not that a human approved |
