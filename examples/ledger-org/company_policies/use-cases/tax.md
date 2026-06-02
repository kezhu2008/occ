# Instruction Manual — Tax

Tap-by-tap user instructions for every Tax use case the **current** app supports, named under the
canonical `UC-<AREA>-<n>` scheme. Grounded in the live code — not the stale `ui-review/UI-MANUAL.md`.
Index: [`use-cases/index.md`](index.md). Backing contracts (R#/A#):
[`company_policies/specs/domain-core.md`](../specs/domain-core.md) (these are on-device rows, so
every `R#/A#` below resolves in `domain-core.md`'s flat namespace, never `backend-lift.md`).

> The five TAX sub-tabs are verbatim: **SUMMARY · CGT · <count> · OPTIMIZE · INCOME · FOREX**. Entity
> scope is NOT a control inside the Tax screen — it's the shared TopBar **SCOPE** switcher (UC-POS-7);
> the Tax VM resolves the taxpayer from scope. FY and parcel-policy are the in-screen controls.

> **Tax never moves with display (read before doubting a number).** Price marks, daily P/L, the
> midnight snapshot, cash, and the `netCash` attribute feed positions/cash/daily-P/L **only** — they
> never enter any tax cell (the R8 re-lock, `domain-core.md` R23 / A18). The **only** engine that
> intentionally moves a tax cell is the Division 775 forex engine (R19/R36). Every figure here is
> exact `Decimal` to the cent, byte-identical-replayable (R9, A1, A3).

---

## UC-TAX-1
**View the per-entity EOY tax summary (net taxable + breakdown)**

- **Goal:** See the headline net taxable figure and its breakdown (LT discounted CGT, short-term, options net, forex / Div 775) for the active taxpayer + FY.
- **Actor / preconditions:** Any user on the TAX tab. A taxpayer is resolved from scope ("TAXPAYER <label>") and an FY is selected. Data may be $0 — a valid state, not an error.
- **Entry point:** Bottom tab bar → **TAX** → SUMMARY sub-tab (default).
- **Steps (action → see-that):**
  1. Tap the **TAX** tab. → the entity callout "TAXPAYER <label>" left, "DISCOUNT 50% · LT" (or 33% SMSF, 0% Company) right.
  2. Read the FY strip + sub-tab row. → e.g. [FY26 ● LIVE] and "SUMMARY · CGT · n · OPTIMIZE · INCOME · FOREX"; SUMMARY active.
  3. Read the hero. → a large value under "NET TAXABLE CAPITAL GAIN" (relabels "NET TAXABLE INCOME" when Div 775 folds in), suffixed "AUD · <entity short>", "<FY label> · <live/lodged>" below.
  4. Read the 2×2 breakdown grid. → "LONG-TERM (DISC)" (gross → "AFTER 50% DISC"), "SHORT-TERM", "OPTIONS NET · n events", "FOREX (TR 96/8)", plus a full-width "GAIN ON YOUR US DOLLARS" Div 775 cell.
- **Outcome:** The user understands their assessable position for the cell; the EOY figure = `netCapitalGainAUD + grossedUpDividendsAUD + (div775ForexRealisedAUD when present)` (R22) and matches the hand-derived oracle to the cent (A1).
- **Variations & edge states:** A $0 hero with prior gains + losses → "$0 — gains offset by losses". A present Div 775 figure adds a hero line "includes $X forex gain/loss on USD cash" and flips precision to 2 dp; the Div 775 cell is **absent** (nil key) when zero, so the EOY bytes are byte-identical to a no-Div-775 world (A21). A live FY shows a green "LIVE · recomputes on every fact ingestion" callout.
- **Backed by spec:** R22, A1 (index anchors for UC-TAX-1 — the per-entity EOY summary R22, proven to the cent against the oracle A1).
- **Source:** TaxSummaryView.swift:18 (hero logic: TaxViewModel.swift:102; breakdown: TaxAtoms.swift:62).

## UC-TAX-2
**Switch financial year (AU FY = 1 Jul–30 Jun)**

- **Goal:** Change which financial year the entire Tax screen computes and displays.
- **Actor / preconditions:** A user on the TAX tab. The FY strip is data-driven (current FY always + lodged FYs + every FY the active entity has activity for).
- **Entry point:** TAX tab → the FY tab strip under the entity callout.
- **Steps (action → see-that):**
  1. Read the FY strip. → one two-line tab per FY — the FY id ("FY26") over "● LIVE" (green) / "● LODGED" (dim). Selected = an amber 1.5px underline.
  2. Tap a different FY chip. → the screen re-queries the cell and the income row; hero, breakdown, CGT count, income, and forex all update.
  3. Read the hero subtitle. → updates to "<FY label> · <status>".
- **Outcome:** Every sub-tab reflects the chosen FY's cell (AU FY = 1 Jul–30 Jun); a lodged FY reads as a pinned snapshot (UC-TAX-11), an open FY recomputes live.
- **Variations & edge states:** Switching FY resets any open disposal/forex card. A fresh real account with only FY26 facts renders only [FY26 ● LIVE]. Selecting the same FY is a no-op.
- **Backed by spec:** R22 (index anchor for UC-TAX-2 — the per-entity-per-FY EOY summary the FY strip selects).
- **Source:** TaxAtoms.swift:10 (selectFY: TaxViewModel.swift:490; data-driven strip: TaxViewModel.swift:544).

## UC-TAX-3
**Switch tax entity (Individual / SMSF / Company), per-entity discount**

- **Goal:** Change the taxpayer entity the whole Tax screen is computed for, applying that entity's CGT discount.
- **Actor / preconditions:** A user on the TAX tab (the SCOPE switcher shows on POS + TAX). Multiple entities/accounts must be configured to be meaningful.
- **Entry point:** TopBar → the "SCOPE ▾" switcher (the shared scope control, UC-POS-7).
- **Steps (action → see-that):**
  1. Tap "SCOPE". → a popover with "CONSOLIDATED" (n entities · n accounts), one row per entity (label · tax kind), one per account (id · label). The active row is amber-highlighted.
  2. Tap an entity row (or CONSOLIDATED / an account). → the menu closes; the taxpayer re-resolves; the Tax cell reloads.
  3. Read the entity callout + discount. → "TAXPAYER <label>" and "DISCOUNT n%" update (Individual 50% / SMSF 33⅓% / **Company 0%**); the FY strip rebuilds for the new entity.
- **Outcome:** Hero, CGT, income, forex, and optimise reflect the newly-scoped taxpayer; the per-entity discount is correct by exact equality against the oracle — a Company disposal taking any discount FAILs (A2).
- **Variations & edge states:** `.entity(id)` → that entity; `.account(id)` → that account's owning entity; `.all` → IND if configured, else the first configured entity (a single-Company real account resolves to that Company, never a hardcoded "IND"). An unassigned account fails safe to 0% (A26). Scope change also re-scopes Positions + daily P/L.
- **Backed by spec:** R17, A2 (index anchors for UC-TAX-3 — per-entity CGT at the per-entity rate R17, proven 50%/33⅓%/0% by A2).
- **Source:** ScopeSwitcher.swift:8 (resolveEntity: TaxViewModel.swift:424; changeScope: AppContainer.swift:1604).

## UC-TAX-4
**View realised CGT disposals per parcel (12-month discount, carry-forward)**

- **Goal:** Inspect each share/ETF disposal's proceeds/cost/gross, the LT/ST split, and the exact parcels consumed.
- **Actor / preconditions:** A user on the TAX tab with ≥ 1 share/ETF disposal in the active cell. The CGT sub-tab label carries the count "CGT · n".
- **Entry point:** TAX tab → CGT sub-tab → "SHARE / ETF DISPOSALS · n" section.
- **Steps (action → see-that):**
  1. Tap "CGT · n". → "SHARE / ETF DISPOSALS · n": one card per disposal — ticker · "SELL · <qty> @ <unit>", date · "n parcels", a right-aligned taxable delta over "TAXABLE", a chevron.
  2. Tap a disposal card. → expands (single-open): a "PROCEEDS / COST BASE / GROSS" row, then "LT GAIN · disc applied" ("<gross> → <discounted>") and "ST GAIN", then a "PARCELS CONSUMED" grid (ID · acq date · held days · LT/ST · qty · gain).
  3. Tap again to collapse.
- **Outcome:** The user can audit each disposal's taxable figure to the parcel level; the engine applies per-event FX translation, the 12-month discount at the per-entity rate, and capital loss carry-forward (R17), matching the oracle to the cent (A1). A wash-sale flag, if present, moves no number (A22).
- **Variations & edge states:** One card open at a time. Switching FY/policy closes any open card. The LT/ST block is hidden when both are zero. No share AND no option events → "No CGT events realised in this FY".
- **Backed by spec:** R17, A1, A22 (index anchors for UC-TAX-4 — per-parcel per-entity CGT R17 to the cent A1; wash-sale flags adjust nothing A22).
- **Source:** TaxDisposalsView.swift:45 (mapping: TaxViewModel.swift:719).

## UC-TAX-5
**View option CGT events (capital default, premium linkage)**

- **Goal:** See each option disposal's taxable outcome — gain/loss, or a cost-base fold via premium linkage.
- **Actor / preconditions:** A user on the CGT sub-tab with ≥ 1 option event.
- **Entry point:** TAX tab → CGT sub-tab → "OPTION CGT · n" section (below share disposals).
- **Steps (action → see-that):**
  1. Scroll to "OPTION CGT · n". → one non-expandable card per event: the underlying · a derived detail ("CLOSE · LONG · nct" / "EXPIRY · LONG · nct"), date, a derived label ("BUY-TO-CLOSE · gain/loss" / "EXPIRY · OTM · premium = loss").
  2. Read the right side. → a taxable delta over "TAXABLE", OR "cost-base ↗" when the event folds into the underlying's cost base (gain == 0).
- **Outcome:** The user sees option CGT outcomes; option treatment is capital by default (revenue overridable per entity), and exercise/assignment fold the premium into the underlying cost base/proceeds (R20), reconciled by the cross-key assignment proof (A14) and matching the oracle (A1).
- **Variations & edge states:** A zero-proceeds negative-gain event renders as an expiry; a positive/negative event with proceeds is a buy-to-close. The section is absent when there are no option events. These cards are not expandable.
- **Backed by spec:** R20, A1, A14 (index anchors for UC-TAX-5 — options tax R20 to the cent A1; the cross-key option-assignment / premium-link proof A14).
- **Source:** TaxDisposalsView.swift:199 (mapping: TaxViewModel.swift:748).

## UC-TAX-6
**View income detail (dividends / franking / withholding / FITO)**

- **Goal:** See dividend/distribution income with franking and withholding, plus itemised rows and the FITO inputs.
- **Actor / preconditions:** A user on the TAX tab; an income row exists for the active entity/FY.
- **Entry point:** TAX tab → INCOME sub-tab.
- **Steps (action → see-that):**
  1. Tap "INCOME". → a three-stat grid: "GROSS" (grossed-up dividends), "FRANKING" (with a "manual *" sub-note when the FY is not lodged), "WITHHELD".
  2. If the FY is not lodged, read the gap card. → "* IBKR DATA GAP" — "IBKR Flex does not expose franking attributes. Franking credits above are user-entered. Withholding tax comes from the US 1042-S equivalent in Flex."
  3. Read the itemised rows. → ticker · type ("DIV"/"DIST"), "n% franked" when present, a neutral grossed-up amount.
- **Outcome:** The user understands dividend income, what is franked, and which figures are user-entered vs from Flex; a foreign dividend is two facts sharing `stableSourceID` (a `dividend`/`distribution` + a `withholding`), paired for the FITO (R18/A10), and `depositWithdrawal` is excluded as capital.
- **Variations & edge states:** Empty → "No income in this FY". On a lodged FY the "manual *" note and the IBKR-gap card are suppressed (franking pinned). Income amounts are neutral (not gain-green).
- **Backed by spec:** R18, A1, A10 (index anchors for UC-TAX-6 — franking + FITO R18 to the cent A1; the two-fact foreign-dividend / FITO pairing A10).
- **Source:** TaxIncomeView.swift:8 (mapping: TaxViewModel.swift:849).

## UC-TAX-7
**View the FX-gain disclosure + Division 775 forex-realisation line**

- **Goal:** See per-event Division 775 forex realisations on USD (the assessable engine) and the captured-FX (TR 96/8) disclosure.
- **Actor / preconditions:** A user on the TAX tab. The Div 775 section appears only when there are realisations.
- **Entry point:** TAX tab → FOREX sub-tab.
- **Steps (action → see-that):**
  1. Tap "FOREX". → if there are realisations, "DIVISION 775 FOREX REALISATION EVENTS · n": one expandable card each — a label ("MSFT" / "AUD→USD conversion") · "DISPOSED · US$<x>", date, a signed gain delta over "FOREX".
  2. Tap a forex card. → expands: "US$ DISPOSED / AUD COST" then "AUD PROCEEDS / GAIN / LOSS" (2-dp).
  3. Read the footer. → "TOTAL FOREX REALISED · ASSESSABLE" with the signed Σ, then the Division 775 explanation.
  4. Read the lower TR 96/8 card. → "FOREX GAINS · TR 96/8": "FX FACTS · n captured at ingest" and "USD EXPOSURE · USD/AUD <rate>".
- **Outcome:** The user can audit each forex realisation and see captured-FX provenance; `div775ForexRealisedAUD` matches a hand-computed FRE-1/FRE-2 cost-base walk (FY26 real bytes **−$1.76 AUD**, A19), is assessable revenue (Company → no discount, R19/R36), and does **not** double-count the share's embedded settlement-FX. The disclosed-only TR 96/8 engine stays separate (AUD→0).
- **Variations & edge states:** Single-open expand, shared with the SUMMARY's FOREX CGT EVENTS section (UC-TAX-12). No realisations → the Div 775 section is absent (only the TR 96/8 card). The TR 96/8 fold is a placeholder. Absent FX on a fact (`fxAtEvent == nil`) surfaces as an FX gap, never a fabricated rate (A7).
- **Backed by spec:** R19, R36, A19 (index anchors for UC-TAX-7 — the Division 775 forex-realisation engine R19/R36, proven against an independent FRE-1/FRE-2 oracle A19).
- **Source:** TaxForexView.swift:14 (mapping: TaxViewModel.swift:868; card: TaxForexView.swift:111).

## UC-TAX-8
**Change parcel-selection policy (FIFO / LIFO / min-gain / max-gain)**

- **Goal:** Re-compute the cell under a different parcel-selection policy and see the tax-affecting difference vs FIFO.
- **Actor / preconditions:** A user on the SUMMARY sub-tab.
- **Entry point:** TAX tab → SUMMARY → "PARCEL-SELECTION POLICY" section.
- **Steps (action → see-that):**
  1. Scroll to "PARCEL-SELECTION POLICY". → a 2×2 grid: "FIFO" (AU default), "MIN GAIN" (sell highest cost-base first), "MAX GAIN" (sell lowest cost-base first), "LIFO" (advisory). The active card is amber-filled.
  2. Tap a non-FIFO card. → the cell re-queries under that policy; the active card highlights.
  3. Read the FIFO Δ panel. → a "FIFO" baseline, the "<POLICY LABEL>" taxable figure, a dashed "Δ tax-affecting" delta.
- **Outcome:** The user sees how the policy changes net taxable vs the FIFO baseline; the CGT engine re-matches parcels↔disposals under the chosen `ParcelPolicy` (R17), and a config change invalidates the cached projection (R28).
- **Variations & edge states:** The Δ panel only appears when policy ≠ FIFO. On a lodged FY the cell is pinned to FIFO and the panel shows "⚠ POLICY LOCKED ON LODGED FY — DIFF ADVISORY ONLY" (UC-TAX-11). The same policy is a no-op. Policy is a screen-wide UI state.
- **Backed by spec:** R17, A1 (index anchors for UC-TAX-8 — per-entity CGT under the selected ParcelPolicy R17, to the cent A1).
- **Source:** PolicyToggle.swift:9 (Δ panel: TaxSummaryView.swift:213; selectPolicy: TaxViewModel.swift:500).

## UC-TAX-9
**Read wash-sale flags + reasoning**

- **Goal:** See likely wash sales flagged with reasoning (Part IVA advisory), and confirm the flag adjusts no tax number.
- **Actor / preconditions:** A user on the TAX tab with a constructed/real wash-sale scenario in the active cell.
- **Entry point:** TAX tab → CGT sub-tab → a wash-sale badge on the relevant disposal (and the "ATO ANTI-AVOIDANCE" advisory on OPTIMIZE, UC-TAX-13).
- **Steps (action → see-that):**
  1. Open the CGT sub-tab. → a disposal that matches the wash-sale heuristic carries a flag with a one-line reasoning.
  2. Read the reasoning. → a plain-language note (e.g. a 30-day re-purchase) marking the disposal as a likely wash sale — advisory only.
  3. Compare the taxable cell with vs without detection. → the CGT/EOY cells are byte-identical (flagging moves nothing, A22).
- **Outcome:** The user sees the wash-sale advisory and trusts that no tax cell silently moved — wash sales are flagged, never adjusted (R21/A22).
- **Variations & edge states:** No matches → no flag (and the OPTIMIZE advisory still explains the Part IVA / 30-day rule). The flag is informational, never a computed adjustment.
- **Backed by spec:** R21, A22 (index anchors for UC-TAX-9 — wash-sale flags-not-adjustments R21, byte-identical-with-flag proof A22).
- **Source:** TaxDisposalsView.swift:45 + TaxOptimizeView.swift:8 (advisory).

## UC-TAX-10
**View tax optimisation (harvest / discount-timing / year-end what-if) — planned (W5)**

> **PLANNED — W5 (carried in the index as `(planned — W5)`, no manual deep-link until it ships).**
> The OPTIMIZE suite is scope-driven (not FY/policy) and folds over today's positions. Steps below
> describe the intended flow.

- **Goal:** See planning levers — parcels nearing the LT threshold, loss-harvest candidates, policy comparison, and a harvest what-if — folded over today's positions.
- **Actor / preconditions:** A user on the TAX tab. The optimise suite is scope-driven, so it does not change with the FY chip.
- **Entry point (intended):** TAX tab → OPTIMIZE sub-tab.
- **Steps (action → see-that):**
  1. Tap "OPTIMIZE". → an intro panel "PROJECTIONS · NO DATA DEPENDENCY".
  2. Read "12-MONTH DISCOUNT TIMING · n". → "save ≈ $X" total; each row: a days-to-LT urgency chip, ticker · parcel · kind, "acq <date> · held <n>d", an unreal-gain delta, "≈ $X if held".
  3. Read "LOSS-HARVEST CANDIDATES · n". → "offset ≈ $X" total, a TICKER·TYPE / MKT VAL / UNREAL table, a reconcile note.
  4. Read "PARCEL-POLICY COMPARISON · Δ vs FIFO" + "WHAT-IF · harvest now". → each policy's net taxable + Δ; "<net if held> → <net if harvested>".
  5. Read the "ATO ANTI-AVOIDANCE" advisory. → Part IVA / 30-day re-purchase.
- **Outcome (intended):** Actionable, plain-language planning guidance with no fabricated numbers — a deterministic fold over projection outputs (A3), so the suite is byte-reproducible for fixed inputs.
- **Variations & edge states:** Empty discount-timing → "No parcels approaching long-term threshold". Empty losses → "No positions at a loss — nothing to harvest". The reconcile note adapts when net taxable ≤ 0. No tappable action column on the loss table.
- **Backed by spec:** A3 (index anchor for UC-TAX-10 — the optimise fold is a deterministic, byte-identical-replayable projection over outputs).
- **Source:** (planned — W5) TaxOptimizeView.swift:8 (mapping: TaxViewModel.swift:777; reconcile: TaxViewModel.swift:838).

## UC-TAX-11
**Confirm a lodged FY is pinned to its snapshot policy**

- **Goal:** Read a lodged FY as a pinned FIFO snapshot that is not recomputed, with policy changes shown as advisory-only.
- **Actor / preconditions:** A user-marked lodged FY exists in `config.lodgedFYs` (or the cell status reads lodged).
- **Entry point:** TAX tab → tap an FY chip whose status reads "● LODGED".
- **Steps (action → see-that):**
  1. Tap a [● LODGED] FY chip. → the hero callout is the dim "PINNED SNAPSHOT · policy at lodge was FIFO · taxable was $X" card (not the green LIVE callout).
  2. Open PARCEL-SELECTION POLICY and pick a non-FIFO policy. → the Δ panel shows "⚠ POLICY LOCKED ON LODGED FY — DIFF ADVISORY ONLY"; the lodged figure does not change.
  3. Open INCOME. → the "manual *" franking note and the IBKR-gap card are suppressed (franking is pinned).
- **Outcome:** The user understands a lodged FY is frozen at the FIFO snapshot taken at lodge; any policy comparison is informational — a lodged FY is pinned to its snapshot policy and not recomputed under a later live policy (R22), proven by exact equality on the pinned cells (A23).
- **Variations & edge states:** Lodged status comes from cell status when present, else `config.lodgedFYs`. The FY dot is dim ("● LODGED") vs green ("● LIVE"). On the real Acme account (`lodgedFYs == []`) no FY is falsely lodged.
- **Backed by spec:** R22, A23 (index anchors for UC-TAX-11 — the lodged-FY pin R22, proven by the lodged-FY pin test A23).
- **Source:** TaxSummaryView.swift:75 (isLodged: TaxViewModel.swift:625; lodgedForFY: TaxViewModel.swift:635).

## UC-TAX-12
**View Div 775 forex events on the SUMMARY**

- **Goal:** See per-event Div 775 forex realisations and their full-precision assessable total directly on the main SUMMARY, not buried on the FOREX sub-tab.
- **Actor / preconditions:** A user on the SUMMARY sub-tab with ≥ 1 forex realisation.
- **Entry point:** TAX tab → SUMMARY → "FOREX CGT EVENTS · n" section (below the breakdown grid).
- **Steps (action → see-that):**
  1. Scroll past the breakdown. → "FOREX CGT EVENTS · n" with a "tap for detail ›" link, then a "REALISED SHARE CGT" line (or "$0 — no disposals sold this year").
  2. Read the per-event cards. → the same expandable style as the FOREX sub-tab; tap one to expand workings inline.
  3. Read the total. → "TOTAL FOREX REALISED · ASSESSABLE" at FULL precision (e.g. "−$1.76", never rounded "−$2"), then the Division 775 explanation.
  4. Tap "tap for detail ›". → jumps to the FOREX sub-tab (UC-TAX-7).
- **Outcome:** The forex evidence (event list + precise total) is visible on the main tax view, so a "CGT 0" headline is explained rather than reading as "no data"; the figure is the same Div 775 assessable line proven against the independent oracle (R19/A19) and is absent (nil key) when zero (A21).
- **Variations & edge states:** The section is absent when there are no realisations. Single-open expand, shared with the FOREX sub-tab. The total renders verbatim at 2 dp (guarded against a dec:0 regression).
- **Backed by spec:** R19, A19 (index anchors for UC-TAX-12 — the Division 775 forex-realisation line R19, proven against the independent FRE-1/FRE-2 oracle A19).
- **Source:** TaxSummaryView.swift:296 (ForexEventsSection).

## UC-TAX-13
**Confirm display overlays (cash, daily P/L) move ZERO tax cell (the R8 re-lock)**

- **Goal:** Confirm that toggling the display overlays — daily P/L, cash, the midnight snapshot, price marks — moves market value and daily P/L but **no** tax cell.
- **Actor / preconditions:** A user on the SUMMARY sub-tab who can toggle the overlay on/off (or compare across two midnight snapshots).
- **Entry point:** TAX tab → SUMMARY (with the overlay present vs absent, or two different snapshots).
- **Steps (action → see-that):**
  1. Read the SUMMARY hero + breakdown with no daily-P/L / no cash / no snapshot. → record the EOY + CGT cells.
  2. Turn the overlay on (daily P/L + cash + a snapshot present). → market value and daily P/L move on POS, but the TAX hero, breakdown, and CGT cells are **byte-identical** (A18).
  3. Switch to a different midnight snapshot. → again the tax cells are unchanged; only display figures move.
- **Outcome:** The user confirms the R8 firewall holds — price marks, daily P/L, the snapshot, the cash projection, and `netCash` feed valuation/cash/daily-P/L only, never any lot-ledger/CGT/income/FITO/FX-gain/Div-775/EOY input (R23). The only intentional tax mover is the Div 775 engine.
- **Variations & edge states:** A deliberate leak of `snapshot.mark` / `currentFX` / `netCash` into a tax input turns the byte-identical-tax test RED (the non-vacuous negative control, A18). Refreshing prices (UC-POS-9) is the live equivalent of this toggle and likewise moves no tax cell.
- **Backed by spec:** R23, A18 (index anchors for UC-TAX-13 — display-never-moves-tax R23, the mutation-proven byte-identical-tax re-lock A18).
- **Source:** TaxSummaryView.swift:18 + PositionsView.swift:135 (overlay toggle: AppContainer.swift:1477).

## UC-TAX-14
**Export / share CGT & EOY reports (PDF + CSV) — planned (W4)**

> **PLANNED — W4 (carried in the index as `(planned — W4)`, no manual deep-link until it ships).**
> Steps below describe the intended flow.

- **Goal:** Generate and share one of the Tax reports for the entity/FY currently in view.
- **Actor / preconditions:** A user on the TAX tab; EXPORT is in the TopBar trailing slot.
- **Entry point (intended):** TopBar → the "EXPORT" glyph.
- **Steps (action → see-that):**
  1. Tap "EXPORT". → a menu: "RPT-4 Realised CGT Report", "RPT-5 EOY Tax Summary", "RPT-6 Income & Franking Report", "RPT-7 Foreign Income & FITO Report", "RPT-8 Forex-Gain (captured-FX) Summary", "RPT-9 Tax-Optimisation Report".
  2. Tap a report. → it builds off the active entity/FY (RPT-9 uses scope); if render > ~150ms a quiet "EXPORTING <TITLE>…" shows.
  3. Wait for success. → the system share sheet opens with **both** the PDF and the CSV.
- **Outcome (intended):** A shareable PDF + CSV of the chosen tax report for what the user was viewing, built from materialised projections (no recompute at export) whose figures already match the oracle to the cent (A1).
- **Variations & edge states:** RPT-4..8 are built for the active entity + FY; RPT-9 for the active scope. A single app-level share presenter. A render miss → "EXPORT FAILED · render · RETRY".
- **Backed by spec:** A1 (index anchor for UC-TAX-14 — the exported tax figures are the oracle-exact `TaxProjection`).
- **Source:** (planned — W4) ExportButton.swift:19 (catalog: ReportProvider.swift:53; wiring: RootView.swift:56 / AppContainer.swift:496).
