# Knowledge — ingest
Budget: ≤ ~150 lines / ≤ ~12 entries. HR-gardened: on add, dedup + merge + supersede.
Authoritative sources: `specs/domain-core.md` R2·R5·R7·R12·R34 / A4·A5·A11; `decisions.md` D70/D71/D74; data-model.md §4·§6·§13; `use-cases/onboarding-and-sync.md`; `Sources/LedgerCore/Facts/`.

## FactID = hash(stableSourceID, kind) — re-import is a content-addressed no-op
- Map: ingest is Flex XML → `RawRecord` → `Reconciler` → `[Fact]`; the `FactStore` rejects any fact whose `FactID` is already present, so a re-pull / overlapping `queryId` window appends zero facts with no sync cursor.
- Gotcha: assuming a since-date cursor (or keying identity off anything but the length-delimited `(stableSourceID, kind)`) re-imports duplicates or drops a real correction — `("a","bc")` must stay distinct from `("ab","c")`.
- Lesson: let FactID dedup absorb the overlap; a correction arrives as a NEW fact (e.g. `costBasisFill`), never an in-place edit.
- Ref: `specs/domain-core.md` R2 (content-addressed) + R12 (idempotent ingest) / A4·A5; data-model.md §6; `use-cases/onboarding-and-sync.md` (UC-SYNC-6)
- Added: 2026-05-31 by HR

## Use the IBKR <Trade netCash> verbatim — never re-derive it from nativeAmount − fee
- Map: the trade cash leg is the `attributes["netCash"]` display value taken verbatim from the IBKR `<Trade netCash>`; it feeds `CashProjection`/positions only, never a tax input.
- Gotcha: re-deriving the leg as `nativeAmount − fee` makes the cash oracle a re-sum of the projection's own fold (not independent) and quietly diverges from the broker's figure.
- Lesson: source each leg from the statement, never re-compute one leg from another; keep `netCash` strictly display-side behind the R8 firewall.
- Ref: `specs/domain-core.md` R23 (display ≠ tax) / A16 (cash signed); `decisions.md` D70/D71; data-model.md §4.2
- Added: 2026-05-31 by HR

## .forexConversion is the shared R34 fact — no instrument, no parcel (phantom-equity path is REMOVED)
- Map: each AUD↔USD conversion is ONE `FactKind.forexConversion` carrying both signed legs + commission + the implied sourced rate; `LotLedger.key → nil`; it serves both `CashProjection` (display) and `Div775Engine` (tax).
- Gotcha: routing a conversion leg through the equity path (the old phantom-equity behaviour, now gone) fabricates a fake parcel or data-gap; an unusable leg must degrade to a flagged `forexGap`, never a silent phantom.
- Lesson: model the conversion as the single `.forexConversion` fact, both legs SOURCED; on no usable pair, flag `forexGap`.
- Ref: `specs/domain-core.md` R7 + R34 (shared forex-conversion fact model) / A11 (no parcel); `decisions.md` D74; data-model.md §13
- Added: 2026-05-31 by HR
