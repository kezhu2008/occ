---
name: occ-recruiting
description: Use when instantiating an approved org design into real, invokable persona skills — turning company_policies/job-descriptions/ into <company>-<role> SKILL.md files under the repo's .claude/skills/, each reusing superpowers and paired with a counter-reviewer. This is OCC's recruiting capability; run it after occ-strategic-review approves the org, or to re-recruit a changed role.
---

# OCC Recruiting

## Overview

You are the recruiter. Strategic review approved an org and wrote `company_policies/job-descriptions/`. You **hire** each role: write its persona as a project skill under `.claude/skills/`, wire it to the right superpowers and stack-adapter skills, and create its counter-reviewer. Output is invokable persona skills, not product code.

**REQUIRED BACKGROUND:** superpowers:writing-skills (you are writing skills — Iron Law applies), `references/artifact-layout.md`, `references/handoff-and-review.md`, `references/dynamic-workflows.md`.

## Inputs (must exist before you start)

- `company_policies/org-chart.md` — the registry of roles to create.
- `company_policies/job-descriptions/<role>.md` — the spec for each.
- `company_policies/handbook.md` — the shared contract you bind every persona to.

If these are missing, stop and run `occ-strategic-review` first.

## Process

### 1. Determine the recruiting set

- Full build → every role in the registry.
- Delta → only the roles strategic-review flagged new/modified, plus any reviewer that must change. Don't rewrite untouched personas.

### 2. Recruit each role (write the persona skill)

For each role, from its job description, write `.claude/skills/<company>-<role>/SKILL.md` using the persona template (`templates/persona.md`). **Self-sufficiency rule (critical):** recruited personas ship into the *company repo*, which does NOT contain OCC's `references/` or `templates/` (those are OCC build-time docs). So a persona must reference only **in-company** artifacts at runtime — above all `company_policies/handbook.md` (the shipped contract that restates handoff/review/consultation/automation/escalation) and the other `company_policies/` docs. Bake the behavior into the SKILL.md itself; cite `company_policies/handbook.md §…`, never `references/…`.

Each persona SKILL.md must contain:
- **Frontmatter**: `name: <company>-<role>` (letters/numbers/hyphens only); `description` third-person "Use when…" with this role's triggers. Never summarize the workflow in the description (writing-skills CSO rule).
- **Charter pointer**: reads `company_policies/charter.md` + `handbook.md` at the start of every run.
- **Ownership**: exactly what this role owns (the persistence artifacts) and what it must never do.
- **Reporting & review**: who it reports to; its named counter-reviewer; the rule that its output is not DONE until that reviewer PASSes (`references/handoff-and-review.md`).
- **Superpowers wiring**: the REQUIRED sub-skills this role uses (e.g. executor → superpowers:test-driven-development; manager → superpowers:writing-plans + the Workflow orchestration pattern).
- **Stack adapters**: the stack skills named in the registry (e.g. ios-engineer → ios, swiftui, swift).
- **Handoff contract**: returns the standard handoff artifact.
- **MEMORY pointer**: reads `<persona>/MEMORY.md` at start; never edits its own SKILL.md (only HR does, via writing-skills).
- **Definition of done**: a binary checklist specific to the role.

Create an empty `MEMORY.md` next to each persona (memory tier starts empty).

### 3. Recruit a reviewer for EVERY producing role (not just executors)

Review is a seat at every layer (`references/handoff-and-review.md`). From the reviewer template, write a reviewer skill for each producer:
- `<company>-board-reviewer` — reviews the CEO's roadmap/milestone specs for coherence + spec quality before the founder sees them.
- `<company>-pm-reviewer`, `<company>-ux-reviewer`, `<company>-architect-reviewer`, `<company>-tech-lead-reviewer` — each checks its manager's artifact (use-cases / flows / design vs charter & seams / task sizing & design-fit).
- `<company>-<executor>-reviewer` — two-stage code review (spec compliance + class-closing sweep, then code quality with mutation probe).

Each reviewer: a *different* skill from its producer (independence is structural); returns `verdict: PASS | FAIL` with concrete `blocking_issues`; drives the produce→review→revise→converge loop.

