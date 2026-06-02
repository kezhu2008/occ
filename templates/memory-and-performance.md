# Memory & Performance templates

Two artifacts in the feedback loop. See `references/memory-vs-persistence.md`.

---

## `<persona>/MEMORY.md` (MEMORY tier — co-located with the persona skill)
```md
# MEMORY — <company>-<role>
> Short-term, role-scoped feedback. HR promotes durable lessons into SKILL.md. The persona reads this every run but never promotes its own.

## <date> — from <reviewer|manager|founder> (<task_id>)
<what to do differently>
Source: <…> | Severity: low|medium|high | Recurrence: <1st|2nd|…>
[promoted → SKILL.md on <date>]   ← HR appends this when promoted
```

---

## `company_policies/performance/<date>.md` (PERSISTENCE tier — HR-owned)
```md
# Performance Review — <date> (<wave/milestone>)

## Per-persona assessment
| persona | tasks | first-pass | reworks | blockers | notes |
|---------|-------|-----------|---------|----------|-------|
| <company>-backend-engineer | 7 | 5 | 2 | 0 | strong TDD; weak on FX edge cases |

## Failure modes this wave (the catalog — queryable across runs)
- **FM-<n> (<date>)**: <name>. What happened, root cause, the structural/behavioral fix.

## Promotions (memory → skill)
| persona | lesson | RED baseline reproduced? | edit applied | result |
|---------|--------|--------------------------|--------------|--------|
| <…> | <…> | yes | <file+change> | verified |

## Rejected / deferred
- <lesson> — one-off, left in memory.
```
