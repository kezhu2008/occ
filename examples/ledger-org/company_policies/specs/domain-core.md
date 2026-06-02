# Domain-Core Backend Spec

> **Co-owned by Product Manager + Architect.** PM frames the required behaviour from the use-cases; the Architect makes it a precise, invariant-bearing contract. `ledger-pm-reviewer` **and** `ledger-architect-reviewer` both gate this file. Lives in `company_policies/specs/domain-core.md`.
>
> This is the **behavioural contract `ledger-backend-engineer` implements against and `ledger-qa-engineer` asserts against** — no guessing. Organized **by subsystem, not by use-case** (one `R#`/`A#` backs many use-cases).
>
> **Where things live (do not mix):** business vision → `charter.md` / `roadmap.md`; user flow → `use-cases/`; **technical invariants** (Decimal-only, Foundation-only, determinism, ordering, byte-identical replay, tax-to-the-cent) → **here**. The `Source` column at the bottom points back at real `Sources/LedgerCore` in `/Users/dev/git/ledger`.

## Key acceptance anchors

> **The stable concept → `A#` map for this spec.** Other files cite the `A#` by looking up the concept here, never by guessing a number. These `A#` are the load-bearing, executable acceptance anchors for the on-device domain. (W6/server lift anchors live in `backend-lift.md`'s own anchor table.)

| Concept | A# | One-line assertion (a test can assert this) |
|---|---|---|
| `byte-identical-replay` | **A3** | Drop any projection + replay from empty ⇒ output **byte-identical** to the pre-drop canonical bytes; a `Date()`/`UUID()` leak breaks it. |
| `tax-to-the-cent-vs-oracle` | **A1** | Full `TaxProjection` per `(entity, FY)` **== `expected_tax.json` by Decimal exact equality** (IND×SMSF, FY25×FY26); a 1¢ delta in any cell FAILs. |
| `reconcile-by-Decimal-exact-equality` | **A24** | Reconciled derived figure **== statement figure by Decimal exact equality** against an INDEPENDENT oracle; an injected **1¢ delta FAILs**. |
| `web-source==file-source-byte-identical` | **A4** | Re-ingest the same statement (file import or Flex-Web pull) ⇒ `factCount` unchanged, log digest **byte-identical**, every downstream projection stamp unchanged (one `stableSourceID` id space ⇒ source-independent bytes). |
| `per-entity-discount-correct` | **A2** | Same disposal under IND/SMSF/Company ⇒ 50% / 33⅓% / **0%**; a Company disposal taking any discount FAILs. |
| `no-float-in-money-path` | **A6** | `serialize→deserialize→re-serialize` byte-identical; canonical `"<amount> <ccy>"` `en_US_POSIX`; cross-currency `+` traps. |
| `cash-derived-signed` | **A16** | `CashProjection` over real bytes == **AUD +25,616.73 / USD +1,269.53** vs an independent dropped-legs oracle; drop the `.forexConversion` fold ⇒ RED. |
| `frozen-fx-daily-pl` | **A17** | Per-position daily P/L is **insensitive** to a change in current FX (FX frozen at snapshot); vary `currentFX` ⇒ per-security number unchanged. |
| `display-never-moves-tax` (the R8 re-lock) | **A18** | Full `TaxProjection` **byte-identical** across `{no daily-P/L}` vs `{daily-P/L+cash+snapshot}` and across two snapshots; a leak of snapshot/FX/`netCash` into a tax input ⇒ RED. |
| `div775-vs-independent-oracle` | **A19** | `div775ForexRealisedAUD` == hand-computed FRE-1/FRE-2 cost-base walk (FY26 real bytes **−$1.76 AUD**); drop a realisation event ⇒ RED. |
| `div775-no-double-count` (share-CGT byte-identical) | **A20** | Adding the Div 775 engine leaves share CGT disposals + EOY CGT cells **byte-identical**; the engine touching a CGT input ⇒ RED. |
| `foundation-only-portability` | **A25** | `LedgerCore` import graph is Foundation-only and compiles for the host target; any UI import FAILs the build. |
| `snapshot-reconstruction` | **A27** | Snapshot provider returns a per-symbol at-or-before price + per-currency FX for an explicit Sydney-midnight `Date`; a seriesless symbol degrades safe (no daily P/L, never a crash). |
| `snapshot-persistence-stable` | **A28** | A stored per-day snapshot is stable across re-reads within the same Sydney day and re-keys at the next Sydney midnight. |
| `sync-now-re-pull` | **A29** | `resyncNow()` from the stored token is FactID-idempotent, guards concurrent pulls, degrades token-free on failure, and never logs the token. |

> The flat anchors **A16/A17/A18/A19/A20** (cash, frozen-FX, byte-identical tax, Div 775, no-double-count) plus the overlay/transport anchors **A27/A28/A29** are the canonical home for the historical DPC/FXC sub-acceptance. The fold from the umbrella IDs is recorded at the bottom of this file.

## Subsystem

The **domain core** — `LedgerCore`, the Foundation-only Swift package that turns IBKR Flex data into an append-only fact log and folds that log deterministically into every read model the app and (W6) the backend serve. It **owns**: the `Fact` event atom + the closed `FactKind` taxonomy, the `Reconciler` (raw → facts), the `FactStore` (append-only canonical JSONL + `FactID` dedupe), the `LotLedger` ground state, every `Projection` (positions, cash, income, gaps, stats, optimise, tax), and the tax engines (CGT, Div 775, FX-gain, franking, FITO, option, wash-sale, EOY). It **calls out to** but does not own: the raw-bytes transport (`RawRecordSource` / `FlexTransport`), the price/snapshot overlay (`QuoteSource` / `PriceOverlay`), secret storage (`TokenStore`), and — at W6 — the storage seam (`FactStore` protocol over DynamoDB). The boundary that makes the W6 server lift "a lift, not a rebuild": this package imports **Foundation only** (no UIKit/SwiftUI/SwiftData), so it compiles verbatim into a Lambda. Display, secrets, schedule, transport, and notification live **outside** this subsystem behind seams.

## Backed use-cases

This spec is the contract behind the data-and-tax substance of the use-cases below; the user-flow/labels live in `use-cases/`, the offline-testable behaviour lives here.

- Sync / reconcile / gaps: `UC-SYNC-5`, `UC-SYNC-6`, `UC-SYNC-7`, `UC-SYNC-10`, `UC-SYNC-8`
- Positions / cash / daily P&L: `UC-POS-1`, `UC-POS-5`, `UC-POS-6`, `UC-CASH-1`, `UC-CASH-2`, `UC-TODAY-1`, `UC-TODAY-3`, `UC-HOLD-2`, `UC-HOLD-3`
- Tax: `UC-TAX-1`, `UC-TAX-3` (per-entity scope), `UC-TAX-4`, `UC-TAX-5`, `UC-TAX-6`, `UC-TAX-7`, `UC-TAX-8` (parcel policy), `UC-TAX-11` (lodged-FY pin), `UC-TAX-12`
- Config that drives the fold: `UC-CONFIG-*` (entity tax profile, Div 775 election, option treatment)

---

## Requirements (R#)

> Each `R#` is a behaviour or invariant the system MUST hold, stated **architecture-agnostic** (no file format unless the format *is* a non-negotiable invariant). One requirement, one line. Number monotonically; never renumber — append.

### Facts & the event log

R1. **Facts are the only truth.** Every read model (lot states, positions, cash, income, tax, reports) is a pure projection folded over `[Fact]`; no derived value is ever the source of any other. Stored derived state is always a rebuildable cache, never authoritative.

R2. **Append-only, content-addressed identity.** The fact log is append-only — a fact is never mutated in place; corrections arrive as new facts (e.g. a user-appended cost-basis fill, or a higher-revision restatement). A fact's identity is `hash(stableSourceID, kind)` over a length-delimited encoding (so `("a","bc") ≠ ("ab","c")`), seed-free (no per-process random hashing), making re-ingest idempotent **with no sync cursor**.

R3. **Native-only facts.** A fact stores the signed amount in the instrument's **native currency** and the raw native→AUD rate observed at the event; it NEVER stores an AUD/marked/derived value. AUD conversion happens at projection time. Absent FX (`fxAtEvent == nil`) is the FX-gap signal and is never fabricated.

R4. **Entity + account on every fact.** `entityId` and `accountId` are stamped at reconcile time on every fact (never bolted on later), because tax is grouped per taxpayer entity and accounts are isolated.

R5. **Closed event vocabulary.** `FactKind` is a closed taxonomy of 13 kinds — `trade`, `dividend`, `distribution`, `interest`, `depositWithdrawal` (capital in/out, **never assessable**), `withholding` (foreign tax split out, for FITO), `corporateAction` (split/consolidation/DRP), `optionExercise`, `optionAssignment`, `optionExpiry`, `transferIn` (may arrive basis-less → gap), `costBasisFill` (user-appended gap resolution, not reconciler-emitted), `forexConversion` (AUD↔USD cash leg, no instrument, no parcel). Every `FactKind` switch in the core is **exhaustive — no `default:`** — so a new kind forces a compile-time decision at every consumer.

R6. **Two-fact split for foreign income.** A foreign dividend emits **both** a `dividend`/`distribution` income fact **and** a separate `withholding` fact that shares the same `stableSourceID` but has a distinct `kind` (hence a distinct `FactID`), so income and the withheld foreign tax are independently foldable for FITO.

R7. **No phantom equity for cash legs.** An AUD↔USD `forexConversion` carries both signed legs (base AUD amount, quote USD amount), the commission leg, and the implied sourced rate in its attributes; it has **no instrument and produces no lot-ledger parcel** (`LotLedger.key → nil`). A conversion leg never falls through the equity path to fabricate a parcel or a data-gap; an unusable leg degrades to a flagged `forexGap`, never a silent phantom.

### Determinism & projections

R8. **Determinism (the A6 property).** A projection reads only `[Fact]` plus an explicit `ProjectionConfig`; it reads **no** `Date()`, `TimeZone.current`, `Locale.current`, `UUID()`, or randomness. Same facts + same config ⇒ **byte-identical** output. The fold's total order is `(eventDate, stableSourceID, FactID)` — day-granular, same-day events tie-break on id.

R9. **Money is exact Decimal.** Every monetary value is `Decimal` (signed, full precision) wrapped in `Money{amount, currency}`; **no binary floating-point** appears anywhere in the money path, intermediate or stored. Cross-currency `+`/`-` traps on a currency mismatch. Rounding is explicit at the edges only (bankers' / half-to-even at scale 2). Canonical money form is `"1234.50 AUD"`, `en_US_POSIX`, no grouping.

R10. **Canonical byte-stable serialization.** Facts serialize via `CanonicalJSON` with sorted keys and fixed `Decimal` formatting, and the `FactStore` rewrites the log in `(eventDate, stableSourceID)` order on every mutation, so the JSONL bytes are stable run-to-run and machine-to-machine. The log's SHA-256 `digest()` over the canonical bytes is the projection stamp's change key.

R11. **Versioned, digest-stamped, rebuildable projections.** Every projection output is stamped `(name, version, factCount, factDigest)`. A consumer reuses a cached projection only while version **and** digest match; otherwise it rebuilds by replay. Bumping a projection's `version` ⇒ rebuild-by-replay. Dropping any projection and replaying from the same facts reproduces it **byte-for-byte**.

R12. **Idempotent ingest.** Re-importing the same statement (or re-pulling the same Flex `queryId` window) appends zero new facts — the store rejects any fact whose `FactID` is already present. The total fact count is unchanged and every downstream projection is byte-identical after the re-ingest.

### Lot ledger & state machine

R13. **Per-ticker append-only lot ledger.** State is materialised as one `LotLedgerState` per `(entityId, accountId, instrumentSymbol)`, holding open `Parcel`s (oldest first) and `DisposalLine`s kept **separate** from parcels so the CGT engine can re-match parcels↔disposals under any policy.

R14. **Preserved acquisition originals.** A `Parcel`'s `quantity` and `costBaseNative` are written once and never mutated; only `remaining` decrements as the parcel is consumed. A split **supersedes** (creates a new rebased parcel flagged `splitAdjusted`) — it never edits an existing parcel.

R15. **Event → transition rules.** `trade` BUY / `transferIn` opens a parcel; `trade` SELL FIFO-consumes `remaining` and records a `DisposalLine`; `corporateAction` FS/RS supersedes (rebased) parcels; DRP-restate applies a cost-base delta to the latest open parcel; `optionExpiry` closes the option parcel; `optionExercise`/`optionAssignment` closes the option **and** opens the underlying (a cross-key mutation, premium folded into underlying cost base/proceeds); `costBasisFill` clears the gap flag.

R16. **Instrument model.** `Instrument = {symbol, kind ∈ {equity, etf, option}, currency, exchange, option: OptionSpec?, conid?}`. `OptionSpec = {underlyingSymbol, strike: Decimal, expiry, putCall ∈ {P,C}, multiplier}`. Long vs short is **the sign on `quantity`**, not a field. The OCC symbol is a fixed 21-char `root(6)+YYMMDD(6)+C|P(1)+strike×1000(8)` with exact round-trip.

### Tax (per entity)

R17. **Per-entity CGT.** The CGT engine computes per-disposal, per-parcel gains/losses under the entity's `ParcelPolicy` (FIFO / LIFO / min-gain / max-gain), applies per-event FX translation (cost at `fxAtAcq`, proceeds at `fxAtDisposal`), the **12-month discount at the per-entity rate** (Individual 50%, SMSF 33⅓%, Company **0%**), and capital **loss carry-forward**. The CGT engine reads `nativeAmount`/`fee` directly and **never** reads any display attribute (`netCash`), price mark, or daily-P/L value.

R18. **Franking + FITO.** The income engines split franked/unfranked dividends, compute the franking gross-up, and compute the foreign income tax offset from `withholding` facts paired to their dividend via the shared `stableSourceID`. `depositWithdrawal` is **excluded** from all income (it is capital).

R19. **Division 775 forex realisation (the assessable-FX engine).** A per-currency cash cost-base ledger (weighted-average default / FIFO alternative, the chosen method recorded on the result; AUD functional currency) computes FRE-1/FRE-2 forex realisation gains/losses on the foreign currency itself, over the `forexConversion` legs plus the USD-settled trade cash flows that move the USD cost base. This is **assessable revenue** (full value, Company → no CGT discount), folded into the EOY net taxable as a distinct line. It must **not double-count** the FX already embedded in the share's per-event CGT translation. The disclosed-only TR-96/8 `FXGainEngine` (share per-event translation) stays separate and is **not** the assessable engine.

R20. **Options tax.** Option treatment is **capital by default, revenue overridable per entity**; exercise/assignment link the option's premium into the underlying's cost base or proceeds; a written (short) option carries premium received.

R21. **Wash-sale flags, not adjustments.** Likely wash sales are **flagged with reasoning** (Part IVA advisory) — never silently adjusted; flagging moves no tax number.

R22. **EOY summary per entity per FY.** A per-entity end-of-financial-year summary (AU FY = 1 Jul–30 Jun) reports `netCapitalGainAUD + grossedUpDividendsAUD + (div775ForexRealisedAUD when present)`. The Div 775 line is **absent (nil key)** when zero, so the canonical EOY bytes are byte-identical to a no-Div-775 world until the figure is genuinely non-zero. A **lodged** FY is pinned to its snapshot policy and not recomputed under a later live policy.

### Display ≠ tax (the R8 re-lock)

R23. **Display never touches tax.** Price marks, the daily-P/L numbers, the per-Sydney-day snapshot, the cash projection's balances, and the trade `netCash` attribute feed positions/valuation/cash/daily-P/L **only**. No lot-ledger, CGT, income, FITO, FX-gain, Div 775, EOY, or optimise input reads any of them. Swapping the price feed or the snapshot can move market value or daily P/L, never cost base or any tax cell. The only engine that intentionally moves a tax cell is the Div 775 engine (R19).

R24. **Cash is a derived signed fold, not state.** The cash balance per `(entityId, accountId, currency)` is a pure projection folded from `depositWithdrawal` + `dividend`/`distribution`/`interest`/`withholding` `nativeAmount` + `trade` `netCash` + **both** legs and the commission of each `forexConversion`. Balances are **signed and never clamped to 0** (a negative/margin FX balance must render as negative). Cash is not a lot-ledger entity. The cash fold is version-stamped and A6-rebuildable.

R25. **Daily P/L is frozen-FX, outside the tax pipeline.** Per-position daily P/L (AUD) = `(currentMarkNative − snapshotMarkNative) × netQty × multiplier × snapshotFX` — the price move only, **insensitive** to a change in the current FX rate (FX frozen at the snapshot midnight). FX-cash daily P/L (AUD) = `currencyBalance × (currentFX − snapshotFX)`; home AUD cash is flat (0). The daily-P/L calculator is a pure function over projection **outputs** + snapshot + current overlay/FX + an explicit `asOf` — it is **not** a projection and never feeds a tax input.

### Reconciliation (exact equality)

R26. **Reconcile to the statement by exact equality.** After reconciling a Flex statement into facts, a reconciliation check compares the derived figure against the statement's authoritative figure (e.g. the cash position, the fact-count for a known fixture) by **Decimal exact equality** — a 1¢ delta FAILs. Reconciliation oracles are **independent ground truth** (a first-principles re-derivation over the real bytes), never the projection's own re-sum.

### Forward-compat / server-lift seams

R27. **Foundation-only core (the lift seam).** `LedgerCore` imports Foundation only — no UIKit, SwiftUI, or SwiftData — so the same code compiles for a non-Apple host/server target. The transport (`RawRecordSource` / `FlexTransport`), the overlay (`QuoteSource` / `PriceOverlay`), secret storage (`TokenStore`), and (W6) the persistent `FactStore` are protocol seams the core depends on but does not implement against any one backend.

R28. **Config is explicit data, not facts.** `ProjectionConfig` (per-entity `EntityDef{id, taxKind, cgtDiscount, accountIds}`, `parcelPolicy`, `perEntityOptionTreatment`, `div775Election`, `forexCostBaseMethod`, `today`, `lodgedFYs`) is byte-canonical explicit input to every fold. `EntityDef.id` MUST equal the reconciler-stamped `entityId`; an unmatched/unassigned account fails **safe** — treated as a flagged gap or at minimum the most conservative **0% CGT discount**, never a silent 50% — so a gain is never understated. A config change invalidates cached projections.

### Daily-Performance & Cash overlay (DPC — display-only, ground `requirements.md` R27–R33)

R29. **Cash balances, derived per currency (the DPC cash fold).** Cash per `(entity, account, currency)` — home AUD + each foreign currency — is DERIVED from the existing fact stream (`depositWithdrawal` + `dividend`/`distribution`/`interest`/`withholding` `nativeAmount` + `trade` `netCash` + both `forexConversion` legs + commission), since the real statement carries no `<CashReport>`. Balances are **signed and never clamped to 0** (a negative/margin FX balance, e.g. the real USD position, must render negative). The cash projection is a pure, A6-rebuildable, version-stamped fold; cash is NOT a lot-ledger/position entity. *(This is the display side of R24; R29 names the DPC user surface that R24's fold serves.)*

R30. **12am-Sydney price snapshot (reconstructed + persisted, overlay-only).** For each held instrument + held foreign currency, reconstruct the price/FX at the most-recent 00:00 Australia/Sydney instant (DST-aware AEDT/AEST) from the provider's intraday/historical series (last trade at-or-before the instant; closed-market/weekend → the prior trading-day daily close), and PERSIST it as a per-day snapshot keyed by Sydney calendar date so it is **stable through the day** and re-keys at the next Sydney midnight. The snapshot is overlay/display data, NEVER a fact (R23).

R31. **Per-position + FX-cash daily P/L, frozen-FX (D61), display-only.** Per-position daily P/L (AUD) `= (currentMarkNative − snapshotMarkNative) × netQty × multiplier × snapshotFX` — the price move only, **insensitive to a change in the current FX rate** (FX frozen at midnight). FX-cash daily P/L (AUD) `= currencyBalance × (currentFX − snapshotFX)`; home AUD cash is flat (0); the aggregate securities-FX-revaluation line (Σ Q·P0·ΔR) is surfaced so the breakdown RECONCILES to the day-over-day total AUD change. All display-only — never a tax input. *(This is the display-surface naming of the calculator R25 specifies.)*

R32. **DPC determinism preserved (D8/D66).** No `Date()`/`TimeZone.current`/`Locale.current` read inside any DPC fold or calculator; the Sydney-midnight instant, the snapshot, the current FX, and `today` are EXPLICIT inputs. Every new DPC projection/calculator is A6-rebuildable / byte-reproducible for fixed inputs. *(R32 is the DPC-scoped restatement of the R8 determinism property.)*

R33. **DPC UX-validation-first (D63).** The cash + per-position-daily-P/L + frozen-FX/FX-cash presentation passes a `persona-ui-reviewer` audit + a NON-TECHNICAL `persona-ui-critique` layperson validation BEFORE the engineering milestones build (a layperson can read the surface and understand cash, daily P/L, and the midnight/as-of framing). *(A process gate carried from `requirements.md` R33; proven by the A11-UX leg, not a byte-test — see Acceptance note.)*

### Foreign-Currency Correctness (FXC — ground `requirements.md` R34–R39; D72–D78)

R34. **Shared forex-conversion fact model (D74/D78).** Each AUD↔USD conversion (`<Trade assetCategory="CASH" symbol="AUD.USD">`) is modelled ONCE as a first-class `FactKind.forexConversion` carrying BOTH signed legs in their own currencies (AUD base `quantity`, USD quote `proceeds`), the commission (in `ibCommissionCurrency`), and the implied native→AUD rate — all SOURCED from the leg, never re-derived one from the other (D71). The PHANTOM-EQUITY path is REMOVED (no fake equity parcel/data-gap; `LotLedger.key → nil`). This single fact serves BOTH the cash projection (display) and the Div 775 engine (tax); a leg with no usable pair degrades to a flagged `forexGap`, never a silent phantom. *(R34 is the fact-model expression of R7; together they close the phantom-leg hazard.)*

R35. **Corrected cash balance (TICKET-8) — display/R8.** Folding the `.forexConversion` facts (both legs + commission) into `CashProjection` counts the previously-dropped AUD↔USD conversions: corrected **AUD +25,616.73 / USD +1,269.53** (signed), against an INDEPENDENT dropped-legs first-principles oracle (D75 — NOT a re-sum of the fold). The fix is DISPLAY only — it moves ZERO tax cell (R23 re-lock). The projection version-bumps (fold changed → rebuild-by-replay), stays signed/never-clamped, A6-rebuildable.

R36. **Division 775 forex-realisation engine (TICKET-10) — the assessable-FX mover.** A `Div775Engine` (core, Foundation-only) computes FRE-1/FRE-2 forex realisation gains/losses on the FOREIGN CURRENCY (the USD cash) itself over a per-currency cash COST-BASE ledger (AUD functional currency; weighted-average default / FIFO alternative, RECORDED on the result), over the `forexConversion` legs plus the USD-settled trade cash flows. This is ASSESSABLE REVENUE (Company → no CGT discount), a distinct EOY line (`EOYSummary.div775ForexRealisedAUD`). It must **not double-count** the FX already embedded in the share's per-event CGT translation; the disclosed-only TR-96/8 `FXGainEngine` (R11-of-W1 / the share per-event translation) stays separate and AUD→0. *(R36 is the engine-level expression of R19; the only intentional tax mover per R23.)*

R37. **Div 775 election as a user-facing account-level config input (D73).** The Div 775 election is a per-entity Config toggle read by the engine as a `ProjectionConfig` input (`div775Election: [entityId: Div775Election]`, default `.standard`). The toggle APPLIES the election (e.g. the $250k-balance election → disregard qualifying forex gains/losses) AND shows explanatory copy. The engine NEVER hardcodes the election; a config change invalidates the cached projections (`dropAll()`); determinism preserved (explicit input, D8). *(R37 specialises R28 for the Div 775 election input.)*

R38. **R8 re-lock (cash) + share-CGT byte-identical (binding).** The corrected cash balance is a DISPLAY metric — it never enters the fact log's tax semantics, the lot ledgers, or any tax/CGT/income/FXGain/EOY/optimise input; adding it moves ZERO tax cell (byte-identical, non-vacuous + mutation-proven). SEPARATELY, adding the Div 775 engine must NOT perturb the SHARE CGT numbers (the per-event translation is byte-identical before vs after); at the Div-775-wired milestone the Div 775 EOY cells move INTENTIONALLY (the genuine correction) and are re-baselined against the hand-computed oracle while share-CGT stays byte-identical. *(R38 is the FXC-scoped binding of R23.)*

R39. **Sync-now control (TICKET-9), idempotent + degrade-safe + token-free.** A foreground `resyncNow()` re-pulls the real Flex statement from the STORED credential (no re-typing), with a busy/error state, idempotent on FactID (a concurrent-pull guard), degrade-safe (no token → reconnect prompt, never a crash; a failed pull leaves prior data intact + a token-FREE error), and the token never logged (D31). *(The on-device "Sync now" requirement; W6 lifts this same control server-side — see `backend-lift.md` R12–R16/R22.)*

> Guidance: if a reviewer cannot point to an `A#` that *proves* an `R#`, the requirement is too vague — sharpen it until it is testable. (R33's gate is proven by the A11-UX process leg, not a byte-test — the one acceptance carried as a named persona-validation outcome rather than an offline assertion.)

---

## Acceptance (A#) — offline-testable; this is what tests assert

> Each `A#` is a **provable fact** a test asserts with no human judgement: a fixture/precondition, the exact operation, and the **exact expected result** (value, equality kind, error). Tolerance is **zero** unless stated. Each `A#` names the `R#`(s) it proves and the `UC`(s) it serves. Negative-control / mutation-proven where noted (a deliberate break MUST turn the test RED — a vacuous green is a reviewer FAIL).

A1. **Tax oracle, to the cent** — Given the bundled `flex_dummy.xml` (multi-account, multi-entity; ≥1 option trade, 1 assignment, 1 expiry, 1 transfer-in with missing basis), build the full `TaxProjection` per `(entity, FY)` and assert it **equals `expected_tax.json` by Decimal exact equality** across IND × SMSF and FY25 × FY26 — CGT (shares + options), franking, FITO, FX, and EOY cells. The fixture's fact-log digest is pinned (`7f6784e1…`); a 1¢ delta in any cell FAILs. — *(proves R9, R17, R18, R20, R22; serves UC-TAX-1, UC-TAX-4, UC-TAX-5, UC-TAX-6)*

A2. **Per-entity discount is correct** — For the same disposal under Individual vs SMSF vs Company entity, the discounted gain uses 50% / 33⅓% / **0%** respectively, by exact equality against the hand-derived oracle; a Company disposal receiving any discount FAILs. — *(proves R17, R28; serves UC-TAX-3)*

A3. **Replay determinism (A6)** — Drop **any** projection and replay the fact log from empty ⇒ the rebuilt output is **byte-identical** to the pre-drop canonical bytes. A `version` bump rebuilds by replay to the same content; a single non-determinism leak (`Date()`/`UUID()` inside a fold) breaks the byte equality. — *(proves R1, R8, R11; serves UC-TAX-1, UC-POS-1)*

A4. **Idempotent double-import** — Ingest `flex_dummy.xml`, record `factCount` and the log digest, ingest the **same** statement again ⇒ `factCount` is unchanged and the digest is byte-identical; every downstream projection's stamp is unchanged. — *(proves R2, R12; serves UC-SYNC-6, UC-SYNC-5)*

A5. **Content-addressed identity collision-safe** — `FactID("a","bc") ≠ FactID("ab","c")` (length-delimited), and `FactID` is stable across two processes (seed-free hash) — a property/round-trip test over N random `(stableSourceID, kind)` pairs FAILs if any collision or per-run drift appears. — *(proves R2; serves UC-SYNC-7)*

A6. **No float in the money path** — A property/round-trip test over N random amounts: `serialize → deserialize → re-serialize` is byte-identical, the canonical form is `"<amount> <ccy>"` `en_US_POSIX` no-grouping, and no value equals its `Double`-rounded form when they should differ at the cent. A `Money + Money` across mismatched currencies **traps** (or returns nil via the non-trapping variant). — *(proves R9, R10; serves UC-POS-5)*

A7. **Native-only facts; FX gap not fabricated** — Every persisted `Fact.nativeAmount` is in the instrument's native currency and no fact stores an AUD/marked value; a foreign fact with no observed rate has `fxAtEvent == nil` and surfaces as an FX gap, not a fabricated rate. AUD conversion appears only in projection output. — *(proves R3; serves UC-TAX-7, UC-CASH-1)*

A8. **Entity + account on every fact** — After reconcile, **every** fact in the log has a non-empty `entityId` and `accountId`; a fact missing either FAILs. The multi-account fixture rolls up to the correct per-entity tax grouping. — *(proves R4; serves UC-TAX-3, UC-POS-5)*

A9. **Exhaustive FactKind switches** — A compile-time guard (and a test enumerating `FactKind.allCases`) asserts every core consumer handles all 13 kinds with **no `default:`** branch; adding a 14th kind fails to compile until every switch is updated. — *(proves R5; serves UC-SYNC-7)*

A10. **Two-fact foreign dividend** — A foreign dividend row produces exactly two facts sharing `stableSourceID` — one `dividend`/`distribution`, one `withholding` — with distinct `FactID`s; the FITO engine pairs them by `stableSourceID`. — *(proves R6, R18; serves UC-TAX-6)*

A11. **Forex conversion has no parcel** — A `forexConversion` fact has `instrument == nil`, `LotLedger.key(forFact:) == nil`, and produces zero mutations / zero parcels / zero data-gaps via the equity path; an unusable leg sets `forexGap` and never fabricates a phantom equity. — *(proves R7; serves UC-CASH-1)*

A12. **Append-only correction** — After a `transferIn` with missing basis, appending a `costBasisFill` leaves the original `transferIn` fact present and unchanged; the fill is a distinct new fact that clears the gap flag, and CGT updates accordingly. — *(proves R2, R15, R28; serves UC-SYNC-10)*

A13. **Preserved parcel originals through a split** — After a buy then a corporate-action split, the original parcel's `quantity`/`costBaseNative` are unchanged and a **new** rebased parcel (flagged `splitAdjusted`) supersedes it; only `remaining` ever changed. Re-matching disposals under a different `ParcelPolicy` reproduces consistent gains. — *(proves R13, R14, R15; serves UC-HOLD-2)*

A14. **Cross-key option assignment** — An `optionAssignment` closes the option parcel **and** opens the underlying parcel under the underlying's `LotLedgerKey`, folding the per-contract premium into the underlying cost base/proceeds; the option and underlying states reconcile. — *(proves R15, R16, R20; serves UC-HOLD-3, UC-TAX-5)*

A15. **OCC symbol round-trips** — For N option specs, `parse(format(spec)) == spec` exactly (fixed 21-char layout, `strike×1000`), and long/short is recovered as the sign on `quantity`, not a field. — *(proves R16; serves UC-HOLD-3)*

A16. **Cash derived per currency, signed** — Over the real `.flex-local` fact set under the Acme Company entity, `CashProjection` yields **AUD +25,616.73 / USD +1,269.53** (signed, post the `forexConversion` fold) against the **independent dropped-legs first-principles oracle** (not the projection's own sum); a deposit-only/empty path is correctly `0` (non-vacuous). A mutation dropping the `forexConversion` fold case reproduces the pre-fix **AUD +93,627.88 / USD −47,376.22** → RED. A negative balance renders negative, never clamped to 0. — *(proves R24, R26; serves UC-CASH-1, UC-CASH-2)*

A17. **Frozen-FX per-position daily P/L** — For a USD holding, per-position daily P/L (AUD) `== (currentMark − snapshotMark) × qty × multiplier × snapshotFX` (hand-derived) and is **insensitive** to a change in the **current** FX rate (mutation-proven: vary `currentFX`, the per-security daily P/L is unchanged). FX-cash daily P/L `== balance × (currentFX − snapshotFX)`; AUD cash daily P/L `== 0`; per-position + FX-cash + the securities-FX-revaluation line `==` the day-over-day total AUD change (reconciles). — *(proves R23, R25; serves UC-TODAY-1, UC-TODAY-3)*

A18. **R8 byte-identical tax (the re-lock)** — The full `TaxProjection` (EOY cells + CGT disposals, canonical JSON) is **byte-identical** across `{no daily-P/L}` vs `{daily-P/L + cash + snapshot present}`, and across two **different** snapshots; market value / daily-P/L move, **no tax cell** moves. **Negative control:** a deliberate leak of `snapshot.mark` / `currentFX` / `netCash` into a tax input turns the test RED. — *(proves R23; serves UC-TAX-1, UC-TODAY-1)*

A19. **Division 775 against an independent oracle** — Under the standard election, `div775ForexRealisedAUD` equals a **hand-computed FRE-1/FRE-2 per-currency cost-base walk** over the real legs (independent oracle), with the chosen `forexCostBaseMethod` recorded on the result; the balance-election toggle correctly disregards qualifying gains/losses when applied; a mutation dropping a realisation event → RED. The current real-bytes FY26 figure is **−$1.76 AUD**. — *(proves R19, R28; serves UC-TAX-7, UC-TAX-12)*

A20. **Div 775 does not perturb share CGT / no double-count** — Adding the Div 775 engine leaves the full share CGT disposals + EOY CGT cells **byte-identical** (the share per-event FX translation is untouched), and the disclosed-only `FXGainEngine` stays AUD→0; a proof shows the share's embedded settlement-FX is **not** re-counted in the Div 775 line; a mutation letting the Div 775 engine touch a CGT input → RED. — *(proves R19, R23; serves UC-TAX-7)*

A21. **EOY Div-775 line absent when zero** — When the Div 775 figure is zero, the EOY summary's `div775ForexRealisedAUD` key is **absent** and the canonical EOY bytes are byte-identical to a no-Div-775 baseline; the line appears only when genuinely non-zero. — *(proves R22; serves UC-TAX-1)*

A22. **Wash-sale flag moves no number** — A constructed wash-sale scenario produces a flag with reasoning, and the CGT/EOY cells are byte-identical with the flag present vs a control where detection is disabled — flagging adjusts nothing. — *(proves R21; serves UC-TAX-4)*

A23. **Lodged-FY pin** — Marking an FY lodged then changing the live `ParcelPolicy` leaves the lodged FY's EOY cells unchanged (pinned to its snapshot policy) while an open FY recomputes; exact equality on the pinned cells. — *(proves R22, R28; serves UC-TAX-11)*

A24. **Reconcile by exact equality** — For a fixture with a known statement figure, the reconciled derived figure equals it by **Decimal exact equality**; a deliberately injected 1¢ delta FAILs. The reconciliation oracle is an independent re-derivation, not the projection's own re-sum (a self-referential oracle is a reviewer FAIL). — *(proves R26, R9; serves UC-SYNC-7)*

A25. **Foundation-only portability lock** — A test asserts `LedgerCore`'s import graph is Foundation-only (no UIKit / SwiftUI / SwiftData) and the package compiles for the host (non-iOS) target; adding any UI import FAILs the build. — *(proves R27; serves UC-POS-1, UC-TAX-1)*

A26. **Safe-fallback for an unassigned account** — A fact whose `accountId` matches no `EntityDef.accountIds` is treated as a flagged gap / **0% CGT discount**, never a silent 50%; the resulting gain is **≥** the gain any valid assignment would yield (never understated). A config change invalidates the cached projections (next read rebuilds). — *(proves R28; serves UC-TAX-3, UC-SYNC-10)*

### Overlay & transport acceptance (A27–A29) — flat anchors with no DPC/FXC tax equivalent

> These three flat anchors cover overlay-provider, overlay-persistence, and control/transport behaviour that has no tax-side flat equivalent above. They are cited by older files (`use-cases/index.md`, `knowledge/`, `handbook.md`, `milestones/W6.md`).

A27. **Snapshot reconstruction** — Given an explicit 00:00-Sydney `Date`, the snapshot provider returns a per-symbol price at-or-before that instant (intraday bar when open; prior daily close when closed/weekend) and a per-currency snapshot FX; a symbol with no series degrades safe (no snapshot → that row shows no daily P/L, never a crash/fabrication). — *(proves R30; serves UC-DAILY-PL, UC-SNAPSHOT-SCHED; overlay-provider behaviour)*

A28. **Snapshot persistence stability** — The persisted per-day snapshot is stable across re-reads within the same Sydney day (re-fetching does not change a stored day's baseline) and re-keys at the next Sydney midnight. — *(proves R30; serves UC-SNAPSHOT-SCHED; overlay-persistence behaviour)*

A29. **Sync-now re-pull** — A `resyncNow()` from the stored token dedupes on FactID (idempotent, A5), the concurrent-pull guard holds, no-token degrades to the reconnect prompt, a failed pull leaves prior data intact + a token-free error; the token never appears in any log (grep-clean, D31). — *(proves R39, R12; serves UC-RESYNC-NOW, UC-CONN, UC-TRIGGER-SYNC; control/transport behaviour, also lifted server-side in `backend-lift.md` A11)*

> **Tracing checklist (reviewers enforce):**
> - [ ] Every `R#` (R1–R39) is proven by ≥1 `A#` (R33 by the A11-UX process leg).
> - [ ] Every `A#` names its `R#`(s) and ≥1 `UC` (or an explicit named invariant).
> - [ ] Every `A#` is assertable with **no human judging** — exact value/equality/error, tolerance stated (zero).
> - [ ] Every equality/reconciliation `A#` uses an **independent oracle**; every risk `A#` is **mutation-proven** (negative control turns RED).
> - [ ] No business vision or user-flow prose leaked in (those live in `roadmap.md` / `use-cases/`).

---

## Out of scope / non-goals

- **Storage backend & multi-tenancy (W6).** The DynamoDB single-table schema, tenant isolation from the verified JWT, conditional-put idempotency, and the snapshot-freeze are the `system-design.md` (TECH-LEAD) contract, behind the `FactStore` seam — this spec only requires the seam (R27) and the byte-identical-replay property (R11) that the lift preserves.
- **Transport, secrets, schedule, notify, API.** Flex Web Service pull mechanics, Keychain/Secrets-Manager token storage, EventBridge scheduling, SNS→APNs, and the read API + Cognito are seams/other subsystems, not the domain core.
- **UI / presentation.** Labels, layout, render chokepoints, accessibility — `ux/` + the iOS spec.
- **Quote/snapshot provider internals.** Yahoo endpoints, symbol mapping, Sydney-midnight reconstruction — the overlay subsystem; this spec only requires they stay display-only (R23) and that the daily-P/L math is frozen-FX (R25).
- **Multi-source reconciliation (Layer-2 economic fingerprinting).** A documented future extension; this spec's reconciliation (R26) is single-source by construction (one IBKR `stableSourceID` id space).

> Historical DPC/FXC sub-acceptance from the real `requirements.md` (umbrella A11/A12) are folded into the flat anchors here: cash A16, frozen-FX A17, byte-identical-tax A18, Div775 A19, no-double-count A20, snapshot A27/A28, sync-now A29.

> Also note the older suffixed **process** legs `A11-UX` / `A12-UX` (the layperson-validation gate, R33/R39) and `A11-visual` / `A12-visual` (the boss's on-device confirmation leg) are **not** offline `A#` assertions — they are named persona/boss outcomes carried from `requirements.md`, not renamed here.

## Open questions

- **OQ1 — Restatement/supersede.** Identity is `(stableSourceID, kind)`, so an IBKR restatement of a row (same id+kind, corrected amount) is currently deduped away. Adding `revision: Int = 0` + `supersedes: FactID?` (highest revision authoritative, predecessors retained for audit) is additive and version-bumping but **not yet locked** — it blocks any `A#` asserting restatement behaviour. (Boss decision; tracked as OD-DM1 in `data-model.md`.)
- **OQ2 — Accountant ratification of the Div 775 all-USD-disposals scope** (every USD-share purchase as an FRE-2 event) before a FY that sells a USD share or carries USD income. Until ratified, A19's oracle is pinned to the current real-bytes walk.

## Source

`LedgerCore` in `/Users/dev/git/ledger`: facts `Sources/LedgerCore/Facts/{Fact,FactKind,FactStore,FactLog,Reconciler}.swift`; ids/money `Sources/LedgerCore/Support/{Ids,Money,CanonicalJSON,ProjectionConfig,FY,Scope}.swift`; lot ledger `Sources/LedgerCore/LotLedger/*` + `Sources/LedgerCore/Mutations/FactInterpreter.swift`; projections `Sources/LedgerCore/Projection/Models/{Positions,Cash,Income,Gaps,LedgerStats,Optimise,Tax}Projection.swift` + `Projection/DailyPLCalculator.swift`; tax engines `Sources/LedgerCore/Tax/{CGTEngine,Div775Engine,FXGainEngine,FrankingEngine,FITOEngine,OptionTaxEngine,WashSaleDetector,EOYSummary,DiscountRule,ParcelPolicy}.swift`. Acceptance harnesses: `Tests/LedgerCoreTests/{TaxAcceptanceTests,RebuildabilityTests,PortabilityLockTests,Div775AcceptanceTests,DailyPLR8RelockTests}.swift` + `Fixtures/{flex_dummy.xml,expected_tax.json}` (digest `7f6784e1…`). The developer's TDD unit tests and QA's acceptance/integration tests both point back to the `A#` ids above.
