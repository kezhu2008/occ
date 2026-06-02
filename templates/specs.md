# Spec templates — roadmap, milestone, task

Spec-driven is non-negotiable: CEO owns roadmap + milestone success criteria; managers own task specs + acceptance. No execution without a written spec to review against.

---

## `roadmap.md` (CEO-owned)
```md
# <Company> Roadmap

## Vision
<one line>

## Milestones
| id | title | success criteria (observable) | state |
|----|-------|------------------------------|-------|
| M1 | <…> | <how we KNOW it's done> | done |
| M2 | <…> | <…> | in-progress |
| M3 | <…> | <…> | pending |

## Locked decisions
<founder-locked scope/strategy this roadmap assumes>
```

---

## `milestones/<id>.md` (CEO/manager-owned)
```md
# <id> — <title>
Goal: <what this milestone delivers>
Success criteria: <observable, testable>
Use cases covered: <UC ids from use-cases/>
Tasks: <T1..Tn — links to tasks/<id>.md>
Founder prerequisites (frontloaded): <keys/signups/permissions needed>
State: pending | in-progress | in-review | done | blocked
```

---

## `tasks/<id>.md` (manager-owned)
```md
# <id> — <title>
Goal: <single, context-sized unit of work>
Spec: <exactly what to build — the reviewer checks against THIS>
Acceptance criteria: <binary checks; what the test must prove>
Persona chain: <executor> → <executor>-reviewer  (high-stakes: + panel lenses)
High-stakes: yes | no  (drives review panel size)
State: pending | in-progress | in-review | done | blocked
```
