# Ledger — Milestone-Delivery Workflow

The concrete, runnable workflow the Ledger org uses to take **one milestone** from intent to a verified, acceptance-passing increment. This is the execution primitive of `dynamic-workflows.md`, specialised to Ledger's roster, artifacts, and invariants. The CEO sequences milestones *inline* (each milestone reads the previous one's result and decides the next); **inside** a milestone the Tech Lead runs the staged workflow below, with PM, UX, and Architect feeding it.

It is written to be real for the wave we are in: **W6 — Backend Lift** (move `LedgerCore` verbatim into AWS behind a `FactStore` protocol; Path A single-tenant unattended sync before Path B multi-tenant SaaS). The acceptance bar for the whole wave is non-negotiable and threads every stage: **end-to-end on real `.flex-local` data, tax numbers BYTE-IDENTICAL to the on-device build** (the A6 determinism property).

---

## 0. The shape

```
Scope ──► Design ──► Decompose ──► Build + Review ──► Integrate
 PM        Architect   Tech Lead     backend-eng /      QA
 (+pm-rev) (arch +     (tasks +      ios-eng, each      (acceptance +
           data-model  system-design pipelined into     integration
           + co-owned  ; tech-lead-  a reviewer panel    vs A#) ──► milestone
           spec)       reviewer +    sized to stakes)    success gate
                       architect)
```

Five stages, **each its own review loop** (produce → independent review → revise → converge). The reviewer is always a *different persona dispatched fresh* — never self-review. High-stakes tasks (money, determinism, secrets, public API) get an **adversarial panel**, not one reviewer. The final **Integrate gate** is a barrier: QA needs every task before it can assert cross-task acceptance.

Roster (from `company_policies/org-chart.md`; all skills prefixed `ledger-`):

| stage | producer | reviewer(s) | owns |
|-------|----------|-------------|------|
| Scope | `ledger-product-manager` | `ledger-pm-reviewer` | `use-cases/` (+ co-owns `specs/`) |
| Design | `ledger-architect` | `ledger-architect-reviewer` | `architecture.md`, `data-model.md` (+ co-owns `specs/`) |
| Decompose | `ledger-tech-lead` | `ledger-tech-lead-reviewer` **+** `ledger-architect` (design-fit) | `tasks/`, `system-design.md` |
| Build | `ledger-backend-engineer` / `ledger-ios-engineer` | `ledger-*-reviewer` panel | core / app code |
| Integrate | `ledger-qa-engineer` | (its tests reviewed by `ledger-backend-engineer-reviewer` for non-vacuity) | `test-strategy.md` |

---

## 1. The workflow (JS)

