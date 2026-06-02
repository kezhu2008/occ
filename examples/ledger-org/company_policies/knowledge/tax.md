# Knowledge — tax
Budget: ≤ ~150 lines / ≤ ~12 entries. HR-gardened: on add, dedup + merge + supersede.
Authoritative sources for this domain: `specs/domain-core.md` R17–R22 (CGT/income/Div775) + R34–R38 (FXC) / A1·A2·A10·A19·A20; `decisions.md` D74/D84; architecture.md §"The R23 lock"/§"Decision pointers"; data-model.md §9; `Sources/LedgerCore/Tax/`.

## Two FX engines — assessable forex routes through Div775Engine, NOT the CGT engine
- Map: USD-cash forex realisation (FRE-1/FRE-2) is computed by `Div775Engine`; the share per-event CGT FX translation (cost@fxAtAcq vs proceeds@fxAtDisposal) lives in `CGTEngine`; the disclosed-only TR-96/8 `FXGainEngine` stays separate and AUD→0.
- Gotcha: a USD movement that skips `Div775Engine` — or is double-counted because the share's embedded settlement-FX is re-added to the Div 775 line — silently mis-states assessable income. No error, wrong number. (Div 775 is the ONLY engine that intentionally moves tax — D84.)
- Lesson: when touching any currency-crossing path, confirm it enters `Div775Engine` via the `.forexConversion` quote legs + every USD-settled `.trade` netCash×fxAtEvent; never re-route it through `CGTEngine`/`FXGainEngine`, and keep share-CGT byte-identical (A20).
- Ref: `specs/domain-core.md` R36/R19 + A19 (oracle) / A20 (no-double-count); `decisions.md` D84; architecture.md §"Decision pointers" (`Div775Engine.swift`, `CGTEngine`); data-model.md §9
- Added: 2026-05-31 by HR (from ledger-architect MEMORY FXC)

## Div 775 election is config (per-entity), never hardcoded; a config change MUST drop the cache
- Map: the election is `ProjectionConfig.div775Election[entityId]` (default `.standard`); the engine reads it as an explicit input (I4 determinism), the Settings toggle writes it.
- Gotcha: hardcoding the election, or editing config without invalidating projections, leaves stale tax cells — the `$250k-balance` election silently never applies (or applies after it's toggled off).
- Lesson: read the election from config only; on any config edit call `dropAll()` so the toggle recomputes cleanly (D22).
- Ref: `specs/domain-core.md` R37 (election as config) + R28 (config invalidates cache) / A19; `decisions.md` D73/D22; data-model.md §9 (`ProjectionConfig`, `Div775Election`)
- Added: 2026-05-31 by HR (from ledger-architect MEMORY FXC)

## Entity discount comes from EntityDef, and an unmatched account must fail SAFE to 0%
- Map: CGT discount is STORED on `EntityDef.cgtDiscount` (Individual 50% / SMSF 33⅓% / Company 0%), keyed by the D39 join `EntityDef.id == Fact.entityId`; `CGTEngine` applies it via `DiscountRule`.
- Gotcha: an unrecognized/unassigned real account that defaults to the 50% individual discount UNDER-states the gain — the worst-possible silent error (I8/I9). Company forex is assessable revenue with NO discount.
- Lesson: never derive the discount; on an unmatched entity degrade to a flagged data-gap or the most-conservative 0%, never a silent 50%.
- Ref: `specs/domain-core.md` R17 (per-entity CGT) + R28 (safe fallback) / A2 (discount correct) · A26 (unassigned → 0%); data-model.md §1 I8/I9, §9 (`EntityDef.cgtDiscount`); `decisions.md` D33/D39
- Added: 2026-05-31 by HR (from ledger-architect MEMORY FXC)

## Foreign tax (FITO) pairs via the two-fact rule on stableSourceID, not by re-reading the dividend
- Map: a foreign dividend emits BOTH a `.dividend`/`.distribution` income fact AND a separate `.withholding` fact sharing `stableSourceID` (distinct `FactID` via `kind`); `FITOEngine` pairs them through `attributes["dividendSource"]`; `IncomeProjection` does franking gross-up.
- Gotcha: treating withholding as part of the dividend fact (or keying FITO off the wrong id) drops the offset — foreign tax is silently forfeited, or franking is double-counted.
- Lesson: keep withholding a first-class second fact; pair on `stableSourceID`/`dividendSource`, never collapse the two-fact rule.
- Ref: `specs/domain-core.md` R6 (two-fact split) + R18 (franking + FITO) / A10 (two-fact dividend); data-model.md §4.1 (two-fact rule), §4.2 (`dividendSource`)
- Added: 2026-05-31 by HR (from ledger-qa-engineer MEMORY FXC)
