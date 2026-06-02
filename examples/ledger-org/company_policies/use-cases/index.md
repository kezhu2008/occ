# Ledger — Use-case catalog (THE canonical index)

The master catalog of **every use case Ledger supports**, owned by the **Product Manager**. This file
is **THE authority for use-case IDs**: nothing anywhere in the example may cite a `UC-…` that is not a
row below. Each row is one user goal: *what the user does*, named under the **one canonical scheme**
`UC-<AREA>-<n>`.

**Canonical scheme.** `AREA ∈ {ONB, SYNC, POS, HOLD, TAX, CASH, CONFIG, EXPORT, CONN, SCHED, HEALTH}`;
`<n>` is a stable monotonic integer **per area** (never renumbered — deprecate instead). All older
schemes found in the codebase (`UC-INGEST-WEB`, `UC-SYNC-5`, `UC-POS-1`, `UC-POSITIONS`, `UC-ONB-FLEX`,
`UC-SYNC-SCHED-1`, the dotted-spec citations, …) translate to a canonical ID via the **[Alias map](#alias-map-old--canonical-uc)** at the bottom.

**Columns.** `ID` (canonical) · `Use case` (the user goal) · `Entry point` (where the user starts) ·
`Manual` (deep-link into the per-area manual; only links files that exist) · `Backed-by spec` (the
`R#`/`A#` contract — **only IDs that `specs/domain-core.md` or `specs/backend-lift.md` actually
define**) · `Source` (`file:line` once built, else `(to be implemented)`).

> **The chain (PM → ARCHITECT → ENGINEER → QA).** A use case is the *demand*; its `Backed-by spec`
> `R#/A#` is the *contract* (offline-testable acceptance in `specs/`); QA asserts the manual's
> *see-that* against that `A#`. A row with no backing spec is a gap reviewers FAIL.

**Spec-namespace note (read before citing a number).** There are **two** flat `R#`/`A#` namespaces:
- **On-device** — `specs/domain-core.md` (defines `R1–R39`, `A1–A29` — all **flat**; the historical
  DPC/FXC sub-acceptance are folded into those flat anchors, no dotted IDs). Used by every area
  **except** the W6 server lift.
- **W6 server lift** — `specs/backend-lift.md` (defines its **own** flat `R1–R24`, `A1–A16`).
The same number means different things across the two files; resolve by **file + concept**, never by
raw number. Rows in areas **CONN / SCHED / HEALTH** cite the **`backend-lift.md`** namespace; all other
rows cite **`domain-core.md`**. The `Backed-by spec` cell tags `[BL]` when it means `backend-lift.md`.

**Manuals.** Only three per-area manuals exist today and are linked:
[`onboarding-and-sync.md`](onboarding-and-sync.md) · [`positions-and-holdings.md`](positions-and-holdings.md) ·
[`tax.md`](tax.md). Genuinely-future areas (watchlist/alerts, the W6 backend backlog) carry
**`(planned — Wn)`** with **no link** until their manual lands.

The 5 on-device bottom tabs: **POS** (Positions) · **WATCH** (Watchlist) · **TAX** · **SYNC** · **CONFIG**.
The W6 surfaces are server-side flows + the thin iOS viewer.

---

## 1. Onboarding & sync — `onboarding-and-sync.md`

Real-data path: first-run onboarding, the Flex Web Service token pull (W2), multi-month history
stitching, reconciliation to the cent, the "Sync now" re-pull, and the cost-basis data-gap inbox.

| ID | Use case | Entry point | Manual | Backed-by spec | Source |
|----|----------|-------------|--------|----------------|--------|
| UC-ONB-1 | First-run onboarding via Flex token (live connect) | Launch → onboarding → CONNECT IBKR | [manual](onboarding-and-sync.md#uc-onb-1) | R24, R39, A1 | (to be implemented) |
| UC-ONB-2 | First-run onboarding via Flex XML file import | Launch → onboarding → Import a file | [manual](onboarding-and-sync.md#uc-onb-2) | R1, R12, A4 | (to be implemented) |
| UC-ONB-3 | In-flow "how to get your Flex token" helper | Onboarding → ▸ HOW DO I GET THESE? | [manual](onboarding-and-sync.md#uc-onb-3) | R39 | (to be implemented) |
| UC-ONB-4 | Retry a failed onboarding pull (token-free error) | Onboarding → FAILED → RETRY | [manual](onboarding-and-sync.md#uc-onb-4) | R39, A1 | (to be implemented) |
| UC-SYNC-1 | Connect IBKR / import a local Flex XML statement | SYNC → SOURCES → IMPORT FILE… | [manual](onboarding-and-sync.md#uc-sync-1) | R1, R2, A4 | (to be implemented) |
| UC-SYNC-2 | Pull the real statement via Flex Web Service (SendRequest → poll → GetStatement) | SYNC → SOURCES → CONNECT IBKR (token + Query ID) | [manual](onboarding-and-sync.md#uc-sync-2) | R1, R12, A24 | (to be implemented) |
| UC-SYNC-3 | Add another Flex query (multi-query / multi-account ingest) | SYNC → SOURCES → ADD QUERY | [manual](onboarding-and-sync.md#uc-sync-3) | R2, R4, A8 | (to be implemented) |
| UC-SYNC-4 | Stitch history across Flex's capped date ranges | SYNC → SOURCES → REFRESH FROM IBKR | [manual](onboarding-and-sync.md#uc-sync-4) | R2, R12, A4 | (to be implemented) |
| UC-SYNC-5 | "Sync now" — re-pull the real statement from the stored token | SYNC → STATE → SYNC NOW | [manual](onboarding-and-sync.md#uc-sync-5) | R39, A29 | (to be implemented) |
| UC-SYNC-6 | Refresh from IBKR / re-import the same statement safely (idempotent) | SYNC → SOURCES → re-run a pull | [manual](onboarding-and-sync.md#uc-sync-6) | R2, R12, A4 | (to be implemented) |
| UC-SYNC-7 | View sync state + reconcile to the cent (FY span, fact digest, cash tie-out) | SYNC → STATE → LEDGER STATE | [manual](onboarding-and-sync.md#uc-sync-7) | R26, A24, A3 | (to be implemented) |
| UC-SYNC-8 | Reconnect after a degraded launch | SYNC → SOURCES → reconnect prompt | [manual](onboarding-and-sync.md#uc-sync-8) | R39, A29 | (to be implemented) |
| UC-SYNC-9 | Disconnect the IBKR connection | SYNC → SOURCES → DISCONNECT FLEX | [manual](onboarding-and-sync.md#uc-sync-9) | R39 | (to be implemented) |
| UC-SYNC-10 | View the cost-basis data-gap inbox | SYNC → GAPS | [manual](onboarding-and-sync.md#uc-sync-10) | R28, A26 | (to be implemented) |
| UC-SYNC-11 | Resolve a cost-basis gap with a manual fill | SYNC → GAPS → RESOLVE | [manual](onboarding-and-sync.md#uc-sync-11) | R15, R28, A12 | (to be implemented) |
| UC-SYNC-12 | Inspect a flagged forex-leg gap (no usable pair) | SYNC → GAPS → forex gap card | [manual](onboarding-and-sync.md#uc-sync-12) | R34, A11 | (to be implemented) |
| UC-SYNC-13 | Snooze / ignore a cost-basis gap | SYNC → GAPS → row menu | [manual](onboarding-and-sync.md#uc-sync-13) | R28, A26 | (to be implemented) |

## 2. Positions, cash & daily P&L — `positions-and-holdings.md`

| ID | Use case | Entry point | Manual | Backed-by spec | Source |
|----|----------|-------------|--------|----------------|--------|
| UC-POS-1 | View holdings list (ticker/qty, price, market value, today's change, lifetime P/L) | POS tab | [manual](positions-and-holdings.md#uc-pos-1) | R8, R25, A25 | (to be implemented) |
| UC-POS-2 | Filter holdings by asset class (ALL · EQ · ETF · OPT) | POS → class chips | [manual](positions-and-holdings.md#uc-pos-2) | R16 | (to be implemented) |
| UC-POS-3 | Group positions (Flat / by account / by entity) with subtotals | POS → GROUP menu | [manual](positions-and-holdings.md#uc-pos-3) | R4 | (to be implemented) |
| UC-POS-4 | Sort holdings (Weight / P/L % / A→Z) | POS → SORT menu | [manual](positions-and-holdings.md#uc-pos-4) | R8 | (to be implemented) |
| UC-POS-5 | Read the portfolio summary (MARKET VAL / COST BASIS / UNREALISED / FY REALISED) | POS → summary block | [manual](positions-and-holdings.md#uc-pos-5) | R9, A6, A8 | (to be implemented) |
| UC-POS-6 | Read incomplete-cost-basis (GAP / WRITTEN) flags + jump to Sync | POS → row GAP badge | [manual](positions-and-holdings.md#uc-pos-6) | R28, A26 | (to be implemented) |
| UC-POS-7 | Switch scope (Consolidated / entity / account) | Top bar → SCOPE switcher | [manual](positions-and-holdings.md#uc-pos-7) | R4, R28 | (to be implemented) |
| UC-POS-8 | View per-position daily P/L vs the midnight snapshot (frozen-FX) | POS → TODAY column / hero | [manual](positions-and-holdings.md#uc-pos-8) | R25, R31, A17 | (to be implemented) |
| UC-POS-9 | Manually refresh delayed prices | POS → REFRESH | [manual](positions-and-holdings.md#uc-pos-9) | R23, A18 | (to be implemented) |
| UC-CASH-1 | View per-(entity, account, currency) cash balances, signed | POS → CASH section | [manual](positions-and-holdings.md#uc-cash-1) | R24, R35, A16 | (to be implemented) |
| UC-CASH-2 | Read the CASH hero (net AUD, "AUD only" / "AUD + N FX") | POS → CASH hero | [manual](positions-and-holdings.md#uc-cash-2) | R24, R29, A16 | (to be implemented) |
| UC-HOLD-1 | Open a holding's detail (parcels / disposals / income / events / option terms) | POS → tap a row | [manual](positions-and-holdings.md#uc-hold-1) | R9, R13, A13 | (to be implemented) |
| UC-HOLD-2 | Inspect parcels through a split / corporate action (preserved originals) | Holding → PARCELS | [manual](positions-and-holdings.md#uc-hold-2) | R14, R15, A13 | (to be implemented) |
| UC-HOLD-3 | Inspect an option's events + premium linkage (exercise / assignment / expiry) | Holding → EVENTS / option terms | [manual](positions-and-holdings.md#uc-hold-3) | R15, R16, A14 | (to be implemented) |
| UC-EXPORT-1 | Export the portfolio (valuation / allocation, PDF + CSV) | POS → EXPORT | (planned — W4) | A25 | (to be implemented) |
| UC-EXPORT-2 | Export a single holding (PDF + CSV) | Holding → EXPORT | (planned — W4) | A25 | (to be implemented) |

## 3. Tax — `tax.md`

CGT (shares + options), income / franking / FITO, FX / Division 775, and the per-entity end-of-FY summary.

| ID | Use case | Entry point | Manual | Backed-by spec | Source |
|----|----------|-------------|--------|----------------|--------|
| UC-TAX-1 | View the per-entity EOY tax summary (net taxable + breakdown) | TAX → SUMMARY | [manual](tax.md#uc-tax-1) | R22, A1 | (to be implemented) |
| UC-TAX-2 | Switch financial year (AU FY = 1 Jul–30 Jun) | TAX → FY tab strip | [manual](tax.md#uc-tax-2) | R22 | (to be implemented) |
| UC-TAX-3 | Switch tax entity (Individual / SMSF / Company), per-entity discount | Top bar → SCOPE switcher | [manual](tax.md#uc-tax-3) | R17, A2 | (to be implemented) |
| UC-TAX-4 | View realised CGT disposals per parcel (12-month discount, carry-forward) | TAX → CGT → tap a disposal | [manual](tax.md#uc-tax-4) | R17, A1, A22 | (to be implemented) |
| UC-TAX-5 | View option CGT events (capital default, premium linkage) | TAX → CGT → OPTION CGT | [manual](tax.md#uc-tax-5) | R20, A1, A14 | (to be implemented) |
| UC-TAX-6 | View income detail (dividends / franking / withholding / FITO) | TAX → INCOME | [manual](tax.md#uc-tax-6) | R18, A1, A10 | (to be implemented) |
| UC-TAX-7 | View the FX-gain disclosure + Division 775 forex-realisation line | TAX → FOREX | [manual](tax.md#uc-tax-7) | R19, R36, A19 | (to be implemented) |
| UC-TAX-8 | Change parcel-selection policy (FIFO / LIFO / min-gain / max-gain) | TAX → SUMMARY → policy toggle | [manual](tax.md#uc-tax-8) | R17, A1 | (to be implemented) |
| UC-TAX-9 | Read wash-sale flags + reasoning | TAX → CGT → wash-sale badge | [manual](tax.md#uc-tax-9) | R21, A22 | (to be implemented) |
| UC-TAX-10 | View tax optimisation (harvest / discount-timing / year-end what-if) | TAX → OPTIMIZE | (planned — W5) | A3 | (to be implemented) |
| UC-TAX-11 | Confirm a lodged FY is pinned to its snapshot policy | TAX → SUMMARY → FY lodged badge | [manual](tax.md#uc-tax-11) | R22, A23 | (to be implemented) |
| UC-TAX-12 | View Div 775 forex events on the SUMMARY | TAX → SUMMARY → forex events | [manual](tax.md#uc-tax-12) | R19, A19 | (to be implemented) |
| UC-TAX-13 | Confirm display overlays (cash, daily P/L) move ZERO tax cell (the R8 re-lock) | TAX → SUMMARY (overlay on/off) | [manual](tax.md#uc-tax-13) | R23, A18 | (to be implemented) |
| UC-TAX-14 | Export / share CGT & EOY reports (PDF + CSV) | TAX → EXPORT | (planned — W4) | A1 | (to be implemented) |

## 4. Config & settings — `(planned — W4)`

Entity / account setup, per-entity tax policy inputs, the Div 775 election, notifications and price alerts.

| ID | Use case | Entry point | Manual | Backed-by spec | Source |
|----|----------|-------------|--------|----------------|--------|
| UC-CONFIG-1 | Open CONFIG / view settings (AU + AUD locked) | CONFIG tab | (planned — W4) | R28 | (to be implemented) |
| UC-CONFIG-2 | View entities + accounts | CONFIG → ENTITIES | (planned — W4) | R4, R28 | (to be implemented) |
| UC-CONFIG-3 | Set per-entity option treatment (capital / revenue) | CONFIG → entity → OPTION TREATMENT | (planned — W4) | R20, R28 | (to be implemented) |
| UC-CONFIG-4 | Set the default parcel policy | CONFIG → TAX DEFAULTS → Default parcel policy | (planned — W4) | R17, R28 | (to be implemented) |
| UC-CONFIG-5 | Toggle the per-entity Division 775 forex-gain election | CONFIG → entity → Forex-gain election | (planned — W4) | R37, A19 | (to be implemented) |
| UC-CONFIG-6 | Read the Division 775 election explainer | CONFIG → entity → Forex-gain election → ▸ explainer | (planned — W4) | R37, A19 | (to be implemented) |
| UC-CONFIG-7 | Toggle notification preferences | CONFIG → NOTIFICATIONS | (planned — W4) | R23 | (to be implemented) |
| UC-CONFIG-8 | View / create / remove price-alert rules (ARMED / FIRED) | CONFIG → PRICE ALERTS | (planned — W4) | R23 | (to be implemented) |
| UC-CONFIG-9 | See a fired-alert banner | Any tab → top-pinned banner | (planned — W4) | R23 | (to be implemented) |

## 5. Watchlist — `(planned — W4)`

The WATCH tab (held = close marks, non-held = quote-pending). No spec contract yet — display-only,
behind the R23 display firewall; spec items land when the watchlist subsystem is specced.

| ID | Use case | Entry point | Manual | Backed-by spec | Source |
|----|----------|-------------|--------|----------------|--------|
| UC-CONFIG-10 | View the watchlist (held = close marks, non-held = quote-pending) | WATCH tab | (planned — W4) | R23 | (to be implemented) |
| UC-CONFIG-11 | Add a symbol to a watchlist | WATCH → ADD | (planned — W4) | R23 | (to be implemented) |
| UC-CONFIG-12 | Switch between / create watchlists | WATCH → LIST menu | (planned — W4) | R23 | (to be implemented) |
| UC-CONFIG-13 | Remove a watched row (edit mode) | WATCH → EDIT → row minus | (planned — W4) | R23 | (to be implemented) |

## 6. W6 backend lift — connections · `(planned — W6)`

Move `LedgerCore` verbatim behind the seams: a `FactStore` protocol over durable multi-tenant storage,
server-held tokens, scheduled + on-demand sync, a thin read API, and remote push. The acceptance bar is
**tax numbers BYTE-IDENTICAL to the on-device build**. These rows cite the **`backend-lift.md`** spec
namespace (tagged `[BL]`). The full PROPOSED server-era backlog (intended flows + acceptance) lives in
[`backlog-backend.md`](backlog-backend.md) **(planned — W6)**.

| ID | Use case | Entry point | Manual | Backed-by spec | Source |
|----|----------|-------------|--------|----------------|--------|
| UC-CONN-1 | Add a server-managed IBKR connection (token → server secret store) | Thin client → Connections → Add connection | [manual](onboarding-and-sync.md#uc-conn-1) | R1, R17, A11 [BL] | (to be implemented) |
| UC-CONN-2 | Hold N connections per tenant; remove one in isolation | Thin client → Connections → remove | (planned — W6) | R10, A6 [BL] | (to be implemented) |
| UC-CONN-3 | Read server sync state, tenant-isolated, with freshness | Thin client → Connections / Sync health | (planned — W6) | R9, R20, A5, A13 [BL] | (to be implemented) |
| UC-CONN-4 | See a degraded connection's last-good data honestly stale-labelled | Thin client → Connections → degraded card | (planned — W6) | R16, A10 [BL] | (to be implemented) |
| UC-CONN-5 | Assign / classify a discovered account to a tax entity (safe 0% fallback) | Thin client → Accounts → assign entity | (planned — W6) | R11, A7 [BL] | (to be implemented) |
| UC-CONN-6 | Resolve a cost-basis gap server-side (write round-trips, recomputes) | Thin client → Gaps → RESOLVE | (planned — W6) | R22, A14 [BL] | (to be implemented) |
| UC-CONN-7 | Re-pull / re-ingest the same statement server-side (idempotent, no cursor) | Thin client → Sync now | (planned — W6) | R7, A3 [BL] | (to be implemented) |

## 7. W6 backend lift — schedule, health & read · `(planned — W6)`

Unattended scheduled + on-demand sync, the Sydney-midnight snapshot, rate-limit throttling, the
thin-client read surface, sync-health, and remote push — all `backend-lift.md` namespace (`[BL]`).

| ID | Use case | Entry point | Manual | Backed-by spec | Source |
|----|----------|-------------|--------|----------------|--------|
| UC-SCHED-1 | Server pulls Flex on a schedule with no phone open (unattended) | Server cron → sync handler → FactStore | (planned — W6) | R12, R14, A8 [BL] | (to be implemented) |
| UC-SCHED-2 | Trigger an on-demand server sync from the thin client | Thin client → Sync health → SYNC NOW | (planned — W6) | R13, R15, A8, A9 [BL] | (to be implemented) |
| UC-SCHED-3 | Reconstruct + persist the 00:00 Australia/Sydney price snapshot server-side | Server cron (Sydney midnight) → snapshot | (planned — W6) | R12, A8 [BL] | (to be implemented) |
| UC-SCHED-4 | Respect the IBKR per-token rate limit (error 1018) under schedule + on-demand | Sync handler → per-connection throttle | (planned — W6) | R15, A9 [BL] | (to be implemented) |
| UC-HEALTH-1 | View server-computed projections from the thin iOS client (read API) | Thin client → POS / TAX (server state) | (planned — W6) | R19, A2, A12 [BL] | (to be implemented) |
| UC-HEALTH-2 | View sync health / last-pull status from the thin client | Thin client → Sync health | (planned — W6) | R21, A13 [BL] | (to be implemented) |
| UC-HEALTH-3 | Receive a remote price alert (server-evaluated, push-delivered) | Device push notification (FiredAlert seam) | (planned — W6) | R24, A15 [BL] | (to be implemented) |
| UC-HEALTH-4 | Confirm server tax numbers match the on-device build to the cent | QA harness → server vs device tax digest | (planned — W6) | R5, R23, A2, A4 [BL] | (to be implemented) |

---

## Alias map (old → canonical UC)

Every distinct old ID found by grepping the example (`grep -rhoE 'UC-[A-Z]+(-[A-Z]+)?-?[0-9]*'`),
translated to its canonical `UC-<AREA>-<n>`. A fixer rewrites the left column to the right column.
Synonyms that denoted the **same** user goal collapse to one canonical row.

### Onboarding & sync

| Old ID(s) | Canonical | Note |
|---|---|---|
| `UC-ONB-1`, `UC-ONB-FLEX` | **UC-ONB-1** | First-run onboarding via Flex live connect (same goal). |
| `UC-ONB-FILE` | **UC-ONB-2** | First-run onboarding via XML file import. |
| `UC-ONB-3` | **UC-ONB-3** | In-flow Flex-token helper (kept). |
| `UC-ONB-4` | **UC-ONB-4** | Retry a failed onboarding pull (kept). |
| `UC-INGEST-XML`, `UC-SYNC-1` | **UC-SYNC-1** | Import a local Flex XML statement / connect from SOURCES. |
| `UC-INGEST-WEB`, `UC-SYNC-6` (pull sense) | **UC-SYNC-2** | Flex Web Service pull (SendRequest→poll→GetStatement). |
| `UC-SYNC-2` | **UC-SYNC-3** | Add another Flex query (multi-query/account). |
| `UC-STITCH` | **UC-SYNC-4** | Stitch history across Flex's capped date ranges. |
| `UC-SYNC-5`, `UC-RESYNC-NOW` | **UC-SYNC-5** | "Sync now" re-pull from the stored token. |
| `UC-SYNC-6` (refresh sense), `UC-IDEMPOTENT`, `UC-REFRESH-1` (IBKR-refresh sense) | **UC-SYNC-6** | Refresh-from-IBKR / safe idempotent re-import. |
| `UC-SYNC-7`, `UC-RECONCILE` | **UC-SYNC-7** | View sync state + reconcile to the cent (read-only tie-out). |
| `UC-SYNC-3` | **UC-SYNC-8** | Reconnect after a degraded launch. |
| `UC-DISCONNECT`, `UC-SYNC-9` | **UC-SYNC-9** | Disconnect the IBKR connection. |
| `UC-GAP-INBOX`, `UC-SYNC-10` | **UC-SYNC-10** | View the cost-basis data-gap inbox. |
| `UC-GAP-FILL`, `UC-SYNC-11` | **UC-SYNC-11** | Resolve a cost-basis gap with a manual fill. |
| `UC-FOREX-GAP`, `UC-SYNC-12` | **UC-SYNC-12** | Inspect a flagged forex-leg gap. |
| `UC-SYNC-13` | **UC-SYNC-13** | Snooze / ignore a cost-basis gap. |

### Positions, cash & holdings

| Old ID(s) | Canonical | Note |
|---|---|---|
| `UC-POS-1`, `UC-POSITIONS` | **UC-POS-1** | View the holdings list. |
| `UC-POS-2` | **UC-POS-2** | Filter by asset class. |
| `UC-POS-3`, `UC-POS-GROUP` | **UC-POS-3** | Group (flat / account / entity). |
| `UC-POS-4` | **UC-POS-4** | Sort holdings. |
| `UC-POS-5` | **UC-POS-5** | Portfolio summary block. |
| `UC-POS-6`, `UC-POS-7`, `UC-POS-FLAGS` | **UC-POS-6** | Read GAP/WRITTEN flags + jump to Sync (synonyms collapsed). |
| `UC-SCOPE`, `UC-SCOPE-1` | **UC-POS-7** | Switch scope (consolidated / entity / account). |
| `UC-TODAY-1`, `UC-TODAY-3`, `UC-DAILY-PL` | **UC-POS-8** | Per-position daily P/L vs midnight snapshot (frozen-FX). |
| `UC-REFRESH-1` (prices sense) | **UC-POS-9** | Manually refresh delayed prices. |
| `UC-CASH`, `UC-CASH-1` | **UC-CASH-1** | Per-currency cash balances, signed. |
| `UC-CASH-2` | **UC-CASH-2** | The CASH hero. |
| `UC-HOLDING`, `UC-HOLD-1` | **UC-HOLD-1** | Open a holding's detail. |
| `UC-HOLD-2` | **UC-HOLD-2** | Parcels through a split / corporate action. |
| `UC-HOLD-3` | **UC-HOLD-3** | Option events + premium linkage. |
| `UC-EXPORT-1`, `UC-EXPORT-PORTFOLIO` | **UC-EXPORT-1** | Export the portfolio. |
| `UC-EXPORT-2` | **UC-EXPORT-2** | Export a single holding. |

> **Range-shorthand fold.** `UC-TODAY-2` and `UC-HOLD-4`, `UC-HOLD-5`, `UC-HOLD-6` appear only inside
> range shorthands (`UC-TODAY-1/2/3`, `UC-HOLD-1…6`) in the UX manual, never as distinct specced
> goals. A fixer translates `UC-TODAY-2 → UC-POS-8` (today) and any of `UC-HOLD-4 / UC-HOLD-5 /
> UC-HOLD-6 → UC-HOLD-1` (holding-detail). Add a distinct canonical row only when one is specced.

### Tax

| Old ID(s) | Canonical | Note |
|---|---|---|
| `UC-TAX-1`, `UC-TAX-SUMMARY` | **UC-TAX-1** | EOY tax summary. |
| `UC-TAX-FY` | **UC-TAX-2** | Switch financial year. |
| `UC-TAX-3`, `UC-TAX-ENTITY` | **UC-TAX-3** | Switch tax entity / per-entity discount. |
| `UC-TAX-4`, `UC-CGT-DISPOSALS` | **UC-TAX-4** | Realised CGT disposals per parcel. |
| `UC-TAX-5`, `UC-CGT-OPTIONS` | **UC-TAX-5** | Option CGT events. |
| `UC-TAX-6`, `UC-TAX-INCOME` | **UC-TAX-6** | Income / franking / FITO detail. |
| `UC-TAX-7`, `UC-TAX-FOREX` | **UC-TAX-7** | FX-gain disclosure + Div 775 line. |
| `UC-TAX-8`, `UC-PARCEL-POLICY` | **UC-TAX-8** | Change parcel-selection policy. |
| `UC-WASH-SALE` | **UC-TAX-9** | Wash-sale flags + reasoning. |
| `UC-TAX-OPTIMIZE` | **UC-TAX-10** | Tax optimisation (harvest / what-if). |
| `UC-TAX-11` | **UC-TAX-11** | Lodged-FY pin. |
| `UC-TAX-12`, `UC-DIV775` | **UC-TAX-12** | Div 775 forex events on the summary. |
| `UC-TAX-RELOCK` | **UC-TAX-13** | Display overlays move ZERO tax cell (R8 re-lock). |
| `UC-TAX-EXPORT` | **UC-TAX-14** | Export CGT & EOY reports. |

### Config, settings, alerts & watchlist

| Old ID(s) | Canonical | Note |
|---|---|---|
| `UC-CONFIG`, `UC-CONFIG-` | **UC-CONFIG-1** | Open CONFIG / view settings. |
| `UC-CONFIG-ENTITIES` | **UC-CONFIG-2** | View entities + accounts. |
| `UC-OPTION-TREATMENT` | **UC-CONFIG-3** | Set per-entity option treatment. |
| `UC-PARCEL-DEFAULT` | **UC-CONFIG-4** | Set the default parcel policy. |
| `UC-CONFIG-5`, `UC-DIV775-ELECTION` | **UC-CONFIG-5** | Toggle the Div 775 election. |
| `UC-CONFIG-6` | **UC-CONFIG-6** | Read the Div 775 election explainer. |
| `UC-NOTIFY-PREFS` | **UC-CONFIG-7** | Notification preferences. |
| `UC-ALERT-RULES` | **UC-CONFIG-8** | Price-alert rules (armed / fired). |
| `UC-ALERT-BANNER` | **UC-CONFIG-9** | Fired-alert banner. |
| `UC-WATCHLIST` | **UC-CONFIG-10** | View the watchlist. |
| `UC-WATCH-ADD` | **UC-CONFIG-11** | Add a watchlist symbol. |
| `UC-WATCH-LISTS` | **UC-CONFIG-12** | Switch / create watchlists. |
| `UC-WATCH-REMOVE` | **UC-CONFIG-13** | Remove a watched row. |

### W6 backend lift (connections / schedule / health)

| Old ID(s) | Canonical | Note |
|---|---|---|
| `UC-CONN`, `UC-CONN-1` | **UC-CONN-1** | Add a server-managed connection. |
| `UC-CONN-2` | **UC-CONN-2** | N connections / isolated removal. |
| `UC-CONN-3` | **UC-CONN-3** | Tenant-isolated server read + freshness. |
| `UC-CONN-4` | **UC-CONN-4** | Degraded connection, honest stale label. |
| `UC-CONN-5`, `UC-ACCT-2` | **UC-CONN-5** | Assign a discovered account to an entity (safe fallback). |
| `UC-CONN-6` | **UC-CONN-6** | Resolve a gap server-side (write round-trip). |
| `UC-CONN-7` | **UC-CONN-7** | Idempotent server re-ingest (no cursor). |
| `UC-ACCT-1`, `UC-ACCT-3` | **UC-CONN-1** | [MT] sign-up/delete journey — server side folds into the connection lifecycle (Path B UI deferred). |
| `UC-ACCT-4` | **UC-HEALTH-1** | Read round-trip / token-never-in-response — covered by the server read surface. |
| `UC-SYNC-SCHED-1`, `UC-UNATTENDED-SYNC`, `UC-FRESH-1` | **UC-SCHED-1** | Unattended scheduled sync (app closed). |
| `UC-SYNC-TRIGGER-1`, `UC-TRIGGER-SYNC`, `UC-FRESH-2` | **UC-SCHED-2** | On-demand server sync trigger. |
| `UC-SNAPSHOT-SCHED` | **UC-SCHED-3** | Server-side Sydney-midnight snapshot. |
| `UC-RATE-THROTTLE`, `UC-FRESH-3` | **UC-SCHED-4** | Per-token rate-limit throttle. |
| `UC-VIEW-SERVER-STATE`, `UC-READ-1`, `UC-READ-3` | **UC-HEALTH-1** | View server-computed projections (thin client read API). |
| `UC-READ-2` | **UC-HEALTH-1** | Read last-good projections after a failed pull (same read surface). |
| `UC-SYNC-HEALTH`, `UC-SYNCHEALTH-1` | **UC-HEALTH-2** | View server sync health / last-pull status. |
| `UC-REMOTE-PUSH`, `UC-PUSH-1`, `UC-PUSH-2` | **UC-HEALTH-3** | Server-evaluated alert, push-delivered. |
| `UC-BYTE-IDENTICAL` | **UC-HEALTH-4** | Server tax == device tax to the cent (the wave bar). |

> **Stray / partial captures from grep** — `UC-CONFIG-`, `UC-SYNC-`, `UC-VIEW-SERVER-` are
> truncations of `UC-CONFIG-1`, the `UC-SYNC-*` set, and `UC-VIEW-SERVER-STATE → UC-HEALTH-1`
> respectively; they carry no independent meaning.

---

## Coverage notes & known gaps

- **Real-data path (W2) is in progress.** Onboarding/sync rows (UC-SYNC-2/4/5/7) and the
  foreign-currency correction (UC-CASH-1/2, UC-TAX-7, UC-CONFIG-5) are spec-backed but
  `(to be implemented)` until the entry views/handlers land; fill `Source` then.
- **The R8 firewall is a first-class use case.** UC-TAX-13 (and UC-HEALTH-4 on the backend) make the
  "display overlays move ZERO tax cell" invariant user-visible and QA-assertable, backed by the
  non-vacuous, mutation-proven `A18` (on device) / `A4` `[BL]` (server) tests.
- **W6 is Path A (single-tenant) for now.** The multi-tenant SaaS identity journey (former
  `UC-ACCT-1/3/4`) is Path B and is folded into the connection lifecycle / server read surface above
  rather than given standalone rows; promote to first-class `UC-CONN-*` / a new `ACCT` area only when
  Path B is specced.
- **Every `Backed-by spec` ID exists in `specs/`.** Non-`[BL]` cells resolve in `domain-core.md`
  (`R1–R39`, `A1–A29`, all flat); `[BL]` cells resolve in `backend-lift.md`
  (`R1–R24`, `A1–A16`). A citation to an undefined number is a gap reviewers FAIL.
- **Only three manuals exist** (`onboarding-and-sync.md`, `positions-and-holdings.md`, `tax.md`); all
  other areas carry `(planned — Wn)` with no link until their manual ships.
</content>
</invoke>
