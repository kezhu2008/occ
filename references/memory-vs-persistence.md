# Storage Tiers — persistence, knowledge, memory (and the HR loop)

The org writes three kinds of thing. Keeping them in separate tiers is what lets the company improve over time without polluting its strategic record or blowing up into unreadable documentation.

## The three tiers

| Tier | Lives in | Lifespan | Holds | Owner |
|------|----------|----------|-------|-------|
| **Persistence** | `company_policies/` | whole project, git-tracked | the authoritative FACTS: charter, org-chart, roadmap, architecture, data-model, **specs (R#/A#)**, use-cases, tasks, decisions, performance | each role owns its artifacts |
| **Knowledge** | `company_policies/knowledge/` | until superseded | the shared MAP + GOTCHAS + LESSONS that make the org repo-familiar — pointers, never restated facts | HR gardens; everyone reads |
| **Memory** | `<persona>/MEMORY.md` | until HR triages | short-term role feedback (reviewer/manager/founder corrections) | the persona accumulates; HR triages |

## The deciding test

> - An authoritative FACT/spec/decision everyone obeys → **PERSISTENCE**.
> - A navigational MAP or hard-won GOTCHA that helps you work the repo → **KNOWLEDGE** (shared).
> - A short-term correction about how ONE role behaved → **MEMORY** (that persona).

## Knowledge tier — shared, and deliberately small

Knowledge is what makes developers and leads "more familiar with the repo as they develop." It is **shared** (one store, not per-persona) so a lesson learned once helps everyone. The risk is that it balloons into huge documentation; these rules prevent that:

1. **Reference, never restate.** A knowledge entry holds only the *map* and the *gotcha*, and **points to the authoritative artifact** (`spec R12`, `architecture.md §7.2`, `Parcel.swift:85`). If the content is a fact, it belongs in a spec, not in knowledge. Knowledge that duplicates a spec is deleted.
   - ✅ "FX gains route through `Div775Engine`, not the CGT engine — see `architecture.md §6` / `Div775Engine.swift:40`. Gotcha: a USD movement missed here silently under-reports tax."
   - ❌ (restating the whole FX algorithm — that's the spec's job.)
2. **Modular + indexed.** Per-domain files (`knowledge/<domain>.md`) with a one-line index (`knowledge/index.md`). A persona loads only the domain(s) it touches — knowledge never loads wholesale into context.
3. **Depth vs breadth, same store.** Executor entries are deep/narrow (one domain's traps); lead/architect entries are broad/high-level (cross-domain map, where things live). Different readers, one indexed store.
4. **Hard budget + GC.** Each `knowledge/<domain>.md` has a size cap (e.g. ≤ ~150 lines). When HR adds, HR also **dedups, merges, and supersedes** stale entries. Every wave HR runs a knowledge GC; an entry whose referenced artifact moved/changed is updated or pruned. Growth without pruning is a failure mode HR is accountable for.

## Memory tier — short-term, role-scoped

Raw feedback co-located with the persona. Inert until HR triages. Example:
```
## 2026-05-29 — from ledger-backend-engineer-reviewer (M4-T2)
Left sample data in a `default` object on a retire task. On retire/remove, grep the whole core, not just the obvious file.
Source: reviewer | Severity: medium | Recurrence: 1st
```

## The HR loop — three destinations

HR is the only function that promotes. It reads each persona's MEMORY + performance evidence and routes each lesson to its right tier:

```
reviewers/managers/founder ──► <persona>/MEMORY.md (raw)
                                     │  HR (between waves)
        ┌────────────────────────────┼───────────────────────────────┐
        ▼                            ▼                                ▼
 a BEHAVIOR fix for one role   a durable KNOWLEDGE/gotcha       a FACT/contract everyone obeys
        │                            │                                │
 edit persona SKILL.md         add to knowledge/<domain>.md      add to a spec / architecture /
 via superpowers:writing-skills (reference the artifact,         decisions (PERSISTENCE);
 (RED baseline FIRST)           dedup + respect the budget)      route through that artifact's review
        │                            │                                │
        └────────── annotate the MEMORY entry "promoted → <dest> on <date>"; log in performance/<date>.md ──────────┘
```

Rules:
- **No SKILL.md edit without a RED baseline** (writing-skills Iron Law). Promote only recurring/high-severity behavior.
- **Knowledge promotion obeys the anti-bloat rules above** — reference + dedup + budget + GC. HR that lets knowledge grow unbounded has failed its own job.
- After promotion, annotate the memory entry and log the promotion (and rejections) in `performance/<date>.md` so the loop is auditable and not repeated.
