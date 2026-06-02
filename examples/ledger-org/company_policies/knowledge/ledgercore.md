# Knowledge — ledgercore
Budget: ≤ ~150 lines / ≤ ~12 entries. HR-gardened: on add, dedup + merge + supersede.
Authoritative sources: `specs/domain-core.md` R8·R9·R10·R27 / A3·A6·A25; `specs/backend-lift.md` R1·R4 / A1; `decisions.md` D7/D8; data-model.md §1·§8; `Sources/LedgerCore/`.

## The fold is deterministic — no Date()/UUID()/Locale.current inside Projection.build
- Map: a `Projection` reads only `[Fact]` + an explicit `ProjectionConfig`; `today`, FX, and the Sydney-midnight instant are threaded in as inputs, never read from the environment.
- Gotcha: one hidden `Date()`/`UUID()`/`Locale.current`/random read inside a fold makes the replay digest drift run-to-run — silently, until a byte-compare on another machine fails.
- Lesson: when adding a fold or calculator, take every clock/locale value as an explicit parameter; prove byte-identical replay before merging.
- Ref: `specs/domain-core.md` R8 (determinism) / A3 (replay byte-identical); `decisions.md` D8; data-model.md §1 (I4)
- Added: 2026-05-31 by HR

## Money is Decimal, never Double; the JSONL bytes are canonical and stable
- Map: every monetary value is `Money{amount: Decimal, currency}`; facts serialize via `CanonicalJSON` (sorted keys, fixed `Decimal` form), and the log re-orders to `(eventDate, stableSourceID)` on every mutation.
- Gotcha: a `Double` anywhere on the money path, or a non-canonical encode, breaks both the to-the-cent tax oracle and the byte-stable `digest()` change-key — a cent of drift compounds across a parcel ledger.
- Lesson: keep all arithmetic in `Decimal`; round only at the edges; never hand-format money — go through `Money.canonicalString` / `CanonicalJSON`.
- Ref: `specs/domain-core.md` R9 (exact Decimal) + R10 (canonical serialize) / A6 (no-float round-trip); `decisions.md` D7; data-model.md §8
- Added: 2026-05-31 by HR

## Foundation-only is the W6 lift seam — guarded by PortabilityLockTests
- Map: `LedgerCore` imports Foundation only (no UIKit/SwiftUI/SwiftData), so the same package compiles into a Lambda; transport/overlay/secrets/`FactLogStore`/`ProjectionStore` are protocol seams the core depends on but does not implement.
- Gotcha: an accidental Apple-UI import (or a backend-specific type) leaking into the core makes it un-liftable — the W6 server build stops being "a lift, not a rebuild".
- Lesson: keep new core types Foundation-only and behind the seam protocols; let `PortabilityLockTests` fail the build on any UI import rather than catch it later.
- Ref: `specs/domain-core.md` R27 (Foundation-only) / A25 (portability lock); `specs/backend-lift.md` R1·R4 / A1 (seam preserves replay)
- Added: 2026-05-31 by HR
