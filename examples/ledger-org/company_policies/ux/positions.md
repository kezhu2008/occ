# UX Manual — Positions Screen

> Owned by the **UX Manager** (reviewed by `ledger-ux-reviewer`). The authoritative **experience** spec for the Positions tab: what the user comes here to do, what the screen tells them at a glance, the flows they drive, the states they can land in, and the amber-terminal aesthetic it all wears. This is the UX layer — *how it feels and reads*, not the pixel-faithful component spec (that is the designer's `design-spec-screens.md` §M8-T3, which this references and never repeats) nor the data contract (architect's `data-model.md`).
>
> Scope boundary: this doc answers *"is the Positions experience right for the self-directed investor?"* Use-case **steps** (tap-by-tap → see-that) are owned by the Product Manager in `use-cases/positions-and-holdings.md`; this manual serves those use cases but speaks in the user's intent and the screen's reading order. Where the two touch (which figure is headline, what an empty state should say), this manual is the UX authority and cites the UC it realizes.
>
> Wave: Positions is **W1-DONE** at close-mark fidelity. The W3 affordances (delayed-quote sparks, day-deltas lit from live quotes, price alerts) are built as seams that render their W1 state only (Decision D9). This manual describes the **W1 experience** and flags each W3 seam as **[W3 seam]** so the reader never mistakes a parked affordance for a missing one.

---

## 1. Who is here, and why

The Positions tab is the investor's **home base** — the first place they look when they open Ledger and the place they return to between tasks. The user is the founder-as-user-zero (per the charter): a self-directed Australian value investor holding through one or more IBKR accounts and tax entities, in more than one currency, across shares and options.

They come to Positions to answer, in order of urgency:

