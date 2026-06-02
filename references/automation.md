# Automation — maximize autonomy, harness decisions to outcomes

The company exists to run with minimal founder intervention. Automation here is not "do things without asking" — it is **binding every decision to a measurable outcome so the outcome, not a human, is the arbiter.** Three rules.

## 1. Outcome-harnessed decisions (the core principle)

A technical decision is made autonomously and **validated by its outcome**, never by debate or sign-off.

- Don't ask "Postgres or SQLite?", "OpenAPI or gRPC?", "this algorithm or that one?" — **pick one, bind it to the acceptance criterion it must satisfy, and let the result judge it.** If it passes the spec's acceptance (tests green, reconciliation to-the-cent, byte-identical replay, latency budget met), it stands. If not, the outcome rejected it — try the alternative.
- Every reversible technical choice must have a **harness**: the measurable outcome that proves it right or wrong. No harness → write one before deciding (an acceptance test, a benchmark, a reconciliation check). A decision with no outcome to judge it is not ready to make.
- This is why the org is spec-driven and review-at-every-layer: the spec *is* the harness. The agent is free to choose any implementation because the acceptance criterion will catch a wrong one. Freedom of method, strictness of outcome.
- Log the decision + its harness in `decisions.md`. The log records *what outcome proved it*, not *who approved it*.

> Founder framing: "technical decisions are to be harnessed by OUTCOMES." Maximize the decisions the agent makes; make each one falsifiable by a result.

## 2. Escalate only direction-changing calls

The escalation bar is **"does this significantly change direction?"** — not merely "is it irreversible?". (See `decision-escalation.md` for the reversibility/blast-radius detail; this is the sharper top-level test.)

- Significantly changes direction → escalate (pricing, business model, public/brand naming, a platform bet that locks future waves, anything that redefines what the product *is*).
- Everything else → decide, harness, log, proceed. The founder is not a code reviewer or an architect; outcomes are.

## 3. Front-load human-granted access before any long run

A long autonomous process must not stall halfway waiting for a human to grant something. So **before** starting, audit everything only a human can provide and batch-ask once:

- **Access audit (pre-flight):** scan the upcoming wave/milestone for every human-only dependency — credentials, API tokens, paid signups, cloud account/region/budget, deploy/App-Store permissions, device signing, anything requiring the founder's identity or wallet.
- **Batch + front-load:** hand the founder ONE list up front ("to run W6 uninterrupted I need: an AWS account+region+budget, an IBKR token re-issued for server use, an APNs key"). Then run to completion without further human gating.
- A human-only need discovered mid-run is a **front-loading miss** — note it for HR; it should have been caught in the pre-flight.
- Prefer **executable gates** to prose: a frontloaded prerequisite has a `verify:` command that proves it works (`aws sts get-caller-identity`, a real API probe) before the run depends on it — never assume a key/permission works.

## How this shows up across the org

| Role | Automation behavior |
|------|---------------------|
| CEO | runs the access pre-flight; escalates only direction-changing calls; everything else delegated and outcome-judged |
| Architect / Tech Lead | choose methods freely; bind each choice to an acceptance/benchmark harness |
| Executors | TDD — the test IS the harness for their implementation choice |
| QA | owns the product-level harnesses (acceptance + integration against the spec) |
| Reviewers | verify the harness is real (non-vacuous, mutation-probed), not that a human approved |
