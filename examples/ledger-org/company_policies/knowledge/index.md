# Knowledge Index

The **shared** MAP + GOTCHAS + LESSONS that make the org repo-familiar. One store, read by everyone, **gardened only by HR** (the three-tier MEMORY→knowledge→spec promotion loop + the per-wave GC run in `handbook.md` and each `performance/<date>.md` review). A persona scans this index, then loads **only** the `<domain>.md` file(s) it touches — knowledge never loads wholesale into context.

**The one rule, enforced not aspirational — REFERENCE, NEVER RESTATE.** Each entry holds the *map* (where a thing lives), the *gotcha* (the trap that silently bites), and the *lesson* (what to do instead), then **points to the authoritative artifact** (`spec R#/A#`, `architecture.md §x`, `data-model.md §x`, `File.swift:line`). If the content is a fact, it belongs in a spec, not here. Each `<domain>.md` is **budgeted** (≤ ~150 lines / ≤ ~12 entries); every wave HR runs a **GC** — dedup, merge, supersede stale entries, prune anything now restated by a spec or pointing at a dead `File.swift:line`. Knowledge that duplicates a spec is deleted on the next GC. Growth without pruning is an HR failure mode.

| domain | file | covers (5–8 words) | entries | last GC |
|--------|------|--------------------|---------|---------|
| ledgercore | ledgercore.md | Foundation-only seam, determinism, FactStore/ProjectionStore lift | 3 | 2026-05-31 |
| tax | tax.md | R23 no-display-moves-tax, Div775 FX, CGT discounts | 4 | 2026-05-29 |
| ingest | ingest.md | Flex XML, FactID dedup, netCash, forexConversion | 3 | 2026-05-31 |
| ui | ui.md | RenderedMoney chokepoint, demo-retired copy-truth, signed cash | 3 | 2026-05-31 |

**Domain map** (what each file owns, and the authoritative source it points back to — never restates):

- **ledgercore** — the Foundation-only core seam: the deterministic fold (D8 — no `Date()`/`UUID()`/`Locale.current` inside `Projection.build`), `Money = Decimal` never `Double` (D7), the append-only JSONL fact log + `(eventDate, stableSourceID, FactID)` total order, and the **W6 server-lift seams** (`FactLogStore`/`ProjectionStore`/`RawRecordSource`/`QuoteSource`, `PortabilityLockTests` guarding Foundation-only). Authority: `architecture.md §2/§6/§10`, `data-model.md §1 (I1–I9)/§6/§8`, `system-design.md §7.2`, `specs/domain-core.md`, `specs/backend-lift.md`.
- **tax** — the per-entity tax pipeline traps: the **R8 firewall** (`specs/domain-core.md` R23 — display/cash/daily-P/L/snapshot/quotes NEVER move a tax cell; the only intentional mover is `Div775Engine`, D74/D84), CGT per-entity discounts (IND 50% / SMSF 33⅓% / Company 0%, fail-SAFE on unassigned accounts I8/I9), the Div775 same-day-inflows-before-outflows net-short fold (D84), and the byte-identical-tax acceptance property (A18, mutation-proven). Authority: `specs/domain-core.md` R23/R36 + A18/A19/A20, `data-model.md §9`, architecture.md §"The R23 lock", `test-strategy.md`.
- **ingest** — the raw-data path traps: Flex XML → `RawRecord` → `Reconciler` → `[Fact]`, `FactID = hash(stableSourceID, kind)` content-addressed dedup (re-import is a no-op, A5/I5), the `attributes["netCash"]` display-only trade cash leg (use IBKR `<Trade netCash>` verbatim, never the `nativeAmount−fee` re-derivation — D70/D71), and `.forexConversion` (no instrument/no parcel, both legs sourced not fabricated — D74). Authority: `architecture.md §2.1–§2.3`, `data-model.md §3/§4/§6/§13`, `specs/domain-core.md`, `use-cases/onboarding-and-sync.md`.
- **ui** — the SwiftUI thin-client traps: the `RenderedMoney` **drawn==asserted** render chokepoint (a test that asserts the VM input is vacuous — assert the bytes that DRAW, `TaxSummaryView.swift:272`), the **demo-retired** copy-truth invariant (shipping app NEVER shows demo holdings; enforced by `WaveCodeCopyLockTests` on every surface, D85), signed cash never clamped (OWED as dim text not a badge, D64/DPC-M1), and the app-level export presenter (sheet-per-context limit, D82). Authority: `architecture.md §5/§2.8`, `ux/positions.md`, `use-cases/`, `specs/`.

When a `<domain>.md` file is first created by HR promotion, replace its `0` / `—` cells above with the real `entries` count and `last GC` date so HR can spot stale or bloated domains at a glance.
