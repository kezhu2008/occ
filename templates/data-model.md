# <Company> Data Model

> Owned by the **Architect**. The authoritative type/schema design. Reviewed by `architect-reviewer`. The executors implement against this; the reviewers check code against it.

## Core types
| type | fields (name: type) | notes / invariants |
|------|---------------------|--------------------|
| <Entity> | <id: UUID, amount: Decimal, …> | <e.g. amount never Double; immutable> |

## Relationships
<How the types relate — ownership, references, cardinality.>

## Persisted shape
<On-disk / on-wire representation. Serialization rules (ordering, formatting, versioning/migration).>

## Derived / projected state
<What is computed vs stored; rebuild rules and idempotence guarantees.>

## Validation rules
<Constraints enforced on construction/ingest; what an invalid record means.>