```js
export const meta = {
  name: 'ledger-deliver-milestone',
  description:
    'PM use-cases -> architect arch/data-model + co-owned R#/A# spec -> tech-lead tasks/system-design ' +
    '-> build+review each task (panel scaled to stakes) -> QA acceptance+integration vs A#.',
  phases: [
    { title: 'Scope' },      // PM use-cases (pm-reviewer)
    { title: 'Design' },     // architect architecture.md + data-model.md + spec delta (architect-reviewer)
    { title: 'Decompose' },  // tech-lead tasks/ + system-design.md (tech-lead-reviewer + architect design-fit)
    { title: 'Build' },      // backend/ios engineers implement, TDD vs A#
    { title: 'Review' },     // independent reviewer panel per task, sized to stakes
    { title: 'Integrate' },  // QA acceptance + integration across tasks vs the milestone A#
  ],
}

// args = the milestone spec read from company_policies/milestones/<id>.md
// e.g. W6-M1 "Pre-refactors", W6-M2 "Server core on Lambda", W6-M3 "Durable store + secrets",
//      W6-M5 "Read API + thin iOS client", W6-M7 "Acceptance + deploy".
const milestone = args   // { id, goal, success_criteria:[A#...], path:'A', non_negotiables:[...] }

// CHARTER non-negotiables that EVERY stage is checked against (never silently dropped):
const CHARTER = {
  moneyDecimalOnly: true,            // never Double; bankers' rounding only at the edges
  foundationOnlyCore: true,          // LedgerCore imports Foundation only (PortabilityLockTests) -> the server-lift seam
  deterministicReplay: 'A6',         // same facts + same config => byte-identical projections
  taxOracleParity: true,             // CGT/franking/FX/FITO vs the hand-derived oracle, IND x SMSF, FY25 x FY26
  liftNotRebuild: true,              // keep the core verbatim; move storage/secrets/schedule/notify/transport behind seams
  pathAbeforeB: true,                // single-tenant unattended before multi-tenant SaaS
}

// ── STAGE 1 · SCOPE ───────────────────────────────────────────────────────
// PM writes step-by-step use-cases (action -> see-that) for THIS milestone; pm-reviewer gates that
// each is real, scoped, and names its backing R#/A# spec IDs. (specs-and-use-cases.md)
const useCases = await reviewLoop('Scope',
  issues => agent(
    pmPrompt(milestone, issues), { label:'scope:pm', phase:'Scope', schema: USE_CASES }),
  uc => agent(
    pmReviewPrompt(uc, milestone), { label:'scope:review', phase:'Scope', schema: VERDICT }))
  // e.g. W6-M5: UC-SYNC-HEALTH "see last sync / next sync / status, trigger sync now"
  //            UC-VIEW-SERVER-STATE "open the app, see server-computed positions/tax, no phone-must-be-open"
  // each names its spec: the server-side analogues of R5 (projection replay), R21 (sync/import), A6.

// ── STAGE 2 · DESIGN ──────────────────────────────────────────────────────
// Architect updates architecture.md (current-system seams/invariants) + data-model.md (types/schema/
// serialization), and CO-OWNS the R#/A# spec delta with the PM. architect-reviewer gates fit-to-charter
// + that future-wave seams (Path B) are preserved, not foreclosed.
const design = await reviewLoop('Design',
  issues => agent(
    architectPrompt(milestone, useCases, CHARTER, issues),
    { label:'design:architect', phase:'Design', schema: DESIGN }),
  d => agent(
    architectReviewPrompt(d, CHARTER, useCases),
    { label:'design:review', phase:'Design', schema: VERDICT }))
  // e.g. W6-M1: architecture.md gets `protocol FactStore` (the single most important pre-refactor; today the
  //            actor holds a CONCRETE FactStore) + ConnectionState consolidation; data-model.md gets the
  //            Dynamo partition key tenant#connection / sort key factId, mapping FactID dedup -> conditional put.
  //            Spec delta states the invariant: the on-disk and Dynamo FactStore impls are byte-equivalent (A6 holds
  //            across the seam) and the D83-T5 hazard (silent 0.5 CGT discount for an unrecognised account) is closed
  //            with a conservative 0% fallback. architect-reviewer FAILs if the seam leaks Foundation-only-ness or
  //            bakes an AUD/marked value into a fact (R7/R8 firewall).

// ── STAGE 3 · DECOMPOSE ───────────────────────────────────────────────────
// Tech Lead turns spec+design into independently-shippable tasks/<id>.md (acceptance = the relevant A#) and
// owns system-design.md (AWS topology / API contract / migration). TWO reviewers: tech-lead-reviewer (sizing,
// chain correctness) AND architect (design-fit — does the decomposition respect the seams it just drew?).
const tasks = await reviewLoop('Decompose',
  issues => agent(
    techLeadDecomposePrompt(milestone, useCases, design, issues),
    { label:'decompose:tl', phase:'Decompose', schema: TASKS }),
  t => panelVerdict([
    () => agent(taskSpecReviewPrompt(t, milestone),       { label:'decompose:tl-review',   phase:'Decompose', schema: VERDICT }),
    () => agent(taskDesignFitPrompt(t, design, CHARTER),  { label:'decompose:arch-fit',    phase:'Decompose', schema: VERDICT }),
  ]))  // BOTH must PASS — design-fit is not optional for a load-bearing decomposition.
  // Each task carries { id, title, owner: 'backend'|'ios', acceptance:[A#...], high_stakes:bool, reviewer_chain }.
  // The Architect-vs-TechLead seam itself is a consultation edge: if the decomposition is blocked by a missing
  // protocol, the tech-lead files a consultation (consultation.md), the architect resolves by editing the artifact,
  // and the revised design re-enters its own review loop. NEVER guess past an ambiguous seam.

// ── STAGE 4+5 · BUILD + REVIEW ────────────────────────────────────────────
// Pipeline (NOT a barrier): task A is reviewed the moment its implementation lands while task B still builds.
// Each executor implements TDD-first against its A#; then the reviewGate runs a panel sized to the task's stakes.
const built = await pipeline(tasks.items,
  task => agent(
    executorPrompt(task, design),
    { label:`build:${task.id}`, phase:'Build', stack: task.owner === 'ios' ? ['ios','swiftui','swift'] : ['swift'],
      schema: HANDOFF }),
  (impl, task) => reviewGate(task, impl))

// ── STAGE 6 · INTEGRATE (the QA gate — a barrier; needs every task) ───────
// One QA pass across ALL built tasks: acceptance + integration tests asserting the milestone's A# end-to-end.
// This is PRODUCT QA (2nd level), distinct from the per-artifact reviewers above. For W6 this is THE wave bar.
const qa = await agent(
  qaIntegratePrompt(milestone, built.filter(Boolean), CHARTER),
  { label:'integrate:qa', phase:'Integrate', schema: VERDICT })
  // W6-M7 acceptance, executed not judged:
  //   A6-server: drop server projections + replay from the Dynamo fact log => byte-identical to prior (test).
  //   parity:    run the real .flex-local bytes through the Lambda; CGT/franking/FX/FITO/EOY for IND and SMSF,
  //              FY25 and FY26, EXACTLY equal (Decimal exact equality) to the on-device oracle. A 1-cent delta FAILs.
  //   idempotent: re-ingest the same statement => identical fact count server-side (R2/A5 across the seam).
// QA's tests are themselves reviewed by ledger-backend-engineer-reviewer for non-vacuity (mutation probe) —
// a green test that asserts nothing does not clear the gate.
if (qa.verdict !== 'PASS')
  return escalate('Integrate', milestone, qa.blocking_issues)   // milestone is BLOCKED, never a fake DONE.

return { milestone: milestone.id, useCases, design, tasks: built.filter(Boolean), qa, status: 'DONE' }

// ── helpers ────────────────────────────────────────────────────────────────

// produce -> independent review -> revise against blocking_issues -> converge. Used by EVERY stage, not just code.
async function reviewLoop(phase, produce, review, cap = 3) {
  let artifact = await produce(/* no issues on first pass */)
  for (let i = 0; i < cap; i++) {
    const v = await review(artifact)                 // {verdict, blocking_issues}
    if (v.verdict === 'PASS') return artifact
    artifact = await produce(v.blocking_issues)      // revise against CONCRETE fixable issues
  }
  // convergence discipline: do not loop forever, do not soft-pass. Escalate BLOCKED up the line.
  return escalate(phase, artifact, [`${phase} did not converge in ${cap} rounds`])
}

// The executor review gate: scale the PANEL to the task's stakes — never remove the gate, only size it.
function reviewGate(task, impl) {
  const lenses = task.high_stakes
    // High-stakes lenses for Ledger: tax/secrets/determinism/public-API tasks get an ADVERSARIAL panel,
    // each reviewer prompted to REFUTE; majority-PASS clears.
    ? ['spec-compliance', 'correctness', task.owner === 'backend' ? 'oracle-and-determinism' : 'hig-and-accessibility']
    : ['spec-compliance']                            // trivial/mechanical task -> one reviewer, spec pass only
  return parallel(lenses.map(lens => () =>
    agent(reviewerPrompt(task, impl, lens, CHARTER),
      { label:`review:${task.id}:${lens}`, phase:'Review', schema: VERDICT })
  )).then(vs => {
    const pass = majorityPass(vs)
    if (!pass) {
      // on FAIL the tech-lead re-dispatches the SAME executor with the blocking_issues (retry cap),
      // else escalates BLOCKED. The executor's own passing tests are NOT review — independence is structural.
      return retryOrEscalate(task, impl, vs)
    }
    return { ...impl, verdict: 'PASS', reviews: vs.filter(Boolean) }
  })
}

function panelVerdict(reviewers) {                    // ALL must PASS (used where both reviewers are mandatory)
  return parallel(reviewers).then(vs =>
    vs.every(v => v.verdict === 'PASS')
      ? { verdict: 'PASS' }
      : { verdict: 'FAIL', blocking_issues: vs.flatMap(v => v.blocking_issues ?? []) })
}
```

