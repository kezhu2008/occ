# Instruction Manual — Positions, Cash & Daily P&L

Tap-by-tap user instructions for every Positions / Holdings / Cash / Daily-P&L use case the
**current** app supports, named under the canonical `UC-<AREA>-<n>` scheme. Grounded in the live
code — not the stale `ui-review/UI-MANUAL.md`. Index: [`use-cases/index.md`](index.md). Backing
contracts (R#/A#): [`company_policies/specs/domain-core.md`](../specs/domain-core.md) (these are the
on-device rows, so every `R#/A#` below resolves in `domain-core.md`'s flat namespace, never
`backend-lift.md`).

> **No demo/sample-data path in this area** (post-D85). Rows come from the live `PositionsProjection`
> at the injected scope. The TODAY hero, TODAY column, and CASH hero/section always render — an empty
> state shows a neutral "—" or the "baseline isn't ready" banner, never a fabricated $0, never red.

> **Where the numbers come from (read before trusting a figure).** Market value, price marks, the
> daily-P/L numbers, the midnight snapshot, and cash balances are **display/overlay** outputs — they
> never feed any tax cell (the R8 re-lock, `domain-core.md` R23 / A18). Cash is a pure signed fold
> (R24); daily P/L is frozen-FX (R25). Cost base and tax live in [`tax.md`](tax.md).

---