### 4. Recruit the Architect and the Tech Lead as DISTINCT managers

- **Architect** (`<company>-architect`): owns `architecture.md` + `data-model.md`. Designs system shape, seams, invariants, schema. Does NOT decompose tasks or write code. Reviewed by `architect-reviewer`.
- **Tech Lead** (`<company>-tech-lead`): takes the architect's design + PM's use-cases, writes task specs, and runs the **milestone-delivery workflow** via the Workflow tool (`references/dynamic-workflows.md`) — multi-stage, pipeline by default, scaling the review panel to task stakes. Reviewed by `tech-lead-reviewer`.
- **Product Manager / UX Manager** as needed, each with its reviewer.

### 5. Recruit the QA Engineer (product quality — the second level of QA)

- **QA Engineer** (`<company>-qa-engineer`): owns `test-strategy.md` and writes/runs **acceptance + integration tests against the spec's `A#`** at the milestone Integrate phase, plus holistic cross-task code review. Distinct from the reviewers: reviewers guard the agentic workflow (anti-hallucination, spec-compliance); QA verifies the real product. Its tests are reviewed by the code-reviewer for vacuousness (mutation-probed).

### 6. Recruit the CEO and HR

- **CEO** (`<company>-ceo`): **four commands** — `review-and-plan` (iterative founder brainstorming via superpowers:brainstorming — review current situation, ask one question at a time, set/redirect the roadmap, write the result back into roadmap/charter/milestones/decisions), `report-architecture`, `report-progress`, `run-the-company`. Outcome-harnessed automation + access pre-flight (`references/automation.md`); escalates only direction-changing calls. Its plans pass `board-reviewer` before the founder. Sequences milestones inline; delegates execution to managers.
- **HR** (`<company>-hr`): the three-tier promotion loop (`references/memory-vs-persistence.md`) — behavior → SKILL.md (writing-skills, RED baseline), gotcha → `knowledge/` (referenced, deduped, budgeted), fact → a spec; **gardens the shared knowledge tier** (GC, supersede, cap) each wave.

### 7. Wire knowledge + consultation into every persona

Each persona SKILL.md must: read the relevant `company_policies/knowledge/<domain>.md` on demand (not wholesale); **consult** the owner via a structured artifact when a spec/design is ambiguous instead of guessing (`references/consultation.md`); and write feedback to its `MEMORY.md` (never edit its own SKILL.md). Create an empty `knowledge/` (with `index.md`) as part of the org's persistence.

### 8. Test before declaring hired (writing-skills Iron Law)

You are writing skills — do NOT skip testing. For each discipline-bearing persona (anything with a gate: reviewers, the CEO's escalation, HR's promotion), run a quick pressure scenario against the freshly written persona via a subagent and confirm it complies. If it rationalizes around the gate, tighten the persona and re-test. Document any new rationalization in the persona.

### 9. Update the registry & report

- Confirm `org-chart.md` matches what now exists on disk.
- Report to the founder/CEO: roles created/modified/retired, and that the company is staffed and runnable via `<company>-ceo`.

## Definition of done

- [ ] Every role in the recruiting set has a `<company>-<role>/SKILL.md` + empty `MEMORY.md`.
- [ ] Every executor has an independent `<company>-<role>-reviewer`.
- [ ] Every persona: correct frontmatter, charter+handbook pointer, ownership, reporting+review gate, superpowers+stack wiring, handoff contract, role-specific DoD.
- [ ] Managers have the Workflow orchestration pattern; CEO has the 3 commands + escalation; HR has the promotion loop.
- [ ] Discipline-bearing personas pressure-tested and compliant.
- [ ] `org-chart.md` reflects on-disk reality. No product code written.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Description summarizes the persona's workflow | Description = triggers only ("Use when…"). Workflow goes in the body. |
| Executor and its reviewer collapsed into one skill | Independence is structural — two separate skills, always. |
| Persona allowed to edit its own SKILL.md | Only HR edits personas, via writing-skills. Personas write to MEMORY.md. |
| Skipping the test step "because the JD was clear" | writing-skills Iron Law — clear to you ≠ compliant under pressure. Test gates. |
| Re-recruiting the whole org for one new role | Delta only. |
