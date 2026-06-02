# Dynamic Workflows — how the org delivers a milestone

OCC uses **dynamic workflows** as the execution primitive. Decomposition is decided at runtime by the orchestrator, not hardcoded: look at the milestone, decide how many executors and reviewers to fan out, run them, verify adversarially, converge. Idea: parallel fan-out → adversarial verification → convergence-driven iteration.

## Division of labor

- **CEO sequences milestones inline** — milestones are dependent; it reads results between them and decides the next. It does not fan out milestones.
- **Inside a milestone, the managers run a multi-stage workflow.** The Tech Lead owns the execution fan-out; PM, UX and Architect feed it.

## The milestone-delivery workflow (multi-stage, not a toy)

A real milestone is not "implement N files." It is: decide *what* (use-cases), decide *the technical shape* (architecture/data-model), decompose into *tasks*, build+review each, then integrate and verify the whole. Each stage has its own review loop (`handoff-and-review.md`).

```js
export const meta = {
  name: 'deliver-milestone',
  description: 'PM use-cases → architect design → tech-lead tasks → build+review each → integrate → milestone review',
  phases: [
    { title: 'Scope' },        // PM use-cases (reviewed)
    { title: 'Design' },       // architect arch/data-model delta (reviewed)
    { title: 'Decompose' },    // tech-lead task specs (reviewed for design-fit)
    { title: 'Build' },        // executors implement
    { title: 'Review' },       // independent reviewer panel per task
    { title: 'Integrate' },    // QA across tasks + milestone success criteria
  ],
}

const milestone = args  // {id, success_criteria, ...} from milestones/<id>.md

// 1. SCOPE — PM defines use-cases, pm-reviewer gates. reviewLoop = produce→review→revise→converge.
const useCases = await reviewLoop('Scope',
  () => agent(pmPrompt(milestone),           { schema: USE_CASES }),
  uc => agent(pmReviewPrompt(uc),            { schema: VERDICT }))

// 2. DESIGN — architect updates architecture + data-model for this milestone; architect-reviewer gates.
const design = await reviewLoop('Design',
  () => agent(architectPrompt(milestone, useCases), { schema: DESIGN }),
  d  => agent(architectReviewPrompt(d, CHARTER),    { schema: VERDICT }))  // checks non-negotiables + future-wave seams

// 3. DECOMPOSE — tech-lead writes task specs; tech-lead-reviewer + architect check sizing and design-fit.
const tasks = await reviewLoop('Decompose',
  () => agent(techLeadDecomposePrompt(milestone, useCases, design), { schema: TASKS }),
  t  => agent(taskSpecReviewPrompt(t, design),                      { schema: VERDICT }))

// 4+5. BUILD + REVIEW — pipeline each task: executor implements, then it's independently reviewed
//      the moment its implementation lands (no barrier between tasks).
const built = await pipeline(tasks.items,
  task => agent(executorPrompt(task), { label:`impl:${task.id}`, phase:'Build', schema: HANDOFF }),
  (impl, task) => reviewGate(task, impl))            // scales panel to task stakes; loops on FAIL

// 6. INTEGRATE — one QA pass across all tasks against the milestone success criteria.
const qa = await agent(qaPrompt(milestone, built.filter(Boolean)), { phase:'Integrate', schema: VERDICT })

return { milestone: milestone.id, useCases, design, tasks: built.filter(Boolean), qa }

// ── helpers ──────────────────────────────────────────────────────────────
async function reviewLoop(phase, produce, review, cap = 3) {
  let artifact = await produce()
  for (let i = 0; i < cap; i++) {
    const v = await agent_wrap(review, artifact, phase)        // {verdict, blocking_issues}
    if (v.verdict === 'PASS') return artifact
    artifact = await produce(v.blocking_issues)                 // revise against the issues
  }
  throw new Error(`${phase} did not converge — escalate BLOCKED`)
}

function reviewGate(task, impl) {                                // the executor review gate
  const lenses = task.high_stakes
    ? ['spec-compliance', 'correctness', 'security-or-oracle']   // adversarial panel
    : ['spec-compliance']                                        // single reviewer
  return parallel(lenses.map(lens => () =>
    agent(reviewerPrompt(task, impl, lens), { label:`review:${task.id}:${lens}`, phase:'Review', schema: VERDICT })
  )).then(vs => ({ ...impl, verdict: majorityPass(vs) ? 'PASS' : 'FAIL', reviews: vs.filter(Boolean) }))
  // on FAIL the tech-lead re-dispatches the executor with blocking_issues, retry cap, else escalates
}
```

Rules baked in:
- **Every stage is a review loop** — produce, independent review, revise, converge. Not just the code stage.
- **The reviewer is always a separate `agent()`** — independence is structural, never self-review.
- **High-stakes tasks get a multi-lens adversarial panel**, reviewers prompted to *refute*, majority-PASS to clear.
- **Pipeline, not barrier** — task A reviews while task B still builds. Use a barrier only when a stage needs all prior results (e.g. integration QA, which genuinely needs every task).

## When NOT to spin a workflow

- One task in one context → dispatch one executor + one reviewer; no machinery.
- Strictly sequential subtasks → inline or a 1-wide pipeline.
- Cost-sensitive run → narrow the fan-out and `log()` what you capped — never silently truncate.

## Two places dynamic workflows show up

1. **Building/running OCC itself** — fan out exploration, baseline tests, and adversarial review of generated personas (as this build did).
2. **Inside the recruited company** — the milestone-delivery workflow above. `occ-recruiting` writes this pattern into the Tech Lead persona, with PM/architect/UX as the upstream stages.

## Shared schemas

`USE_CASES, DESIGN, TASKS, HANDOFF, VERDICT` are JSON shapes from `handoff-and-review.md`. Pass them as the `schema` option so `agent()` returns validated objects — no parsing, automatic retry on malformed output.