## UC-POS-1
**View holdings list (ticker/qty, price, market value, today's change, lifetime P/L)**

- **Goal:** See every holding in the active scope as a table with ticker/qty, price, market value, today's change, and lifetime P/L.
- **Actor / preconditions:** A user on the POS tab; ingest + overlay/FX wiring finished (`loaded == true`). At least one onboarded, connected account.
- **Entry point:** Bottom tab bar → **POS** → the scrollable Positions screen.
- **Steps (action → see-that):**
  1. Tap the **POS** tab. → the summary block, the filter/group row, the column header `TICKER · QTY | PX | MKT VAL | TODAY | TOTAL P/L`, then holding rows.
  2. Read each row. → line 1 = ticker + exchange (or option underlying + strike/put-call + expiry) with any GAP/WRITTEN badge; line 2 = qty + masked account (e.g. `U7*…4192`), or option terms (`N ct · ×100 · USD`).
  3. Read the price + valuation cells. → PX (native) with a DELAYED day-change or CLOSE spark; MKT VAL + weight %; TODAY (since-midnight) delta; TOTAL P/L (lifetime unrealised) delta.
- **Outcome:** The user reads their scoped holdings and per-row valuation at a glance; the list is a pure `PositionsProjection` fold, source-independent (web pull == file import yield the same rows, A4).
- **Variations & edge states:** No rows for the filter → "No positions match this filter". A quote-pending symbol → "—" for PX/MKT VAL (never a fabricated mark). At Dynamic-Type AX1 the TODAY/TOTAL P/L dollar figures compact (`+$1.3k`) and % sub-lines drop; columns are never removed. A projection build failure → a "COULD NOT LOAD POSITIONS" panel, never stale data labelled fresh.
- **Backed by spec:** R8, R25, A25 (index anchors for UC-POS-1; `LedgerCore` is Foundation-only/portable per A25, the projection is a deterministic fold per R8).
- **Source:** PositionsView.swift:298 + PositionRow.swift:62.

## UC-POS-2
**Filter holdings by asset class (ALL · EQ · ETF · OPT)**

- **Goal:** Restrict the list to a single asset class — all, equities, ETFs, or options.
- **Actor / preconditions:** A user on the POS tab.
- **Entry point:** POS screen → the class chip row (**ALL · EQ · ETF · OPT**).
- **Steps (action → see-that):**
  1. Tap a chip `ALL` / `EQ` / `ETF` / `OPT`. → the active chip highlights.
  2. Watch the list. → rows re-filter immediately (no reload).
  3. Read the totals. → the summary totals and each row's weight % recompute over the filtered set.
- **Outcome:** The list, totals, and weights reflect only the chosen class.
- **Variations & edge states:** A filter matching nothing → "No positions match this filter". The cost-basis GAP count in the price-source row is computed over the scoped set **before** the class filter, so it does not change with the chip.
- **Backed by spec:** R16 (index anchor for UC-POS-2 — the `Instrument.kind ∈ {equity, etf, option}` model the class filter reads).
- **Source:** PositionsView.swift:284 + PositionsViewModel.swift:338.

## UC-POS-3
**Group positions (Flat / by account / by entity) with subtotals**

- **Goal:** Reorganise holdings into flat order, or per-account / per-entity sections with subtotals.
- **Actor / preconditions:** A user on the POS tab.
- **Entry point:** POS screen → the `GROUP ▾` drop-menu pill.
- **Steps (action → see-that):**
  1. Tap `GROUP ▾`. → a menu with `Flat`, `By account`, `By entity` (active checkmarked).
  2. Pick `By account` / `By entity`. → inline group-header strips (not sticky): an uppercase accent label + "sub · count" + a trailing AUD subtotal.
  3. Read the buckets. → rows bucketed under each header, sorted within each group by the active sort.
- **Outcome:** Holdings grouped with per-group subtotals; `Flat` returns one ungrouped list.
- **Variations & edge states:** Group order is deterministic (sorted by key — R4 entity/account stamping makes the buckets stable). Grouping is a local view transform — no re-query, totals unchanged.
- **Backed by spec:** R4 (index anchor for UC-POS-3 — `entityId`/`accountId` are stamped on every fact, so per-account/per-entity grouping is well-defined).
- **Source:** PositionsView.swift:287 + PositionsViewModel.swift:389 + GroupHeader.swift:7.

## UC-POS-4
**Sort holdings (Weight / P/L % / A→Z)**

- **Goal:** Reorder rows by portfolio weight, P/L percent, or alphabetically by ticker.
- **Actor / preconditions:** A user on the POS tab.
- **Entry point:** POS screen → the `SORT ▾` drop-menu pill.
- **Steps (action → see-that):**
  1. Tap `SORT ▾`. → a menu with `Weight`, `P/L %`, `A → Z` (active checkmarked).
  2. Pick one. → rows reorder: `Weight` = market value desc; `P/L %` = unrealised-percent desc; `A → Z` = ticker asc.
- **Outcome:** Rows (within each group) reorder by the chosen key.
- **Variations & edge states:** A local transform inside every group; no re-query. The old "Day %" sort is intentionally absent. The sort is deterministic for fixed facts + config (R8).
- **Backed by spec:** R8 (index anchor for UC-POS-4 — the ordered, deterministic fold that the sort reorders is the R8 projection property).
- **Source:** PositionsView.swift:291 + PositionsViewModel.swift:427.

## UC-POS-5
**Read the portfolio summary (MARKET VAL / COST BASIS / UNREALISED / FY REALISED)**

- **Goal:** Read the scope's aggregate valuation: market value, cost basis, unrealised gain, and active-FY realised.
- **Actor / preconditions:** A user on the POS tab.
- **Entry point:** POS screen → the summary block at the top.
- **Steps (action → see-that):**
  1. Read the top row. → `MARKET VAL` (+ "N positions"), the `CASH` hero, the `TODAY` hero.
  2. Read the breakdown grid below a hairline. → `COST BASIS`, `UNREALISED` (sign-coloured, with %), `<FY> REALISED` (e.g. "FY26 REALISED", sub "realised + income").
- **Outcome:** The user reads the portfolio's headline valuation for the active scope/filter; every cell is exact `Decimal` money (A6), grouped per the entity/account stamped on each fact (A8).
- **Variations & edge states:** MARKET VAL / COST BASIS / UNREALISED are over the class-filtered set. FY REALISED is scope-aware (one entity for entity/account scope, summed for Consolidated), "—" when no FY tax cell. UNREALISED % is null when cost basis is zero (no fabricated divide).
- **Backed by spec:** R9, A6, A8 (index anchors for UC-POS-5 — exact-Decimal money R9/A6; entity+account on every fact A8).
- **Source:** PositionsView.swift:61 + PositionsViewModel.swift:281.

## UC-POS-6
**Read incomplete-cost-basis (GAP / WRITTEN) flags + jump to Sync**

- **Goal:** Identify holdings with an incomplete cost basis (GAP) or a written/short option (WRITTEN), and jump to the Sync inbox to resolve a gap.
- **Actor / preconditions:** A user viewing the POS list (or a parcel inside a holding detail).
- **Entry point:** POS screen → badges on a row's first line + the aggregate "N COST-BASIS GAP(S) →" link in the price-source row.
- **Steps (action → see-that):**
  1. Scan rows for a red `GAP` badge or an accent `WRITTEN` badge. → flagged holdings stand out; per-parcel `GAP`/`WRITE` badges also show inside a holding's PARCELS tab.
  2. Read the aggregate warning. → a red warning triangle + "N COST-BASIS GAP(S) →" on the right of the price-source row (shown only when `gappedCount > 0`; the label singularises).
  3. Tap the "N COST-BASIS GAP(S) →" link. → the app switches to the SYNC tab to resolve (branch to UC-SYNC-10).
- **Outcome:** The user spots which holdings/parcels carry a gap or are written options, and lands on SYNC to address them; any CGT touching a gapped parcel is provisional until filled (A26 safe-fallback).
- **Variations & edge states:** In the CASH section a negative balance is marked `OWED` (a quiet text marker, NOT a gap badge) — money owed, not a flagged position. The gap count is over the scoped set **pre**-class-filter, so the chip (UC-POS-2) does not change it.
- **Backed by spec:** R28, A26 (index anchors for UC-POS-6 — an unassigned/under-basised parcel fails safe to a flagged gap / conservative 0% discount, never a silent understatement).
- **Source:** PositionRow.swift:126 + HoldingDetailView.swift:295 + PositionsGlyphs.swift:9 + RootView.swift:104.

## UC-POS-7
**Switch scope (Consolidated / entity / account)**

- **Goal:** Re-scope every data screen to the whole portfolio, one entity, or one account.
- **Actor / preconditions:** Any user; the switcher lives in the top bar (POS / TAX; hidden inside a pushed holding detail).
- **Entry point:** Top bar → the `SCOPE ▾` switcher.
- **Steps (action → see-that):**
  1. Tap `SCOPE ▾`. → a popover with `CONSOLIDATED` ("N entities · M accounts"), each entity (label + "K accounts · taxKind"), each account (id + label). The active row is accent-highlighted.
  2. Tap a row. → the popover dismisses and the scope changes.
  3. Watch the screens reload. → POS, TAX, cash, and daily-P/L reload for the new scope; the switcher label updates.
- **Outcome:** Every scoped screen reflects the chosen scope; an unassigned/unmatched account fails safe to a flagged gap / 0% discount (A26), never a silent 50%.
- **Variations & edge states:** An outside-tap dismisses without changing scope. Changing scope reloads positions + tax and recomputes daily P/L (the midnight snapshot itself is portfolio-wide). `.all` on a single Company account resolves to that Company, not a hardcoded "IND".
- **Backed by spec:** R4, R28 (index anchors for UC-POS-7 — entity/account stamping R4; explicit per-entity config + safe unassigned-account fallback R28).
- **Source:** ScopeSwitcher.swift:46 + AppContainer.swift:1604.

## UC-POS-8
**View per-position daily P/L vs the midnight snapshot (frozen-FX)**

- **Goal:** See the since-last-midnight (Sydney) change for the whole scope and each holding, with the breakdown that reconciles to the total.
- **Actor / preconditions:** A user on the POS tab; a midnight snapshot is resolved for the current Sydney day (`hasSnapshot`).
- **Entry point:** POS screen → the `TODAY` hero + the `TODAY` column on each row + the "▸ HOW TODAY ADDS UP" disclosure.
- **Steps (action → see-that):**
  1. Read the `TODAY` hero. → a sign-coloured delta dollar figure with a % sub-line.
  2. Read each row's `TODAY` cell. → the per-symbol since-midnight delta with a % sub-line — the **price move only**, insensitive to a change in the current FX rate (FX frozen at the snapshot, A17).
  3. Read the baseline line under the heroes. → "TODAY = SINCE LAST MIDNIGHT, SYDNEY TIME · BASELINE MIDNIGHT <dd MMM> (AEST/AEDT)".
  4. Tap "▸ HOW TODAY ADDS UP". → expands to "Your share prices moved", "Your foreign cash (currency)", "Currency effect on share value", then a "TODAY TOTAL" that reconciles to the hero.
- **Outcome:** The user reads today's portfolio and per-position movement, and sees the terms visibly sum to the total — all display-only, never a tax input (R23).
- **Variations & edge states:** The hero and column ALWAYS render; with no snapshot both show a neutral "—" (never a fabricated $0, never red) under a neutral "TODAY'S BASELINE ISN'T READY YET" banner. A symbol with no mark → "—". The "HOW TODAY ADDS UP" disclosure is hidden entirely for AUD-only scopes (no foreign terms). At AX1 the figure compacts and % drops.
- **Backed by spec:** R25, R31, A17 (index anchors for UC-POS-8 — frozen-FX per-position daily P/L, the price move only, insensitive to current FX; mutation-proven by A17).
- **Source:** PositionsView.swift:135 + PositionRow.swift:200 + HowTodayAddsUp.swift:16 + DailyPLPresentation.swift:200.

## UC-POS-9
**Manually refresh delayed prices**

- **Goal:** Re-fetch the latest delayed quotes and recompute today's change, without re-pulling the statement.
- **Actor / preconditions:** A user on the POS tab.
- **Entry point:** POS screen → the `↻ REFRESH PRICES` capsule in the price-source row.
- **Steps (action → see-that):**
  1. Read the price-source row. → the `↻ REFRESH PRICES` button next to the "delayed/close · as of …" provenance tag.
  2. Tap it. → a pulsing dot + "REFRESHING…" (disabled; a second tap cannot double-fire).
  3. Wait for completion. → prices, per-row DELAYED day-deltas, the TODAY hero/column, and the price-source tag update; the button returns to "REFRESH PRICES".
- **Outcome:** The displayed quotes and daily P/L are refreshed from the latest delayed marks — a **display/overlay** refresh that moves market value and daily P/L but **zero tax cell** (the R8 re-lock, A18).
- **Variations & edge states:** QUOTES-only — distinct from the SYNC tab's statement re-pull (UC-SYNC-5/6). A concurrent re-tap is a no-op. A symbol the provider can't price degrades to close-mark ("quote pending"), never a crash or fabricated price.
- **Backed by spec:** R23, A18 (index anchors for UC-POS-9 — swapping the price feed moves market value / daily P/L, never any tax cell; the byte-identical-tax re-lock A18).
- **Source:** RefreshPricesButton.swift:14 + AppContainer.swift:1477.

## UC-CASH-1
**View per-(entity, account, currency) cash balances, signed**

- **Goal:** Read signed cash balances per currency — native and converted to AUD at today's rate — including amounts owed.
- **Actor / preconditions:** A user on POS whose scope has ≥ 1 cash cell.
- **Entry point:** POS screen → the `CASH` section under the holdings list.
- **Steps (action → see-that):**
  1. Read the section header. → "CASH" with a trailing net-AUD total, then a 3-column table `CURRENCY | BALANCE | IN AUD`.
  2. Read each row. → the home AUD row first ("home currency"), then each foreign currency: a native **signed** BALANCE and the same balance IN AUD with an "@ <rate> today" sub-line.
  3. Inspect a negative (margin) balance. → an `OWED` marker + "you owe IBKR · margin"; the figure keeps its Unicode minus and stays neutral (never red, never clamped to 0).
- **Outcome:** The user reads cash per currency, including amounts owed; the balance is a pure signed fold over `forexConversion` legs + commission + native cash flows (R24/R35), checked against an independent dropped-legs oracle (A16: real bytes == AUD +25,616.73 / USD +1,269.53).
- **Variations & edge states:** Balances are never sign-coloured and never clamped. A foreign row with no resolvable FX → "—" for IN AUD, excluded from the AUD total (never a fabricated rate, R3/A7). The section is hidden only when the scope has no cash cells. Multiple (entity, account) cells in one currency stay separate rows. Dropping the `.forexConversion` fold reproduces the pre-fix AUD +93,627.88 / USD −47,376.22 → a RED mutation.
- **Backed by spec:** R24, R35, A16 (index anchors for UC-CASH-1 — cash is a derived signed fold, corrected by the forex-leg fold, proven against an independent oracle A16).
- **Source:** CashSection.swift:21 + PositionsView.swift:324.

## UC-CASH-2
**Read the CASH hero (net AUD, "AUD only" / "AUD + N FX")**

- **Goal:** See net AUD cash and how many foreign currencies are held, at a glance, without scrolling.
- **Actor / preconditions:** A user on the POS tab.
- **Entry point:** POS screen → the `CASH` hero (between MARKET VAL and TODAY in the summary block).
- **Steps (action → see-that):**
  1. Read the hero. → the `CASH` label, the net AUD cash figure (neutral), and a sub "AUD only" or "AUD + N FX".
- **Outcome:** The user reads net cash and the foreign-currency count at a glance; the net AUD figure equals the CASH section header total (same R24 fold, same A16 oracle).
- **Variations & edge states:** Never sign-coloured. The same net-AUD total as the CASH section header (UC-CASH-1). With no cash cells the hero shows a neutral "—", never a fabricated $0.
- **Backed by spec:** R24, R29, A16 (index anchors for UC-CASH-2 — the DPC cash hero is the display surface of the R24 signed fold; same cash anchor A16).
- **Source:** PositionsView.swift:119.

## UC-HOLD-1
**Open a holding's detail (parcels / disposals / income / events / option terms)**

- **Goal:** Drill into one holding's header, summary stats, and parcels/disposals/income/events.
- **Actor / preconditions:** A user on the POS list.
- **Entry point:** POS screen → tap any position row → the pushed Holding detail (tab + top bar hidden).
- **Steps (action → see-that):**
  1. Tap a row. → a full-screen detail pushes in with a "‹ POSITIONS" back button.
  2. Read the header. → ticker/name/exchange/currency (or option terms), the headline native price + its DELAYED/CLOSE-MARK label, a "NO CHART · CLOSE-MARK ONLY" marker.
  3. Read the summary stats row. → `QTY` (+ avg cost), `MKT VAL` (+ cost), `TODAY` (since-midnight delta + %), `TOTAL P/L` (sign-coloured + %).
  4. Read the sub-tabs. → `PARCELS · N` (default) / `DISPOSAL · N` / `INCOME · N` (shares/ETFs only) / `EVENTS · N`.
  5. Tap "‹ POSITIONS" to return.
- **Outcome:** The user inspects one holding in full; the per-parcel state is the per-`(entity, account, symbol)` lot ledger (R13), and the figures are exact `Decimal` (R9).
- **Variations & edge states:** The price-source label and headline price agree with the list row. TODAY always renders; "—" when no snapshot/mark. A "SINCE MIDNIGHT, SYDNEY · BASELINE MIDNIGHT …" tag shows once a baseline is resolved. The INCOME sub-tab is hidden for options.
- **Backed by spec:** R9, R13, A13 (index anchors for UC-HOLD-1 — exact-Decimal money R9; the per-ticker append-only lot ledger R13 the detail reads; preserved parcel originals A13).
- **Source:** HoldingDetailView.swift:8 + AppContainer.swift:689.

## UC-HOLD-2
**Inspect parcels through a split / corporate action (preserved originals)**

- **Goal:** Inspect each acquisition parcel — date/id, quantity, per-unit price, cost (AUD), unrealised — including parcels rebased by a split or DRP-restate.
- **Actor / preconditions:** A user inside a holding detail.
- **Entry point:** Holding detail → the `PARCELS · N` sub-tab (default).
- **Steps (action → see-that):**
  1. Read the `PARCELS · N` tab (selected by default). → columns `ACQ DATE · ID | QTY | PX · COST | UNREAL`.
  2. Read each parcel. → its acquisition date (+ GAP/WRITE badge), id, qty, native price + AUD cost, a sign-coloured unrealised delta (+ %).
  3. Inspect a split-affected holding. → a **new** rebased parcel (flagged `splitAdjusted`) supersedes the original; the original's `quantity`/`costBaseNative` are unchanged — only `remaining` ever moved (A13).
- **Outcome:** The user sees the lot-by-lot composition, with corporate-action rebases preserved as new parcels rather than edits — so re-matching disposals under a different parcel policy reproduces consistent gains (A13).
- **Variations & edge states:** No parcels → "No parcels". A partial-gap / missing-basis parcel → a `GAP` badge; a written-option parcel → a `WRITE` badge. A split conserves total cost base across the rebase; a DRP-restate adds a cost-base delta to the latest open parcel (R15).
- **Backed by spec:** R14, R15, A13 (index anchors for UC-HOLD-2 — preserved acquisition originals R14, event→transition rules R15, proven by A13 through a split).
- **Source:** HoldingDetailView.swift:255 + HoldingDetailViewModel.swift:217.

## UC-HOLD-3
**Inspect an option's events + premium linkage (exercise / assignment / expiry)**

- **Goal:** Inspect the option capital events on a holding — exercise / assignment / expiry — and how the premium links into the underlying.
- **Actor / preconditions:** A user inside a holding detail (the EVENTS tab; option contract terms in the header for an OPTION row).
- **Entry point:** Holding detail → the `EVENTS · N` sub-tab (+ the option-contract header for an option holding).
- **Steps (action → see-that):**
  1. Tap `EVENTS · N`. → left-ruled cards "OPTION · LONG-TERM" / "OPTION · SHORT-TERM" with "Realised option capital event · gross <amount>".
  2. For an OPTION holding, read the header. → the underlying + "$<strike><C|P>" (call in accent), "EXPIRY <dd MMM yy> · ×<multiplier>", a `WRITTEN` badge if applicable; QTY = contract count.
  3. Trace the premium linkage. → on exercise/assignment the option parcel closes **and** the underlying parcel opens at strike, the premium folded into the underlying's cost base/proceeds (a cross-key mutation, R15/A14).
- **Outcome:** The user reviews realised option capital events and sees the option↔underlying premium link hold; long vs short is the sign on `quantity`, and the OCC symbol round-trips exactly (R16/A15).
- **Variations & edge states:** No events → "No corporate or option events". Corporate-action cards are not yet surfaced (open item) — only option capital events. The INCOME sub-tab is hidden for options (only PARCELS / DISPOSAL / EVENTS).
- **Backed by spec:** R15, R16, A14 (index anchors for UC-HOLD-3 — event→transition rules R15, the instrument/OCC model R16, the cross-key option-assignment proof A14).
- **Source:** HoldingDetailView.swift:460 + HoldingDetailViewModel.swift:268 + HoldingDetailView.swift:112 + PositionRow.swift:99.

## UC-EXPORT-1
**Export the portfolio (valuation / allocation, PDF + CSV) — planned (W4)**

> **PLANNED — W4 (not yet a linked, shipped flow).** The portfolio export menu is the W4 reports
> surface (RPT-1 Portfolio Valuation, RPT-2 Asset Allocation). Steps below describe the intended
> flow; the index carries this row as `(planned — W4)` with no manual deep-link until it ships.

- **Goal:** Generate and share a PDF + CSV of the scoped portfolio (valuation or allocation).
- **Actor / preconditions:** A user on the POS tab; EXPORT is in the top bar.
- **Entry point (intended):** Top bar → the amber `⤴ EXPORT` glyph (POS tab).
- **Steps (action → see-that):**
  1. Tap the `EXPORT` glyph. → a menu: "RPT-1 · Portfolio Valuation" and "RPT-2 · Asset Allocation".
  2. Pick a report. → it builds; if rendering exceeds ~150ms a quiet "EXPORTING <title>…" label shows.
  3. Wait for success. → the system share sheet appears carrying **both** the PDF and the CSV.
- **Outcome (intended):** The user shares/saves a portfolio report for the active scope, built from the materialised `PositionsProjection` (Foundation-only report content, A25), stamped with the last-sync as-of.
- **Variations & edge states:** Built for the active scope (matches what's on screen). A render failure → "EXPORT FAILED · render · RETRY". The share sheet is presented once at the app root.
- **Backed by spec:** A25 (index anchor for UC-EXPORT-1 — report content is Foundation-only/portable, so it lifts to the W6 server unchanged).
- **Source:** (planned — W4) ExportButton.swift:19 + ReportProvider.swift:26 + RootView.swift:56.

## UC-EXPORT-2
**Export a single holding (PDF + CSV) — planned (W4)**

> **PLANNED — W4 (not yet a linked, shipped flow).** Carried in the index as `(planned — W4)` with
> no manual deep-link until it ships. Steps below are the intended flow.

- **Goal:** Generate and share a PDF + CSV Holding-Detail report for one instrument.
- **Actor / preconditions:** A user inside a holding detail.
- **Entry point (intended):** Holding detail header → the `⤴ EXPORT` glyph (top-right).
- **Steps (action → see-that):**
  1. Tap the `EXPORT` glyph. → a single report builds directly (no menu) — "RPT-3 · Holding Detail — <symbol>".
  2. Wait for success. → the share sheet appears with the PDF + CSV.
- **Outcome (intended):** The user shares/saves the holding's detail report, built for the holding's entity scope from the materialised projection (A25).
- **Variations & edge states:** The same ~150ms building affordance and "EXPORT FAILED · render · RETRY" path; URLs route to the single app-level share presenter.
- **Backed by spec:** A25 (index anchor for UC-EXPORT-2 — Foundation-only/portable report content).
- **Source:** (planned — W4) HoldingDetailView.swift:57 + HoldingDetailViewModel.swift:189 + ReportProvider.swift:40.