1. **"What is my portfolio worth right now?"** — the single number they want before anything else.
2. **"How much cash do I have, and in which currencies?"** — separate from holdings, never blended in.
3. **"What moved today?"** — the since-midnight change, honestly anchored or honestly absent.
4. **"What do I own, and how is each holding doing?"** — the holdings list, sliceable by class, account, entity, and sort.
5. **"Where is my data incomplete?"** — cost-basis gaps surfaced, not hidden, with a one-tap route to fix them (the charter's *honest about incomplete data* promise, made visible).

Everything on the screen is in service of those five questions, and the layout puts them in that priority order top-to-bottom.

### Use cases this screen serves

| Use case | What the user is doing here |
|---|---|
| `UC-POS-1` | Reading the holdings list — ticker/qty, price, market value, today's change, lifetime P/L |
| `UC-POS-2` | Filtering by asset class (ALL · EQ · ETF · OPT) |
| `UC-POS-3` | Grouping (Flat / By account / By entity) with subtotals |
| `UC-POS-4` | Sorting (Weight / P/L % / A → Z) |
| `UC-POS-5` | Reading the portfolio summary (MARKET VAL / COST BASIS / UNREALISED / FY REALISED) |
| `UC-POS-6` | Reading GAP / WRITTEN flags, and jumping to Sync to resolve a gap |
| `UC-POS-7` | Switching scope (Consolidated / entity / account) from the top bar |
| `UC-POS-8` | Reading the daily P/L hero + column, the baseline line, and the "HOW TODAY ADDS UP" reconciliation |
| `UC-CASH-1` / `UC-CASH-2` | Reading the CASH hero and the per-currency CASH section |
| `UC-HOLD-1` / `UC-HOLD-2` / `UC-HOLD-3` | Drilling into one holding (parcels / disposals / income / events / option terms) |
| `UC-POS-9` | Manually refreshing delayed prices |
| `UC-EXPORT-1` / `UC-EXPORT-2` | Exporting the portfolio or a single holding (PDF + CSV) |

---

## 2. The headline field — and why it is the headline

The **MARKET VAL** figure sits top-left of the summary block, the largest and topmost number on the screen. This is deliberate: it is the answer to question 1, the number the investor is here for, so it gets the typographic top billing (the 17pt `StatView` headline, tabular digits, no sign colour — a value, not a delta). Under it: `"<n> positions"`, so the headline carries its own count.

The headline is **scope-aware and filter-aware** (`UC-POS-5`): it shows the market value of exactly the set currently on screen. Switch scope from Consolidated to your SMSF and the headline drops to the SMSF's market value; tap the `OPT` class chip and it recomputes over options only. The number the user reads at the top is always the number the list below it sums to — there is never a headline that disagrees with the rows.

Two companion heroes sit to the right of MARKET VAL, in priority order, so the three questions the user asks most are answered without scrolling:

- **CASH hero** (`UC-CASH-2`) — net AUD cash, with a sub-line `"AUD only"` or `"AUD + N FX"`. Never sign-coloured, never clamped: cash is a fact, not a gain. Holdings and cash are kept visibly distinct so the investor never reads a blended number.
- **TODAY hero** (`UC-POS-8`) — the since-last-midnight (Sydney) change for the whole scope: a sign-coloured Δ dollar figure with a % sub-line. This hero **always renders**; with no resolved midnight snapshot it shows a neutral `"—"` (never a fabricated `$0`, never red).

Below a hairline, the breakdown grid answers the slower questions: **COST BASIS**, **UNREALISED** (sign-coloured, with %), and **`<FY>` REALISED** (e.g. `"FY26 REALISED"`, sub `"realised + income"`). The split is intentional — heroes are *now*, the grid is *the story so far*.

---

## 3. Key flows

### 3a. Re-scope the whole picture (`UC-POS-7`)
The `SCOPE ▾` switcher in the top bar is the master control for the screen. The investor holds across several taxpayers (their own name, an SMSF, a company) and the charter forbids ever blending them, so scope is a first-class, always-visible dial — not buried in settings. Picking Consolidated, an entity, or a single account re-queries the projection and reloads positions, cash, tax, and the daily-P/L heroes together. The headline, the rows, the CASH section, and the gap count all move as one.

### 3b. Slice the list — filter, group, sort (`UC-POS-2` / `UC-POS-3` / `UC-POS-4`)
A single filter/group/sort bar sits between the summary and the column header. All three are **local view transforms** — instant, no reload, no re-query:
- **Filter** by class (ALL · EQ · ETF · OPT). The summary totals and per-row weights recompute over the filtered set.
- **Group** Flat / By account / By entity. Grouping inserts inline header strips (an uppercase amber label + `"sub · count"` + a trailing AUD subtotal); totals are unchanged, order is deterministic.
- **Sort** Weight (market-value desc) / P/L % (unrealised-% desc) / A → Z (ticker asc), applied within each group.

The one cross-cut: the **cost-basis gap count** in the price-source row is computed over the scoped set *before* the class filter, so flipping chips never changes it — a gap doesn't disappear because you filtered it out of view.

### 3c. Understand today, or why today isn't ready (`UC-POS-8`)
Directly under the heroes is the **as-of line** — the honest anchor for "TODAY". With a snapshot it reads `"TODAY = SINCE LAST MIDNIGHT, SYDNEY TIME · BASELINE MIDNIGHT <dd MMM> (AEST/AEDT)"`. Without one, a neutral (not red) banner replaces just that line: `"TODAY'S BASELINE ISN'T READY YET"`, explaining the midnight snapshot hasn't been fetched while reassuring the user that holdings and cash below are still correct. The TODAY hero and column stay (`"—"`); only the anchor line changes.

For multi-currency scopes, a collapsed **"▸ HOW TODAY ADDS UP"** disclosure lets the investor expand the today number into the terms that visibly sum to it — *share prices moved*, *foreign cash (currency)*, *currency effect on share value* — then a `TODAY TOTAL` that reconciles to the hero. It is hidden entirely for AUD-only scopes (no foreign terms to explain) and collapsed by default, so the simple case stays simple.

### 3d. See the gap, fix the gap (`UC-POS-6`)
When the scoped set has ≥ 1 holding with an incomplete cost basis, the price-source row shows a warning-triangle + `"<n> COST-BASIS GAP(S) →"` in the loss colour. This is the charter promise made tactile: the tool never emits a silently-wrong gain, it tells you. One tap routes to the Sync tab to resolve it (`UC-SYNC-10`). Per-row `GAP` badges and per-parcel `GAP`/`WRITE` badges carry the same honesty down to the lot level. (In the CASH section a negative balance is marked `OWED` — a quiet text marker, *not* a badge: money owed, not a flagged position.)

### 3e. Drill into one holding (`UC-HOLD-1` / `UC-HOLD-2` / `UC-HOLD-3`)
Tapping any row pushes a full-screen holding detail (tab bar + top bar hidden, a `‹ POSITIONS` back button). The header and headline price agree with the list row the user just left — the price-source label, the native price, the TODAY Δ all match, so drilling in never surprises. Inside, sub-tabs cover `PARCELS · N` (the lot ledger, default), `DISPOSAL · N` (realised gains with the LT/ST split), `INCOME · N` (dividends + franking; absent for options), and `EVENTS · N` (option capital events). Option holdings additionally read their full contract terms in the header (strike, put/call in amber for calls, expiry, multiplier, WRITTEN badge).

### 3f. Refresh prices, export the picture (`UC-POS-9`, `UC-EXPORT-1` / `UC-EXPORT-2`)
- **Refresh** (`↻ REFRESH PRICES` in the price-source row) re-fetches delayed marks and recomputes today's change *without* re-pulling the statement — a quotes-only refresh, distinct from the Sync tab's statement re-pull. While running it shows a pulsing dot + `"REFRESHING…"` and disables itself (a second tap can't double-fire).
- **Export** (the amber `⤴ EXPORT` glyph in the top bar) offers two portfolio reports — `RPT-1 · Portfolio Valuation` and `RPT-2 · Asset Allocation` — each built for the active scope (matching what's on screen) and stamped with the last-sync as-of, then handed to the system share sheet carrying **both** the PDF and the CSV. Inside a holding detail, the export glyph builds `RPT-3 · Holding Detail` for that one instrument directly (no menu).

---

## 4. States

The chrome (top bar, tab bar, scope switcher) **always renders**; the screen owns its own empty / loading / real / error states. Numbers are never bare text — every figure is tabular-digit (`monospacedDigit`). The screen has no demo/sample-data path: rows come from the live `PositionsProjection` at the injected scope, so an empty result is genuinely empty, not a fabricated zero.

### Empty (real, just nothing to show)
- **No holdings match the filter** — centred `"No positions match this filter"` (Plex Mono 11, `dim2`), padding `40×24`. The summary, CASH, and TODAY heroes still render above it.
- **No income / disposals / events** in a holding detail tab — `"No income events"`, `"No disposals — nothing realised on this position"`, `"No corporate or option events"`. Honest absence, phrased as a fact.
- **Baseline not ready** — the TODAY anchor line becomes the neutral `"TODAY'S BASELINE ISN'T READY YET"` banner; heroes and column show `"—"`. This is an *empty today*, not an error.
- **A quote-pending symbol** — `"—"` for PX / MKT VAL on that row; the rest of the screen is unaffected.

### Loading
A centred `MicroLabel("LOADING")` — no bouncy spinner, no skeleton shimmer. The terminal aesthetic prefers a still, quiet word to motion. The chrome is already on screen, so the user keeps their orientation while the projection folds.

### Real (the resting state)
Summary block (MARKET VAL / CASH / TODAY heroes + COST BASIS / UNREALISED / FY REALISED grid + price-source row) → filter/group/sort bar → column header `TICKER · QTY | PX | MKT VAL | TODAY | TOTAL P/L` → grouped holding rows → CASH section (per-currency, AUD first) → bottom spacer clearing the tab bar. Each row reads in two lines: ticker + exchange (or option terms) with any `GAP`/`WRITTEN` badge on line 1; qty + masked account (`U7*…4192`) on line 2.

### Error
A **terminal error panel** — a bordered card with a loss-colour left-rule:
- **Projection build failure** → `"COULD NOT LOAD POSITIONS"` panel (the rest of the screen does not render fabricated rows behind it).
- **Export render failure** → `"EXPORT FAILED · render · RETRY"`.
Errors are stated plainly and never dressed up; the loss colour signals trouble without alarm.

### Dynamic Type / accessibility
At AX1 the TODAY / TOTAL P/L dollar figures compact (`+$1.3k`) and the % sub-lines drop, but **columns are never removed** — the investor never loses a data field to font size. Signal colours are always reinforced by a `+`/`−` sign glyph and position, never hue alone; the `accessible` temperament (blue/orange) exists for colour-vision needs. Primary numbers always carry `text`/`textHi` weight, never `dim`.

---

## 5. Aesthetic — amber terminal, IBM Plex

Positions wears Ledger's house look: a **dark amber-terminal** surface that reads like a precise instrument, not a consumer fintech dashboard. The feel is *quiet, dense, exact* — a tool the investor trusts with a number they will file on.

- **Surface.** Near-black ground (`bg #0C0C0D`), with `bg2 #121215` for the summary panel and column-header strip, `surface #17171B` for chips/badges/cards. A warm-paper light palette (`bg #F7F6F3`) mirrors it for daytime.
- **The single accent is amber** (`accent #D4A056`, darker `#A47238` on paper). Amber is rationed on purpose: the selected tab, the active chip, group-header labels, call-option strikes, and the EXPORT glyph — nothing else. One restrained accent is what makes the screen read as a terminal rather than a toy.
- **Type.** IBM Plex Sans for titles; **IBM Plex Mono** for every label and number. Micro-labels are 9–11px, uppercase, letter-spaced (`.tracking(0.4–0.5)`), in `dim`/`dim2` — they frame the data without competing with it.
- **Tabular digits everywhere.** Numbers are monospaced so columns align to the decimal and figures don't jiggle as they update — the visual correlate of *accurate to the cent*.
- **Signal colours are muted by default** (`restrained`: sage `#9BB89C` up, clay `#C4937E` down), and only ever colour *deltas* — UNREALISED, TODAY, TOTAL P/L. Values (MARKET VAL, CASH, COST BASIS) and cash balances stay neutral. A zero/neutral delta uses `dim`, not a signal colour.
- **Hairlines, not boxes.** Sections are divided by 1px (`1/displayScale`) hairlines and dashed rules — the density comes from restraint, not chrome. Empty/loading is a single centred Plex Mono line; error is a bordered card with a loss-colour left-rule.

### Wireframe (dark, real state, Consolidated scope)

```
┌──────────────────────────────────────────────────────────────┐
│  POSITIONS                              SCOPE ▾   ⤴ EXPORT    │  ← top bar (chrome, amber accents)
│  14 positions · 3 accts · USD/AUD 0.6543                       │
├──────────────────────────────────────────────────────────────┤
│  MARKET VAL            CASH              TODAY                 │  ← summary block (bg2)
│  $1,284,310            $42,118           +$3,204               │
│  14 positions         AUD + 1 FX        +0.25%                 │
│  · · · · · · · · · · · · · · · · · · · · · · · · · · · · · · · │  (hairline)
│  COST BASIS     UNREALISED          FY26 REALISED             │
│  $1,002,540     +$281,770  +28.1%   +$8,247                   │
│  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -│  (dashed)
│  CLOSE · AS OF 30 MAY 16:00      ⚠ 2 COST-BASIS GAPS →        │  ← price-source row (amber refresh, clay gap)
│  ↻ REFRESH PRICES                                             │
├──────────────────────────────────────────────────────────────┤
│  TODAY = SINCE LAST MIDNIGHT, SYDNEY · BASELINE 30 MAY (AEST) │  ← as-of line
│  ▸ HOW TODAY ADDS UP                                          │  ← reconciliation (multi-ccy only)
├──────────────────────────────────────────────────────────────┤
│ [ALL] EQ  ETF  OPT                       GROUP ▾   SORT ▾     │  ← filter / group / sort (active chip amber)
├──────────────────────────────────────────────────────────────┤
│ TICKER · QTY     PX      MKT VAL    TODAY      TOTAL P/L      │  ← column header (bg2)
├──────────────────────────────────────────────────────────────┤
│ CBA  ASX         168.40  $84,200    +$310      +$22,140       │
│ 500 · U7*…4192                      +0.37%     +35.6%         │
│ BHP  ASX  GAP    45.10   $22,550    −$95       +$1,205        │  ← clay GAP badge
│ 500 · U7*…4192                      −0.42%     +5.6%          │
│ AAPL NASDAQ      189.95  US$37,990  +$540      +$9,870        │
│ 200 · U7*…0021                      +1.44%     +35.1%         │
├──────────────────────────────────────────────────────────────┤
│ CASH                                              $42,118 AUD │  ← cash section header (net AUD)
│ CURRENCY   BALANCE              IN AUD                        │
│ AUD        $31,540              $31,540   home currency       │
│ USD        US$16,170           $10,578    @ 0.6543 today      │
└──────────────────────────────────────────────────────────────┘
```

The wireframe shows the resting reading order: the worth-now headline first, cash and today beside it, the honest baseline anchor and gap link, then the sliceable list, then cash broken out per currency — the five questions of §1, answered top to bottom, in amber on near-black.
