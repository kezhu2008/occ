# <Company> Architecture

> Owned by the **Architect**. The authoritative current-state technical design. Reviewed by `architect-reviewer` against the charter non-negotiables and future-wave seams. Version-stamped; supersedes per-wave design notes.

## System shape
<The major components and how data/control flows between them. A diagram in ASCII.>

## Technical invariants
> The engineering realizations of the charter's business non-negotiables. These are what reviewers enforce.
- <e.g. money represented as `Decimal`, never `Double` — realizes "correct to the cent">
- <e.g. core module imports no UI framework — realizes "server-lift seam" / portability>
- <e.g. deterministic serialization (sorted keys, fixed formatting)>

## Determinism / concurrency contract
<What must be deterministic; how shared mutable state is made safe; strict-concurrency expectations.>

## Persistence strategy
<How state is stored and rebuilt. Pointer to `data-model.md` for the shapes.>

## Seams for future waves
> Swappable boundaries that keep later waves non-blocking.
| seam | today | future wave |
|------|-------|-------------|
| <e.g. RawRecordSource> | <XML file> | <Web service / server> |

## Decision pointers
<Links to the load-bearing entries in `decisions.md`.>