---

## 2. Why each stage exists (Ledger-specific)

- **Scope is reviewed, not assumed.** A wrong use-case ("the app keeps a Sync-now button machine") would carry a single-device assumption into the server lift. The PM frames W6's flows as the *inverted* model — the server pulls on schedule/event, the client only *observes* `last sync / next sync / status` — and `pm-reviewer` FAILs any use-case with no backing R#/A# or that re-introduces the phone-must-be-open coupling.
- **Design owns correctness; it is where the seam is drawn.** The Architect adds `protocol FactStore` to `architecture.md` (today the actor holds a concrete `FactStore` — the one pre-refactor that gates the whole lift), states the data-model partition keys, and the **R7/R8 firewall invariant**: a fact stores only what happened in native currency; no AUD/marked value is ever baked in, so swapping the store or feed can move market value but **never** cost base or tax. `architect-reviewer` checks the lift preserves Foundation-only-ness and the Path B seams.
- **Decompose owns delivery and needs TWO gates.** `tech-lead-reviewer` checks tasks are context-sized, independently shippable, and the persona chain is correct; the **Architect** re-reviews for design-fit so the decomposition can't quietly violate the seam it was handed. `system-design.md` (AWS topology, EventBridge schedule + on-demand trigger, Secrets Manager `TokenStore`, the read API + Cognito, SNS→APNs) is the Tech-Lead-owned doc, distinct from `architecture.md` — they are never merged.
- **Build + Review pipelines.** Backend tasks pull the `swift` stack; iOS tasks pull `ios, swiftui, swift`. Each is TDD-first against its `A#` (the test is the outcome-harness for the executor's method choice). The reviewer panel is sized to stakes: a Secrets-Manager `TokenStore` impl or a CGT-fallback change is high-stakes (`spec-compliance` + `correctness` + `oracle-and-determinism`); a read-only sync-health view is one reviewer.
- **Integrate is the wave's truth.** QA runs the real `.flex-local` bytes through the Lambda and asserts byte-identical tax vs the on-device oracle across IND×SMSF and FY25×FY26, plus server-side replay (A6) and idempotent re-ingest (A5). This is product QA, separate from the reviewers; a 1-cent delta is a FAIL, and the milestone reports `BLOCKED`, never a cosmetic `DONE`.

## 3. Rules baked in (do not relax under deadline)

- **Every stage is a review loop** — produce, independent review, revise, converge. Not just the code stage.
- **The reviewer is always a separate, freshly-dispatched persona.** The Tech Lead reading the diff is not review; the executor's own green tests are not review. Independence is structural.
- **High-stakes → adversarial panel, majority-PASS.** Money (Decimal/oracle), determinism (A6), secrets, and public API never go through a single reviewer. Scale the gate; never remove it.
- **Pipeline, not barrier — except Integrate.** Tasks review while others build; only QA waits for all of them, because cross-task acceptance genuinely needs every task.
- **Consultation, not guessing.** A blocked seam (missing `FactStore` protocol, an under-specified use-case, code contradicting a doc) is a `blocking:true` consultation to the artifact's owner, who resolves by editing the artifact; the revision re-enters its review loop. Cap the round-trips; if it doesn't converge, escalate BLOCKED.
- **Escalation is for direction-changing calls only.** Reversible + internal + low-blast-radius (Dynamo vs Aurora for the fact store, OpenAPI shape of the read API) is decided and logged in `decisions.md`. Externally-visible / money / strategic / future-wave-locking (the licensed price-feed spend, a public rename, Path A→B commitment) is staged behind a flag and frontloaded to the founder as a one-line decision — the milestone proceeds unblocked while it waits.
- **The acceptance bar does not move.** For W6 every stage is ultimately answerable to one fact: real data in, **byte-identical tax out**.
