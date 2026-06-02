# Artifact Layout — where everything lives

One canonical layout. Both OCC capabilities and every persona obey it, so any persona can find any artifact without being told.

## Two roots

```
<company_repo>/
├── company_policies/        # PERSISTENCE — versioned, founder-facing, git-tracked
└── .claude/skills/          # the ORG — invokable persona skills (+ co-located MEMORY)
```

`company_policies/` was chosen as the founder-named home for strategic/meta artifacts. `.claude/skills/` is where Claude Code auto-discovers project-scoped skills, so recruited personas there are invokable via the Skill tool by their `name`.

## `company_policies/` (full)

```
company_policies/
├── charter.md                 # business intent, non-negotiables, success definition
├── org-chart.md               # THE REGISTRY (see below)
├── handbook.md                # handoff schema, review bar, escalation rule (the shared contract)
├── job-descriptions/
│   └── <role>.md              # one per role — the recruiting spec
├── roadmap.md                 # milestones + success criteria (CEO)
├── milestones/<id>.md         # per-milestone spec
├── tasks/<id>.md              # per-task spec + acceptance (manager)
├── decisions.md               # autonomous + resolved-escalated decisions
├── use-cases/
│   ├── index.md               # catalog of shipped use cases (PM)
│   └── <area>.md              # per-area use-case manuals
├── ux/                        # UX manuals / screen specs (UX Manager)
└── performance/<date>.md      # HR reviews + failure-mode catalog
```

## `org-chart.md` — the persona registry (the key v2 upgrade)

v1's weakness was informal persona chains as free strings. v2 makes the org a **queryable registry**. Every row is one role:

```md
| role | skill name | reviewer | reports to | owns | stack adapters |
|------|-----------|----------|-----------|------|----------------|
| ceo | ledger-ceo | founder | founder | charter, roadmap | — |
| tech-lead | ledger-tech-lead | ledger-tech-lead-reviewer | ceo | tasks/, architecture | — |
| product-manager | ledger-product-manager | ledger-pm-reviewer | ceo | use-cases/ | — |
| backend-engineer | ledger-backend-engineer | ledger-backend-engineer-reviewer | tech-lead | core code | swift |
| ios-engineer | ledger-ios-engineer | ledger-ios-engineer-reviewer | tech-lead | app code | ios, swiftui |
| hr | ledger-hr | — | ceo (between waves) | performance/, persona SKILL edits | — |
```

This registry is the single source of truth for: which skills exist, who reviews whom, the reporting line, and which superpowers stack-skills each executor pulls in. Recruiting reads/writes it; managers consult it to build persona chains.

## `.claude/skills/` (the org)

```
.claude/skills/
├── <company>-ceo/
│   ├── SKILL.md               # persistent role instructions
│   └── MEMORY.md              # MEMORY tier — role feedback, HR-promoted
├── <company>-tech-lead/{SKILL.md, MEMORY.md}
├── <company>-<executor>/{SKILL.md, MEMORY.md}
├── <company>-<executor>-reviewer/{SKILL.md, MEMORY.md}
└── <company>-hr/{SKILL.md, MEMORY.md}
```

Naming: `<company>-<role>` (e.g. `ledger-ceo`). The prefix scopes the org to its repo and avoids collision with global `persona-*` skills and other companies' orgs. Reviewers are `<company>-<role>-reviewer`.

## Persistence vs memory — quick map

| If it is… | it goes to… |
|-----------|-------------|
| a versioned fact, spec, decision, or catalog | `company_policies/…` |
| short-term feedback about how one role behaves | `<persona>/MEMORY.md` |
| a promoted, durable behavior change for a role | that persona's `SKILL.md` (HR, via writing-skills) |

See `memory-vs-persistence.md` for the promotion loop.
