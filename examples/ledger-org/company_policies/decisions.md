# Decisions — Ledger

**This file is the canonical authority for decision invariants.** Per
`company_policies/handbook.md`, every decision cited anywhere in the example
(`D<n>`, `E<n>`) MUST resolve to an entry defined here. Nothing may cite a `D#`/`E#`
that this file does not define; this file is a **superset** that defines the full
`D1..D85` range plus `E1`, so any citation resolves.

Reversible/internal decisions are made and logged here. Direction-changing decisions
(irreversible, spend, externally visible, strategic/architecture-locking) are **escalated
to the founder** (see `references/decision-escalation.md`) and logged once answered.

Each entry records **what outcome proved a decision right** — its *outcome-harness* (an
acceptance test, a reconciliation to-the-cent, a byte-identical replay, a benchmark, or —
for an escalation — the routing process itself), not who approved it. Per
`references/automation.md`, a reversible technical choice with no outcome to judge it is
not ready to make. The escalation bar is sharper than "irreversible": **does this
significantly change direction?** — pricing, business model, public/brand naming, a
platform bet that locks future waves. Everything else: decide, harness, log, proceed.

Every entry states: **Reversible?** · **Blast radius?** · **Rationale** ·
**Outcome-harness** (the result that proves it), in one line where the decision is small.

## Alias map (old → canonical)

- The byte-identical-tax / R8-no-tax-leak property is **A18** (`display-never-moves-tax`) in
  `specs/domain-core.md` "## Key acceptance anchors" — the full `TaxProjection` is byte-identical
  whether or not display/cash/daily-P/L overlays are present.
- Historical dotted DPC/FXC IDs resolve to the FLAT `A#` for the named concept in the cited spec
  (`server-tax-byte-identical`, `tax-to-the-cent-vs-oracle`, `web-source==file-source`,
  `FactStore-seam-preserves-replay` in `specs/backend-lift.md`; cash/frozen-FX/byte-identical-tax/
  Div775/no-double-count in `specs/domain-core.md`). No dotted spec IDs exist; look the concept up in
  the cited file's "## Key acceptance anchors" table.
- "FactStore-protocol decision" (older prose name) → **D-seam**, the W6 `FactStore` seam
  (defined under "Logged by architect (W6 backend-lift design)" below); harnessed by
  `FactStore-seam-preserves-replay` / `server-tax-byte-identical`.

---

## Logged (made autonomously)

### D1 — 2026-05-24 — Project layout
- **Decision.** App + domain core live at `/Users/dev/git/ledger/`; company ledger at `ledger/docs/superpowers/company/ledger/`.
- **Reversible?** Yes. **Blast radius?** Internal/cosmetic.
- **Rationale.** A single repo root keeps the portable core, the app, and the company docs co-located.
- **Outcome-harness.** Builds resolve from the documented paths; no harness beyond "it builds where the docs say."

### D2 — 2026-05-24 — Working app name "Ledger"
- **Decision.** Name "Ledger"; bundle id `com.solo.ledger` as a placeholder (final App Store display name + id are a later strategic call).
- **Reversible?** Yes (the final naming is escalated when it matters). **Blast radius?** Internal placeholder.
- **Rationale.** Unblocks targets/signing without pre-committing the public name.
- **Outcome-harness.** Build + sign succeed under the placeholder id; the public name remains an open founder call (cf. E1 for naming-class escalations).

### D3 — 2026-05-24 — Build/test commands run with the bash sandbox DISABLED (environment note)
- **Decision.** Run ALL of `swift build`, `swift test`, `xcodegen generate`, `xcodebuild` with `dangerouslyDisableSandbox: true`.
- **Reversible?** N/A — environment fact, not a product decision. **Blast radius?** Tooling only.
- **Rationale.** Two confirmed causes: (a) `simctl`/sim boot/`xcodebuild` on a sim destination hit a blocked CoreSimulator daemon socket; (b) even pure `swift build`/`swift test`/`xcodegen` fail because SwiftPM applies its OWN nested `sandbox-exec` inside ours and the `~/Library/org.swift.swiftpm` caches are read-only.
- **Outcome-harness.** Sandboxed runs reproducibly fail with `sandbox_apply: Operation not permitted` / "Connection refused"; the disabled run is green.

### D4 — 2026-05-24 — Two-target shape (portable `LedgerCore` + thin SwiftUI app)
- **Decision.** A pure-Swift `LedgerCore` SwiftPM library (Foundation only, zero UIKit/SwiftUI/SwiftData) + a thin SwiftUI app depending on it.
- **Reversible?** Yes, but load-bearing. **Blast radius?** Internal; satisfies the portable-domain-core lock + the W6 server-lift seam.
- **Rationale.** "UI is a thin client"; the core must lift verbatim to a server.
- **Outcome-harness.** The portability lock test (no Apple-UI import in `LedgerCore`) stays green; the same core later runs in Lambda (see the W6 `FactStore` seam).

## Logged by architect (W1 design phase, 2026-05-24)

### D5 — 2026-05-24 — Persistence: append-only Codable JSONL fact log (truth) + in-memory projections behind a `ProjectionStore` seam
- **Decision.** One `Fact` per line — JSON with `.sortedKeys` + fixed `Decimal` formatting — appended in Application Support; `append` dedupes on `FactID`. Projections are rebuilt by replaying the log on launch and cached in RAM (`InMemoryProjectionStore`); a `ProjectionStore` protocol is the seam so a `SQLiteProjectionStore` (GRDB) can slot in later. Chosen over SwiftData/Core Data / a relational store from day one.
- **Reversible?** Yes — the `ProjectionStore` seam isolates storage from the tax/projection math. **Blast radius?** Internal but load-bearing: the JSONL bytes are the canonical truth the W6 server lift moves behind a `FactStore` protocol, so the *format* is a long-lived contract.
- **Rationale.** Keeps the core Foundation-only/portable (D4); W1's data is tiny so a DB is YAGNI; canonical JSONL makes re-import (A5) and projection rebuild (A6) cheap to prove.
- **Outcome-harness.** **A5** (byte-identical re-import: `FactID` dedupe leaves `count` unchanged) and **A6** (byte-for-byte projection rebuild: replaying yields an identical digest), both CI acceptance tests. GREEN at the W1 boundary (digest `33ad4db4…`, FactID `f7afeec2e435e0f6`, 49 facts) and held byte-identical through every later wave.

### D6 / O1 — 2026-05-24 — `reports-catalog.md` referenced by the spec but not supplied → derive from the prototype + AU tax norms (RESOLVED by boss)
- **Decision.** The boss chose **derive** the reports catalog from the handoff prototype + AU tax norms; tech-lead/designer author `reports-catalog.md` as a W1 deliverable; boss reviews the derived list at the W1 boundary.
- **Reversible?** Yes. **Blast radius?** Internal (W1 scope).
- **Rationale.** The missing artifact (O1) blocked report subtasks; deriving it from real sources beats inventing one.
- **Outcome-harness.** Each catalog entry maps to exactly one projection output; the boss ratifies the derived list at the W1 boundary.

### D7 — 2026-05-24 — `Money` is `Decimal`, never `Double`
- **Decision.** All currency/CGT/franking/FX math uses `Decimal` at a fixed scale with `.sortedKeys` Codable. No `Double` on the money path.
- **Reversible?** Only in the trivial (tightening) direction — the harness forbids reverting to `Double`. **Blast radius?** Internal but pervasive: the spine of "accurate to the cent."
- **Rationale.** `Double` cannot represent `0.10` exactly; a cent of drift compounds across a parcel ledger and breaks both the tax oracle and byte-reproducibility.
- **Outcome-harness.** The hand-derived **tax oracle** (IND × SMSF, FY25 × FY26) matches the engine **to the cent**; fixed `Decimal` is also a precondition of A6 (stable JSONL bytes). All oracle EOY/CGT cells exact; tax byte-identical on the on-device build.

### D8 — 2026-05-24 — Determinism contract for projections
- **Decision.** Facts folded in `(eventDate, stableSourceID)` order; no `Date()`/`UUID()`/locale reads inside `build`; `today`, FX, and parcel policy are explicit `ProjectionConfig` inputs.
- **Reversible?** Yes (it is a code-review + test discipline). **Blast radius?** Internal; underpins A6.
- **Rationale.** A pure, ordered fold is what makes the replay digest reproducible.
- **Outcome-harness.** Enforced by code review + the **A6** test; any hidden clock/locale read breaks the digest and goes RED.

### D9 — 2026-05-24 — No `WaveBadge`/W3 toggle in the shipped W1 app
- **Decision.** The prototype's W1↔W3 wave switch is a mock affordance, not a feature; W1 ships close-mark only. The `PriceOverlay` seam + alert engine + `Spark` are built but wired to close-mark data (R25 scaffolds).
- **Reversible?** Yes. **Blast radius?** Internal. (Later superseded on the demo front by D85's retire-sample-data direction.)
- **Rationale.** Ship the real W1 surface; don't expose mock toggles.
- **Outcome-harness.** W1 acceptance renders close-mark data only; the scaffolded seams compile and stay dormant until W3.

### D10 — 2026-05-24 — CSV in core, PDF in app
- **Decision.** Report *content* (`Report` value + CSV) lives in `LedgerCore` (pure, golden-file tested); PDF *rendering* lives in the app target (needs platform graphics).
- **Reversible?** Yes. **Blast radius?** Internal; keeps report content portable to the W6 server.
- **Rationale.** Portability lock — content must not depend on Apple graphics.
- **Outcome-harness.** Golden-file CSV tests in core; the same `Report` value drives both CSV and PDF (cf. D27).

### D11 — 2026-05-24 — Review + QA gates run concurrently against the same immutable artifact
- **Decision.** Per `delivering-a-subtask`, every build subtask is gated by an independent `persona-code-reviewer` and `persona-qa`, dispatched as fresh subagents **concurrently** against the same immutable artifact (neither mutates code → no race; independent verification preserved). If EITHER returns FAIL, the builder loops on the union of issues and the failed gate(s) re-run.
- **Reversible?** Yes — purely scheduling; serialising is a one-line change. **Blast radius?** Internal (process/latency).
- **Rationale.** Series doubles latency for no integrity gain *as long as neither edits source*.
- **Outcome-harness.** A FAIL must reflect a real defect: re-run any FAIL **serially**; if the verdict flips, concurrency was unsound. This harness fired and caught a reviewer false-negative when mutation-probes raced — see **D58**, which carves out the exception (mutation-probe on the same file → run sequentially).

## Logged by backend-engineer (M3-T2 + M3-T3, 2026-05-24)

### D12 — 2026-05-24 — Split CGT-guard model: SUPERSEDE, don't mutate
- **Decision.** `adjustCostBase(ratio:)` replaces each affected parcel *record* with a NEW post-split record (`quantity × factor`, total `costBaseNative` conserved, `splitAdjusted` flag), never mutating the preserved `let` quantity/cost base; no `DisposalLine` (a split is not a disposal). "Acquired on or before the action date" falls out of the `(eventDate, stableSourceID)` fold order.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Conserve total cost base, fabricate no disposals, keep the CGT guard immutable.
- **Outcome-harness.** Total cost base conserved across the NVDA 10:1 split; CGT-engine tests stay green; no fabricated disposal.

### D13 — 2026-05-24 — DRP-restate attaches to the latest prior open parcel
- **Decision.** `adjustCostBase(delta:)` (e.g. BHP +18.40) adds the amount to the LATEST parcel acquired on or before the action date, flagged `costBaseRestated`; quantity unchanged.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** The fixture's restate is a single cost-base delta; "latest prior open parcel" is deterministic.
- **Outcome-harness.** The BHP restate hits parcel P-B2 (never FIFO-consumed in FY25/FY26), so oracle cells stay exact.

### D14 — 2026-05-24 — Option open/close is net-position-inferred in the FOLD; the interpreter stays pure
- **Decision.** The interpreter stays stateless (negative-qty trade → `.close`); the ordered fold decides open/close, opens a SHORT written parcel on a shortfall (`origin .writtenOption`), tags lifecycle events `settleClose`, and emits a second routed mutation opening the UNDERLYING at strike on exercise/assignment. Cross-key routing lives in `LotLedger.buildAll` via `RoutedMutation`.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** IBKR Flex omits `openCloseIndicator`; the stateful layer is the only place that can disambiguate.
- **Outcome-harness.** Lot-ledger tests incl. a cross-key assignment case; the option↔underlying link (R14) holds.

### D15 — 2026-05-24 — Reconciler carries the OptionEAE premium
- **Decision.** `Reconciler.optionEventFact` writes the OptionEAE `tradePrice` into `attributes["premium"]` and `nativeAmount` so the option↔underlying link can fold it (previously dropped).
- **Reversible?** Yes (additive). **Blast radius?** Internal.
- **Rationale.** The premium is needed for the D14 cost-base credit.
- **Outcome-harness.** No existing reconciler test regressed; the premium folds into D14.

## Logged by backend-engineer (M4-T1 + M4-T2, 2026-05-24)

### D16 — 2026-05-24 — One canonical fact-log byte form, shared by store and engine (`FactLog`)
- **Decision.** Extract `canonicalData`/`digest`/`order` into `Facts/FactLog.swift`; `FactStore.digest()`/`canonicalLogData()`/`factOrder` and `ProjectionEngine.factDigest([Fact])` all delegate to it.
- **Reversible?** Yes (DRY refactor, no behavior change). **Blast radius?** Internal.
- **Rationale.** Guarantees the in-memory digest is byte-identical to `FactStore.digest()` for the same fact set.
- **Outcome-harness.** `digestMatchesFactStore`; A6 holds whether facts came from disk or RAM.

### D17 — 2026-05-24 — Pricing/FX seams defined now with trivial defaults; full impls deferred to M5-T1
- **Decision.** Add `PriceOverlay` + a minimal `FXResolver` with `NullPriceOverlay`/`IdentityFXResolver` defaults so `ProjectionInput` is fully typed now; `StatementCloseMarkOverlay` + the full FX resolver stay M5-T1.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Type the seam now; defer the real impls.
- **Outcome-harness.** Engine + tests run on the defaults; M5-T1 swaps in the real overlays with no engine change.

### D18 — 2026-05-24 — `incremental` is faithful-but-simple: rebuild affected keys over COMBINED facts via `buildAll`, reuse untouched keys
- **Decision.** Index ΔF → affected `LotLedgerKey`s routing-aware, write back only affected keys' states from `buildAll(combinedFacts)`, re-run the pure `build` over the merged input.
- **Reversible?** Yes (a finer per-key fold is a later perf optimization, YAGNI). **Blast radius?** Internal.
- **Rationale.** The merged input is value-equal to a full rebuild and `build` is pure → byte-identical output.
- **Outcome-harness.** The M4-T4 invariant: `incremental(F0,ΔF) == rebuild(F0 ∪ ΔF)`, tested incl. a cross-key assignment case.

### D19 — 2026-05-24 — `ProjectionStore.save` takes an explicit `(_ projection: P.Type, _ stamp:)`
- **Decision.** Add the explicit projection-type argument so `P` (hence the `P.name` cache key) is determined; the sketch's signature did not compile.
- **Reversible?** Yes (minor shape tweak). **Blast radius?** Internal.
- **Rationale.** `P` is not inferable from `StampedProjection<P.Output>` alone.
- **Outcome-harness.** Compiles; `load`/`drop` already took the type, so the seam is symmetric.

### D20 — 2026-05-24 — `InMemoryProjectionStore` is `final class` + `NSLock`, `@unchecked Sendable`
- **Decision.** Guard every access with an `NSLock` (Foundation-only); add `loadOrRebuild(_:from:)`.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Satisfy `ProjectionStore: Sendable` and be genuinely concurrency-safe without an unstated single-thread assumption; no Combine/Apple-UI import (portability lock).
- **Outcome-harness.** Stored values are `Sendable`; the portability lock stays green.

## Logged by backend-engineer (M4-T3, 2026-05-24)

### D21 — 2026-05-24 — Implemented vs deferred `LedgerStore` actor methods (no fakes)
- **Decision.** IMPLEMENT `ingest`/`ledgerStats`/`setConfig`/`setPriceOverlay`/generic `projection<P>`; mark `positions`/`tax`/`optimise`/`gaps`/`resolveGap` as honest `// M5-T1 / M9` markers (NOT fake data).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** An honest gap beats fabricated data; each deferred method becomes a one-line wrapper when its projection lands.
- **Outcome-harness.** No fabricated data ships; the markers convert to wrappers in later milestones.

### D22 — 2026-05-24 — Config/overlay changes invalidate via `dropAll()`, not digest-staleness
- **Decision.** Add `InMemoryProjectionStore.dropAll()` (NSLock-guarded, Foundation-only) and call it from `setConfig`/`setPriceOverlay`, because a `ProjectionConfig`/`PriceOverlay` change is NOT in the fact digest.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** A digest-only check would serve a stale cached output after a config/overlay change.
- **Outcome-harness.** A config-reading test projection (fifo→minGain) reflects the change only after the drop.

### D23 — 2026-05-24 — `ingest` invalidates only when the fact set actually changed
- **Decision.** A no-op idempotent re-ingest (A5 dedupe ⇒ `count` unchanged) skips invalidation; a real append invalidates. (Asymmetry vs `setConfig` — D22 — is intentional.)
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Keep double-ingest cheap and provably idempotent through the actor boundary.
- **Outcome-harness.** A5 idempotence holds through the actor; a real append rebuilds.

### D24 — 2026-05-24 — Collaborators injected with W1 defaults
- **Decision.** `LedgerStore` injects `FactStore`/`ProjectionStore`/overlay/FX/config (defaults InMemory/Null/Identity); `FactStore` is a single-owner non-`Sendable` `final class` reached only through the actor.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Tests use a temp `FactStore`; M5-T1 wires the real overlay/FX with no actor change.
- **Outcome-harness.** The compiler's `sending` check enforces single-owner access.

## Logged by backend-engineer (M5-T8 rework, 2026-05-24)

### D25 — 2026-05-24 — Tax CGT is sourced from the CORP-ACTION-REBASED lot-ledger parcels, with commission joined back from the acquisition fact (R13)
- **Decision.** Build `TaxProjection.realisedCGT`'s pool from `input.lotLedgers[*].parcels` (post-split qty, conserved cost base; DRP delta folded), filtered to the entity's non-option open long parcels; join acquisition commission back per ledger key by replaying the deterministic fold order.
- **Reversible?** Yes. **Blast radius?** Internal but correctness-load-bearing (a split holding could otherwise mis-cost or crash).
- **Rationale.** The original built the pool from raw facts, dropping CA-1/CA-2 from the *tax* cost base (R13 violation).
- **Outcome-harness.** The 4 oracle EOY cells stay exact; NVDA tax parcels show 390 split-rebased shares; a synthetic 100-share disposal costs to the cent (gross 27640.64, taxable 13820.32).

### D26 — 2026-05-24 — `CGTEngine.match` over-consumption is a TYPED GAP, not a process-aborting trap
- **Decision.** Replace the "insufficient parcel quantity" `precondition` with a recoverable `CGTDisposalResult.dataGap` (requested/matched/shortfall/reasoning); the disposal becomes a PARTIAL costed over matched units.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** A `precondition` can abort mid-run and mask which results are real; the product's stance is "honest about data gaps" (R7-gaps).
- **Outcome-harness.** The fully-matched path is byte-identical (`dataGap == nil`); existing CGTEngine/TaxProjection tests stay green.

## Logged by backend-engineer (M7-T1, 2026-05-24)

### D27 — 2026-05-24 — One `Report` value type for every report, with typed `Cell`s
- **Decision.** RPT-1…11 share `Report` (title/scope/fy + `Section[]`); `Cell` is a typed enum (`text`/`money`/`decimal`/`percent`/`int`/`bool`/`empty`). `Codable`/`Sendable`/`Hashable`, Foundation-only.
- **Reversible?** Yes. **Blast radius?** Internal; portable to the W6 server.
- **Rationale.** Deterministic formatting; the same value drives both CSV (core) and PDF (app) without duplicated layout (D10).
- **Outcome-harness.** Golden-file CSV tests; PDF renders the same `Report` value.

### D28 — 2026-05-24 — CSV dialect: RFC-4180, CRLF, en-AU decimal convention, NO thousands grouping
- **Decision.** Quote a field iff it contains `,`/`"`/CR/LF (inner quotes doubled); CRLF; `REPORT`/`SCOPE` provenance header; sections as blank line + `# heading` + column header + rows. Numbers use the en-AU decimal convention (money 2dp via `Money.canonicalString`, percent 2dp bankers') but OMIT thousands grouping; all formatting is locale-INDEPENDENT (`en_US_POSIX`).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** en-AU thousands grouping uses a comma → would collide with the separator and harm byte-stability; omitting it means a number never needs quoting.
- **Outcome-harness.** Byte-stable golden CSVs under `Tests/.../Fixtures/reports/` (regen via `LEDGER_REGEN_GOLDEN=1`); bytes never drift (D8).

### D29 — 2026-05-24 — Buildable-now set = RPT-1,2,3,4,5,6,7,8,9,11; RPT-10 (Cost-Basis Gap) DEFERRED to post-M9
- **Decision.** Build every report whose projection exists now; defer RPT-10 (needs `GapsProjection`, M9-T2) behind a `// M9-T2: GapsProjection` marker; RPT-3 renders its projection-backed slice now.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Every report renders exactly one projection's output; inventing gap data is forbidden.
- **Outcome-harness.** Each buildable report has a byte-stable golden CSV; RPT-10's builder slot is marked, not faked.

## Logged by CEO (W4 device-build hotfix, 2026-05-25)

### D30 — 2026-05-25 — `Package.swift` declares an explicit Apple-platform floor
- **Decision.** Add `platforms: [.iOS(.v26), .macOS(.v14)]`; bump `swift-tools-version` 6.0→6.2.
- **Reversible?** Yes. **Blast radius?** Internal (toolchain).
- **Rationale.** With no `platforms:`, SwiftPM floored `LedgerCore` at iOS 12; a physical-device build then failed "Concurrency is only available in iOS 13.0.0 or newer." macOS 14 keeps host `swift test` concurrency-capable; **Linux ignores `platforms`, so W6 server-lift portability is unaffected.**
- **Outcome-harness.** `swift build` clean + 460 tests pass; the device build links.

## Logged by CEO (W2 preflight, 2026-05-25)

### D31 — 2026-05-25 — Real brokerage data lives git-ignored in `.flex-local/`; tokens never persisted to repo/ledger
- **Decision.** Real IBKR Flex statements + live portfolio data are PII, stored under `/Users/dev/git/ledger/.flex-local/` (gitignored with `*.flex.xml`, `.ibkr-secrets`), never committed. The Flex token + query id pass via env var / a config field — never written into ledger/decisions.md or source.
- **Reversible?** Internal but privacy-load-bearing. **Blast radius?** Internal (privacy contract).
- **Rationale.** Financial PII and secrets must never enter the repo or the company ledger.
- **Outcome-harness.** Repeatedly enforced: e.g. **D82/TICKET-16** scrubbed a token that leaked into a boss-confirmation doc; release notes are token-free; only the non-secret Query ID `0000000` remains. Any leak is treated as a guard failure and removed.

### D32 — 2026-05-25 — `FlexWebService` follows SendRequest → (poll) → GetStatement using the `Url` RETURNED by SendRequest
- **Decision.** Read `<Status>`/`<ReferenceCode>`/`<Url>` from the SendRequest response and call `{Url}?t={token}&q={referenceCode}&v=3`, polling with backoff on a non-Success/"in progress" status. The W1 `RawRecordSource` seam stays; this is a new conforming `FlexWebServiceRawRecordSource`.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Observed live: SendRequest hit `ndcdyn…` but the response `<Url>` pointed at `gdcdyn…` (IBKR data-center failover) — a hardcoded host would break.
- **Outcome-harness.** Preflight curl PASS against the live endpoint; the networking sits behind a fakeable transport (D44).

### D33 — 2026-05-25 — Add an AU COMPANY entity tax profile (0% CGT discount); map account U0000000 → Acme Holdings Pty Ltd
- **Decision.** Extend the per-entity discount config with a Company classification at discountRate 0 (AU companies get no 50% discount); map the real account to one Company entity. Full company income-tax-rate modelling is scoped by the architect; the minimum bar: CGT discount MUST be 0%.
- **Reversible?** Yes. **Blast radius?** Internal but correctness-load-bearing; flagged in the W2 boundary report.
- **Rationale.** Derivable from data + AU law, not a boss choice → decided + logged.
- **Outcome-harness.** Realized gains are never under-stated by a wrongly-applied discount; verified by the D39 negative control.

### D34 — 2026-05-25 — W2 token provisioning is dev/Simulator-grade (config field, plaintext); Keychain secure storage stays W4
- **Decision.** The token+queryId are entered in the Sync screen / a local untracked dev config; encrypted on-device storage (Keychain) stays W4 with a clear "secure storage = W4" marker.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** W2 runs on the Simulator pre-App-Store; no backend/OAuth yet.
- **Outcome-harness.** Hardened by **D48** (KeychainTokenStore) when the token starts persisting on-device.

## Logged by architect (W2 design, 2026-05-25) — ratified by CEO

### D35 — 2026-05-25 — W2 Company tax model: CGT discount 0% modelled; company flat income-tax-rate + franking-refund are Workings notes
- **Decision.** Model CGT discount 0% (D33's minimum bar); surface company flat-rate income + franking-refund as Workings notes, not a computed tax-payable (deferred to a later wave).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Don't fabricate a company income-tax-payable the spec never required pre-App-Store.
- **Outcome-harness.** Realized gains never under-stated; the Workings note is informational.

### D36 — 2026-05-25 — Real close marks + FX reference rates sourced from `<OpenPosition>` SUMMARY rows when `<ConversionRates>`/`<MTMPerformanceSummary>` are absent
- **Decision.** Dedicated sections win when present; `<OpenPosition>`-derived marks/FX are a pricing OVERLAY, never facts.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** The real statement has neither dedicated section (where W1 read marks/FX).
- **Outcome-harness.** R8 + the ledger/overlay split hold (marks never become facts).

### D37 — 2026-05-25 — The Flex date parser must accept BOTH dashed ISO `YYYY-MM-DD` and basic `YYYYMMDD`
- **Decision.** Parse both; fixed in W2-M11-T1 with a regression lock.
- **Reversible?** No, in spirit (correctness) — must hold. **Blast radius?** Internal but correctness-load-bearing.
- **Rationale.** Real Flex emits basic `YYYYMMDD`; the W1 parser returned an EMPTY date → would silently corrupt FY assignment, 12-month discount eligibility, and the fold order.
- **Outcome-harness.** Proven by running the W1 parser against `20260520`; the regression lock keeps both formats green.

## Logged by architect (W2 design, 2026-05-25) — A8 reconciliation

### D38 — 2026-05-25 — A8 reconciliation acceptance is proven with a fake `FlexTransport` fed the real on-disk statement bytes
- **Decision.** Reconcile against GENUINE `.flex-local/` data through a fake transport; the live network is exercised only by the preflight curl.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Keep the test deterministic/portable, avoid committing PII or depending on the network.
- **Outcome-harness.** **A8** (reconciliation to the cent) runs offline/sandbox-safe in CI against real bytes; the preflight curl already PASSed live.

## Logged by CEO (W2-M12-T2 real-data correctness, 2026-05-25)

### D39 — 2026-05-25 — A real `EntityDef.id` MUST equal the reconciler-stamped `entityId` (the IBKR account `name`), not a short code
- **Decision.** The Company entity's `EntityDef.id` is the verbatim `"Acme Holdings Pty Ltd"`; `"ACME"` is only the scope-switcher SHORT label. Resolution is by `EntityDef.id == fact.entityId` with an individual (0.5) fallback.
- **Reversible?** No — a binding real-data ingest contract (correctness). **Blast radius?** Internal but correctness-load-bearing.
- **Rationale.** The reconciler stamps `entityId` from the account `name`; a short id silently mis-resolves the discount.
- **Outcome-harness.** Negative control (reviewer + QA, independently): `id="ACME"` silently resolves to a WRONG 50% discount and drops the company Workings note — a D33 violation — so the test bites. (The `.all`-scope-on-a-single-entity 0.5-label fallback is cosmetic; revisit if a 2nd real entity is added.)

## Logged by CEO (W2-M14-T1b real-data correctness, 2026-05-25)

### D40 — 2026-05-25 — IBKR capital cash movements ("Deposits/Withdrawals") are NON-assessable; new `FactKind.depositWithdrawal`
- **Decision.** Map explicit IBKR capital types (Deposits/Withdrawals, Deposit, Withdrawal(s), Disbursement, Electronic Fund Transfer, Internal Transfer) to a new non-assessable `FactKind.depositWithdrawal` BEFORE the catch-all; income engines key only off `.dividend`/`.distribution`. The catch-all `→ .distribution` stays for genuine distributions.
- **Reversible?** No, in spirit (correctness) — must hold. **Blast radius?** Internal but correctness-load-bearing.
- **Rationale.** The catch-all taxed the 8 "Deposits/Withdrawals" (Σ $200,000) as grossed-up dividend income → a fabricated `eoyNetTaxableAUD = 200000`. AU tax law, not a judgment call → decided autonomously.
- **Outcome-harness.** W1 A2 oracle byte-identical (digest `33ad4db4…`, FactID `f7afeec2e435e0f6`, 49 facts unchanged); real `eoyNetTaxableAUD` 200000 → **0** (net capital loss $43.20 CF). NOTE: the unknown-cash-type tail is **TICKET-1** (boss-ratified: map an unrecognised type to a non-assessable + GAP-FLAGGED kind, W4/W5, W1-safe + additive).

## Resolved by boss (W3 quote provider, 2026-05-25)

### D41 — 2026-05-25 — W3 delayed quotes via Yahoo Finance, wrapped in a provider-agnostic abstraction (multi-provider-ready NOW)
- **Decision.** The boss chose Yahoo AND directed building a `QuoteProvider` abstraction up front (a deliberate override of the no-over-build default). Yahoo needs no API key → the W3 human-gate collapses and W3 opened immediately. Source = Yahoo's unofficial JSON endpoints called directly from Swift via the W1 `QuoteSource`/`PriceOverlay` seam; a licensed feed is a W6/App-Store decision.
- **Reversible?** Yes, via the seam. **Blast radius?** Strategic/boss-owned (provider choice + the build-the-abstraction-now override).
- **Rationale.** Free + pragmatic for personal use; the abstraction makes adding providers additive (ledger/tax untouched).
- **Outcome-harness.** Preflight PASS (2026-05-25): `v8/finance/chart/{symbol}` returns non-null delayed prices unauthenticated, US + ASX (AAPL 308.82 USD; WBC.AX 36.63 AUD).

## Logged by architect (W3 design, 2026-05-25) — ratified by CEO

### D42 — 2026-05-25 — W3 option marks use Yahoo `v8/finance/chart/{OCC-contract-symbol}` (same endpoint as equities), NOT the v7 cookie+crumb flow
- **Decision.** One endpoint / one code path / one fake-transport harness for shares, ASX, AND options; an option Yahoo can't price degrades to close-mark (then "quote pending"), never a crash or fabricated price.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Probed live: `v8/chart/BN260717P00042000` returns 200 unauthenticated (`instrumentType:OPTION`, `regularMarketPrice` 1.15); the v7 crumb flow is fragile.
- **Outcome-harness.** The live probe + the fake-transport offline tests; no v7 flow is built.

### D43 — 2026-05-25 — Delayed quotes are a SECOND overlay tier (`TieredPriceOverlay`) over the W1 `StatementCloseMarkOverlay`, set via the existing `store.setPriceOverlay` seam
- **Decision.** The engine/lot-ledgers/tax projections are UNCHANGED (R8 independence, re-locked by a byte-identical-tax test). The overlay handed into `ProjectionInput` is an immutable `QuoteSnapshot`-backed value (pure/sync `mark`); the async fetch + TTL cache live OUTSIDE `build` in the `QuoteService` actor (injected clock).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Keep A6 deterministic per snapshot; keep tax bytes stable.
- **Outcome-harness.** A W3 byte-identical-tax test re-locks R8; the injected clock makes CI deterministic offline.

### D44 — 2026-05-25 — `QuoteSymbol` is the provider-agnostic request key; `YahooSymbolMapper` does the Ledger→Yahoo translation
- **Decision.** `QuoteSymbol` = ledger symbol + kind + exchange hint; `YahooSymbolMapper` maps ASX `.AX`/OCC space-strip. Networking sits behind a fakeable `QuoteTransport` seam (mirrors D32).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Keep the `QuoteProvider` contract free of Yahoo conventions so a 2nd provider (D41) brings its own mapper.
- **Outcome-harness.** All W3 tests run offline; the live Yahoo path is proven by the preflight curl only.

## Logged by CEO (W3→W4 boundary, 2026-05-25)

### D45 — 2026-05-25 — Persona-skill lessons from W2+W3 codified into the persona skills (boss-ratified)
- **Decision.** Codify recurring high-value lessons into the persona skills via `superpowers:writing-skills`: mutation/negative-control discipline (a new test must be seen RED; the reviewer runs a mutation probe; never gitignore a compiled-in source) into `persona-backend-engineer` + `persona-code-reviewer`; design-time external-dependency probing + the additive-overlay pattern into `persona-tech-lead`; the paired-independent-oracle + re-plumb-every-consumer rules into `persona-qa`.
- **Reversible?** Yes. **Blast radius?** Internal (process/skills).
- **Rationale.** The gains were riding on persona judgment, unwritten.
- **Outcome-harness.** Evidence trail: W2-M11-T2 (vacuous test RED) → W3-M16-T1 (mutation-proven GREEN); D37/D40 + D42 (design-time probe); D40/A8 + R8 (equality+correctness). The **mutation-check / RED-first discipline** is the harness for every later fact-model change.

## Logged by architect (W4 design, 2026-05-25) — ratified by CEO

### D46 — 2026-05-25 — W4 on-device price alerts = LOCAL notifications (`UNUserNotificationCenter`), NOT remote APNs; server-pushed APNs deferred to W6
- **Decision.** A W4 background refresh evaluates the `AlertFiringCoordinator` and a `LocalAlertNotifier` posts a local notification per new `FiredAlert`; W6 adds a server APNs sender on the SAME `FiredAlert` seam (additive).
- **Reversible?** Yes (W6 extends the seam). **Blast radius?** Internal.
- **Rationale.** Remote push needs a sender and there is NO backend until W6; none of the 3 installed profiles carries `aps-environment` or targets `com.solo.ledger`.
- **Outcome-harness.** Full on-device daily alerting with no backend / no `aps-environment` (re-confirmed token-free + no-`aps-environment` at the D85-SHIP release).

### D47 — 2026-05-25 — Launch data source decided by PROVISIONED on-device state, retiring env-only `LedgerSource.current()` (the TICKET-2 fix)
- **Decision.** Source = `.realStatement` iff a UserDefaults intent==real AND a Keychain token is present ⇒ real pull on a normal launch; else demo. `LEDGER_SOURCE` survives only as a dev/test override (checked first). A missing/locked token degrades to demo + a reconnect prompt.
- **Reversible?** Yes. **Blast radius?** Internal. (The demo-on-degrade branch is later removed by **D85**.)
- **Rationale.** Fixes TICKET-2 (a normal device launch had no env var → W1 dummy fixture).
- **Outcome-harness.** Existing launches/screenshots stay byte-identical under the dev override; never crash, never label stale dummy as real.

### D48 — 2026-05-25 — Flex credential stored in the iOS Keychain via a `TokenStore` protocol seam (`KeychainTokenStore`, WhenUnlockedThisDeviceOnly), APP-layer only
- **Decision.** `KeychainTokenStore` hardens D34 and keeps D31 true now the token persists; `LedgerCore` stays Foundation-only (Keychain never enters the core); the protocol is the exact W6 swap-point for a server-side per-user store. `FlexDevConfig` is demoted to a dev/CI override (kept compiled-in — never gitignore a source). Token wiped on disconnect.
- **Reversible?** Yes (seam). **Blast radius?** Internal but privacy-load-bearing.
- **Rationale.** Persisting on-device requires real secure storage without breaking the portability or no-secret-in-repo locks.
- **Outcome-harness.** Token scoped device-only, never logged/committed (D31); the seam swaps for a server store in W6.

## Logged by CEO (W4 field-bug correctives, 2026-05-26)

### D49 — 2026-05-26 — Demo and real data live in SEPARATE on-disk FactStore directories (per-source isolation)
- **Decision.** `AppContainer.makeStore`/`appSupportDir` namespace the FactStore dir by `LedgerSource` (`…/Ledger/demo` vs `…/Ledger/real`) via one `appendingPathComponent`; `LedgerCore` untouched (append-only `FactStore`, no delete API). Implemented in W4-M21-T1 (`LedgerSource.storeSubdirectory`).
- **Reversible?** Yes. **Blast radius?** Internal; the smallest blast radius for TICKET-3.
- **Rationale.** Demo W1 facts and real Acme facts must never share one append-only log (TICKET-3: a stranded, unremovable demo account in `scope==.all`).
- **Outcome-harness.** A shared-dir control reproduces the TICKET-3 mixing → RED (mutation-proven); the per-dir test hands the same temp root to a demo + a real `makeStore` and proves isolation. App 253 / core 595 green. **Migration:** the old shared `…/Ledger` root is simply orphaned (real re-pullable from the token, demo re-ingestible) — no migration code, no data loss. This per-source partition generalizes to per-account under per-connection in **D83**.

### D50 — 2026-05-26 — The listing exchange is a first-class `PositionsProjection.Row` field, threaded into `QuoteSymbol.exchangeHint`; the W1-fixture `InstrumentDirectory` table is NOT used for held-row exchange
- **Decision.** `Row` gains `exchange` (from the recovered `Instrument.exchange`); `AppModel.quoteSymbols()` reads `row.exchange` → `exchangeHint` (ASX → `.AX`), removing the W1-ten fallback; `FlexXMLParser` also reads `<OpenPosition listingExchange>`.
- **Reversible?** Yes; R8 unaffected (exchange is quote-routing metadata, not a tax input). **Blast radius?** Internal.
- **Rationale.** TICKET-4: real symbols weren't in the W1 table → the bare ticker matched the wrong exchange's same-ticker (ASX BTI showed the NYSE BTI price).
- **Outcome-harness.** Held rows route to the correct exchange; the mis-priced ASX symbols resolve correctly.

## Resolved by boss (2026-05-26) — strategic pivot (UI freeze)

### D51 — 2026-05-26 — W5/App-Store DELAYED INDEFINITELY; FEATURE FREEZE; UI review-and-harden before any further features
- **Decision.** Hire three independent UI-reviewer personas (lenses: Usability/UX-flows · Visual-design/consistency · UI-bugginess/state-correctness), each reviewing independently (no shared findings); resolve what they surface BEFORE new features or resuming the App-Store wave. Created `persona-ui-reviewer`. (M22 connect-flow correctives are held under the freeze.)
- **Reversible?** Strategic/boss-owned; in effect until the boss lifts it. **Blast radius?** Strategic (direction + hiring).
- **Rationale.** Poor UI surfaced on-device (connect/demo confusion + TICKET-3/4/5/6).
- **Outcome-harness.** The freeze lifts only when the UI-hardening exit gate passes (see D52/D59).

### D52 — 2026-05-26 — Run the UI hardening AUTONOMOUSLY to the human-test boundary
- **Decision.** Fix `ui-review/UI-BACKLOG.md` (blockers→majors→minors) via the normal persona chains, UI fixes ONLY (new features + the App-Store wave stay frozen, D51); the autonomous EXIT GATE = re-run the documentation-driven test (`persona-ui-reviewer` Mode-A writes the UI manual + `persona-ui-critique` reads it). Then cut a release, install on `Tg`, hand back.
- **Reversible?** Process/boss-owned. **Blast radius?** Process.
- **Rationale.** Boss: "run autonomously as much as you can before I can start human test again."
- **Outcome-harness.** Passes when a non-technical critique can complete the core tasks (connect / tell real-vs-demo / remove demo) from the manual alone.

## Logged by architect (UI-hardening design, 2026-05-26) — ratified by CEO

### D53 — 2026-05-26 — "Clear sample data" = an app-layer wipe of the demo FactStore dir
- **Decision.** Wipe `…/Ledger/demo/facts.jsonl` (app-owned); no `LedgerCore` delete API; the demo dir re-ingests the bundled fixture on next demo launch (idempotent).
- **Reversible?** Yes. **Blast radius?** Internal. (Retired entirely by **D85** — nothing to clear once demo is gone.)
- **Rationale.** Gives a token-less user a way to remove the demo (UI-B3).
- **Outcome-harness.** The demo dir empties; a demo relaunch re-seeds idempotently.

### D54 — 2026-05-26 — Loss-harvest candidates carry the real position market value
- **Decision.** Surface the projection's market value instead of the hardcoded `0` at `TaxViewModel.swift:500` (UI-M4).
- **Reversible?** Yes; display-only, R8 unaffected. **Blast radius?** Internal.
- **Rationale.** The candidate list showed `0` MV.
- **Outcome-harness.** The MV the projection already holds renders correctly.

### D55 — 2026-05-26 — Flex-token validation COPY corrected to the real rule (token≥8 / queryId≥5); the gate is NOT tightened
- **Decision.** Drop the "16-digit/6-digit" placeholders + validate inline (UI-B1); do not tighten (tightening risks rejecting valid real tokens).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** The placeholder copy mis-described the rule.
- **Outcome-harness.** Valid real tokens pass; the copy matches the real rule.

### D56 — 2026-05-26 — Dynamic Type via `Font.custom(…, relativeTo:)` replacing `fixedSize:`, with an accessibility-size CEILING on dense financial tables
- **Decision.** Numbers scale without truncating columns (UI-M6).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** `fixedSize:` ignored Dynamic Type.
- **Outcome-harness.** Tables scale up to the ceiling without column truncation.

### D57 — 2026-05-26 — Alert status language reflects real on-device behavior (`ARMED`/`FIRED`), dropping "W3/SCAFFOLD/IDLE/WOULD-FIRE/later"
- **Decision.** Now that W4 ships local-notification firing, use `ARMED`/`FIRED` (UI-B5).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** The scaffold framing no longer matches behavior.
- **Outcome-harness.** The status copy matches the shipped firing path (D46).

### D58 — 2026-05-26 — Gates that run MUTATION PROBES on the SAME source file must run SEQUENTIALLY, not concurrently
- **Decision.** Carve an exception out of D11: when a subtask's gates will mutation-probe the same file/region (the D45 discipline has each gate TEMPORARILY edit production source to prove a test bites, then restore), dispatch them sequentially (or give each its own copy/worktree). The default stays concurrent (immutable artifact, read-only gates).
- **Reversible?** Process; ratified. **Blast radius?** Process.
- **Rationale.** UIH-M26 produced a reviewer FALSE-NEGATIVE FAIL because the concurrent QA agent's `Theme.swift dim` probe reverted the value mid-run.
- **Outcome-harness.** The D11 harness ("a serial re-run must not flip the verdict") is exactly what fired: the CEO re-ran serially → 14/14 green, fix genuinely applied. (HR: fold into `delivering-a-subtask` / D11.)

## Resolved by boss (2026-05-26) — feature track resumes; Daily-Performance & Cash wave

### D59 — 2026-05-26 — Freeze (D51) LIFTED for feature work; "Daily Performance & Cash" wave opens NOW, before the still-frozen App-Store wave
- **Decision.** Queue three improvements: (1) track CASH (AUD home + FX balances); (2) PER-POSITION daily P/L; (3) daily P/L baselined to the price SNAPSHOT at 12am Australian time. Runs autonomously to a device install + boss re-test.
- **Reversible?** Strategic/boss-owned. **Blast radius?** Strategic.
- **Rationale.** The UI-hardening exit gate passed (0.5.0(6) on `Tg`); no new capital/credentials needed.
- **Outcome-harness.** End-to-end DPC acceptance on real data; device install + boss re-test (released 0.6.0(7), D69).

### D60 — 2026-05-26 — Daily baseline = 12am Australia/Sydney, DST-aware (AEST/AEDT)
- **Decision.** Midnight follows daylight saving (UTC+11 summer / UTC+10 winter), matching the ASX wall-clock — NOT fixed UTC+10.
- **Reversible?** Strategic/boss-owned. **Blast radius?** Strategic (the day boundary definition).
- **Rationale.** Boss's answer; the day boundary for "daily P/L" rolls at Sydney local midnight.
- **Outcome-harness.** Realized via D66 (DST handled by `Calendar`/`TimeZone`; probe confirmed `gmtoffset` rolls 36000→39600).

### D61 — 2026-05-26 — Daily P/L uses a FROZEN-FX decomposition: securities show the PRICE move only; FX movement is attributed to cash
- **Decision.** Per-security daily P/L (AUD) = (currentPrice − snapshotPrice) × qty × **snapshot FX (frozen at midnight)**; FX-cash daily P/L (AUD) = balance × (currentFX − snapshotFX); home AUD cash is flat. Whether to ALSO surface the aggregate securities-FX-revaluation line is left to the architect (→ D68).
- **Reversible?** Strategic/boss-owned (decomposition); mechanism = architect. **Blast radius?** Strategic + internal mechanism.
- **Rationale.** Boss: "Even positions should be comparing to midnight snapshot with frozen fx" — isolate the instrument's price move from intraday FX drift.
- **Outcome-harness.** R8 holds (snapshot prices/FX are overlay, tax never moves); realized as the D67 display metric.

### D62 — 2026-05-26 — The 12am snapshot is RECONSTRUCTED, not captured live
- **Decision.** iOS can't reliably wake at Sydney midnight, so the baseline price/FX is read from the quote provider's intraday/historical series at the 00:00-Sydney instant (last trade at-or-before) and PERSISTED as a per-day snapshot.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Fits the D41 `QuoteProvider` abstraction; live-observed fallback if history is unavailable.
- **Outcome-harness.** Confirmed reconstructable by the design-time probe (D64 preamble); finalized as the D65 `DailySnapshotProvider`/`DailySnapshotStore`.

### D63 — 2026-05-26 — UX-VALIDATION-FIRST on the feature concept (boss directive)
- **Decision.** The wave's FIRST milestone is a design + usability-validation leg: `persona-designer` designs the cash + per-position daily-P/L + frozen-FX/FX-cash presentation, then `persona-ui-reviewer` + `persona-ui-critique` validate a layperson can use it; issues fold back into requirements BEFORE engineering builds.
- **Reversible?** Process/boss-owned. **Blast radius?** Process.
- **Rationale.** Boss: "Make sure you pass the idea to ux designers to validate usability"; don't ship another un-usable surface.
- **Outcome-harness.** It earns its keep — the leg caught a pre-code BLOCKER (cited by D77 as the reason to reuse the gate for FXC's new surfaces).

## Logged by architect (Daily Performance & Cash design, 2026-05-26) — ratified by CEO

### D64 — 2026-05-26 — Cash balance is a DERIVED PROJECTION over existing facts (`CashProjection`, Foundation-only core), per (entity, account, currency); there is NO `<CashReport>` to read
- **Decision.** Per-currency cash = Σ(depositWithdrawal nativeAmount) + Σ(trade cash leg) + Σ(dividend/distribution/interest/withholding net), grouped by currency; balances are SIGNED. A pure fold (A6-rebuildable), no core delete/mutation API. Requires the reconciler to carry each cash-bearing fact's signed cash amount + currency (the trade `netCash` is otherwise dropped at the fact boundary — see D70).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** The real statement carries no ending-cash section; cash MUST be derived. The real account holds a NEGATIVE USD margin balance, so balances must be signed.
- **Outcome-harness.** A6-rebuildable, version-stamped; the cash oracle (later corrected by TICKET-8/D84 to **AUD +25,616.73 / USD +1,269.53**) reconciles to the legs. Cash is NOT a lot-ledger entity (no parcels).

### D65 — 2026-05-26 — The 12am-Sydney snapshot is reconstructed via an ADDITIVE app/provider-layer `DailySnapshotProvider` + persisted `DailySnapshotStore`, NOT by overloading the live `QuoteProvider`
- **Decision.** A separate seam (reusing `QuoteTransport` + `YahooSymbolMapper`) takes symbols + an explicit Sydney-midnight `Date` and returns the last price at-or-before per symbol + the snapshot FX per currency; it runs OUTSIDE `Projection.build` (like D43) and PERSISTS each per-day snapshot to disk (app layer).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** The live `QuoteProvider`/`SeriesSpec` only support RELATIVE ranges; reconstruction needs Yahoo's `period1`/`period2` absolute window.
- **Outcome-harness.** Probe 1c returns 5m bars in an arbitrary past window; the snapshot stays stable through the day; `LedgerCore` stays Foundation-only (overlay/display, never a fact, R8).

### D66 — 2026-05-26 — Sydney-midnight day boundary is computed in the APP layer from a DST-aware `TimeZone("Australia/Sydney")` and passed as an EXPLICIT `Date` input into the deterministic daily-P/L computation
- **Decision.** The app resolves "the most recent 00:00 Australia/Sydney instant" (AEDT/AEST via `Calendar`/`TimeZone`) and threads it in as data — the snapshot key AND the as-of anchor the UI labels.
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** The core never reads `Date()`/`TimeZone`/locale inside a fold (D8).
- **Outcome-harness.** Pure + testable with an injected zone/clock; the probe confirmed the offset rolls 36000→39600 across DST.

### D67 — 2026-05-26 — Daily P/L (frozen-FX decomposition, D61) is computed as a DISPLAY METRIC in a thin `DailyPLCalculator` over projection OUTPUTS; it does NOT enter the fact log or any tax projection (R8 RE-LOCKED)
- **Decision.** The calculator lives in the core as a PURE function over (positions output, cash output, snapshot, currentOverlay/currentFX) — unit-testable + portable — but is a SEPARATE read-model assembled OUTSIDE the tax pipeline; it consumes outputs, never feeds the lot ledgers / CGT / EOY inputs.
- **Reversible?** No, in spirit (correctness) — tax numbers never move. **Blast radius?** Internal but correctness-load-bearing.
- **Rationale.** Keep daily P/L portable and testable without ever perturbing a tax cell.
- **Outcome-harness.** An R8-isolation test asserts the full `TaxProjection` (EOY + CGT) is BYTE-IDENTICAL with daily-P/L present vs absent and across two different snapshots (non-vacuous, mutation-proven). This is the byte-identical-tax / no-tax-leak property — **A18** (`display-never-moves-tax`).

### D68 — 2026-05-26 — ALSO surface the aggregate securities-FX-revaluation line (the unattributed Q·P0·ΔR term) as an ADDITIVE portfolio-level reconciliation line
- **Decision.** Surface "currency revaluation (securities)" as one aggregate line so the daily-P/L breakdown RECONCILES to the total AUD change; display-only, additive, never a tax input. The designer (M1) validates whether a layperson needs it shown by default / in a detail / as a toggle.
- **Reversible?** Yes. **Blast radius?** Internal (the boss left this to the architect, D61).
- **Rationale.** Without it, "sum of per-position daily P/L + FX-cash daily P/L ≠ change in total portfolio AUD value" would confuse the user.
- **Outcome-harness.** The breakdown reconciles to the total AUD change; R8 holds; the UX-validation leg owns the final presentation.

### D69 — 2026-05-26 — Release at the DPC end-to-end acceptance proposed as 0.6.0(7); installed on `Tg`
- **Decision.** A feature minor over 0.5.0(6); installed on `Tg` for the boss re-test; the App-Store wave stays FROZEN (D51).
- **Reversible?** Process. **Blast radius?** Process.
- **Rationale.** Natural next minor for the DPC feature wave.
- **Outcome-harness.** DPC end-to-end acceptance green; device install + boss re-test.

### D70 — 2026-05-26 — The reconciler carries the trade cash leg as an additive `attributes["netCash"]` (signed, commission-folded), NOT a new `Fact` field; the A2/A8 fact-digest fixtures re-baseline while TAX cells stay byte-identical
- **Decision.** Carry `netCash` only on `.trade` facts (a non-trade cash fact's cash leg already equals `nativeAmount`); lowest blast radius (no `Fact` struct change → no shape migration), matching the existing additive-attribute pattern (`premium`/`frankingCredit`). `CashProjection` reads it; every other consumer ignores it.
- **Reversible?** Yes. **Blast radius?** Internal (re-baselines the canonical fact bytes).
- **Rationale.** D64 left the mechanism open; this finishes it. A `.trade`'s `nativeAmount` is the trade VALUE with commission separate in `fee`, so its cash leg is `nativeAmount − feeOutflow`.
- **Outcome-harness.** Adding the attribute moves a trade fact's canonical bytes, so the W1 A2 oracle digest + the W2 A8 reconciliation fact-digest are re-derived — and the TAX cells in BOTH stay BYTE-IDENTICAL (mutation-proven that nothing tax reads `netCash`; this is the **A18** byte-identical-tax / no-tax-leak property). The value source is then corrected by D71/D84.

### D71 — 2026-05-26 — `netCash` VALUE SOURCE = IBKR's authoritative per-row `<Trade netCash>` (new `RawTrade.rawNetCash`), PREFERRED over D70's computed `nativeAmount − feeOutflow`, with the computed value as fallback
- **Decision.** The parser captures `<Trade netCash>`; `Reconciler.tradeFact` PREFERS it, falling back to the computed value when absent (the W1 dummy fixture has no `netCash` attr). **D70's mechanism is UNCHANGED** (additive, trade-only, R8-safe, fact-digest re-baselined, tax byte-identical) — ONLY the value source changed.
- **Reversible?** Yes. **Blast radius?** Internal. **NOTE:** later **invalidated for forex-conversion legs** by TICKET-8/D78 (raw `<Trade netCash>` is `0` on AUD↔USD conversion legs); D71 remains correct for ordinary trade legs.
- **Rationale.** D70's computed formula yielded USD +1,255.17, not the correct −47,376.22, because 13 forex `assetCategory="CASH"` legs mis-bucket the AUD notional and sum an AUD commission into a USD `Decimal`.
- **Outcome-harness.** IBKR's `<Trade netCash>` reproduces the A16 oracle to the cent (AUD +93,627.88 / USD −47,376.22 *at the time*); reviewer re-summed 60 trade rows by hand; QA's two-currency oracle + the R8-leak mutations confirm.

## Resolved by boss (2026-05-26) — Foreign-Currency Correctness (FXC) corrective wave

### D72 — 2026-05-26 — FXC corrective wave opens NOW, bundling TICKET-8 + TICKET-9 + TICKET-10, one release, before the frozen App-Store wave
- **Decision.** Cash forex-leg fix (TICKET-8) + Sync-now control (TICKET-9) + Div 775 forex realisation (TICKET-10). Runs autonomously to a device install + boss re-test.
- **Reversible?** Strategic/boss-owned. **Blast radius?** Strategic.
- **Rationale.** Boss: "bundle all three."
- **Outcome-harness.** FXC end-to-end acceptance on real data (released 0.7.0(8), D78); boss re-test.

### D73 — 2026-05-26 — The Div 775 forex election is a USER-FACING toggle in Config that explains Div 775 when toggling
- **Decision.** The tax engine reads the election state as a config input. DEFAULT = standard Div 775 (no election; FRE-1 gains assessable / losses deductible on revenue account); the toggle applies the election (e.g. the $250k balance election) and shows explanatory copy. The election is the boss's tax position — the app exposes it rather than assuming it.
- **Reversible?** Strategic/boss-owned. **Blast radius?** Strategic + internal mechanism. **NOTE:** D73's "account-level" WORDING is superseded by **D83** — the election is PER-TAXPAYER (keyed by `entityId`), not per-account; the wording is a queued copy fix.
- **Rationale.** Boss directive: "add a toggle, account-level setting, explains Div 775 when toggling."
- **Outcome-harness.** The election state flows as a config input into the Div 775 engine; the explainer goes through D77 UX validation.

### D74 — 2026-05-26 — TICKET-8 (cash) and TICKET-10 (Div 775) share ONE forex-conversion fact model
- **Decision.** Model each AUD↔USD conversion ONCE in the facts/reconciler (native foreign amount, AUD-settled amount, the implied native→AUD rate — sourced from the leg, never fabricated; remove the phantom-equity mishandling), serving BOTH the cash projection (R8/display) and a new Div 775 forex-realisation engine (tax).
- **Reversible?** Yes. **Blast radius?** Internal. **Resolved by D78:** a new `FactKind.forexConversion` (NOT `InstrumentKind.currency`) + a per-currency cash cost-base ledger.
- **Rationale.** Single source of truth; avoids a second ad-hoc parse of the CASH rows.
- **Outcome-harness.** Both consumers read the same fact; the phantom-equity path is removed (TICKET-10).

### D75 — 2026-05-26 — FXC acceptance oracles MUST be INDEPENDENT GROUND TRUTH, not same-method sums
- **Decision.** Cash oracle = the first-principles re-derivation (AUD +25,616.73 / USD +1,269.53 over the `.flex-local` bytes) + the boss's on-device LIVE confirmation post-sync. Div 775 oracle = a hand-computed FRE-1 per-currency cost-base ledger over the 13 legs. PLUS a proof the SHARE CGT numbers are UNCHANGED and no double-count under the 12-month rule. Every new test RED-first + mutation-checked (D45).
- **Reversible?** Process/binding. **Blast radius?** Process (correctness discipline).
- **Rationale.** The cash bug slipped the DPC gates because the A16 "oracle" was the architect's own deposits+netCash sum → the gates locked internal consistency, not reality.
- **Outcome-harness.** The independent re-derivation + the boss's live confirmation are the harness; this is the paired-independent-oracle rule (D45) made binding for FXC.

### D76 — 2026-05-26 — AUD functional currency confirmed; NOT Subdiv 960-D USD functional currency — the standard translation model applies
- **Decision.** Account base = AUD per the statement; the architect picks a defensible cost-base method for fungible foreign currency (configurable/recorded).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** Subdiv 960-D (USD functional currency) does not apply.
- **Outcome-harness.** The chosen method is recorded on the engine result (D78b finalizes the default = weighted-average).

### D77 — 2026-05-26 — The FXC wave's NEW user-facing surfaces go through the UX-validation-first gate (D63)
- **Decision.** The NEW surfaces — the Div 775 forex-gain tax line, the election toggle + its explainer, and the "Sync now" control — get a designer → ui-reviewer → non-technical critique pass before/as they're built. (The corrected cash display was already layperson-validated in DPC-M1.)
- **Reversible?** Process. **Blast radius?** Process.
- **Rationale.** Div 775 is hard to explain to a layperson; the toggle's explanatory copy especially needs the critique.
- **Outcome-harness.** The same documentation-driven critique that caught a pre-code blocker in DPC (D63) is reused; the new forex tab is validated before ship (D84 surface).

## Logged by architect (FXC design, 2026-05-26)

### D78 — 2026-05-26 — FXC architecture: `FactKind.forexConversion` (NOT `InstrumentKind.currency`); per-currency cash cost-base method default = weighted-average; release 0.7.0(8)
- **Decision.** (a) The forex-conversion fact is a new `FactKind.forexConversion` (routes to NO lot-ledger mutation: `FactInterpreter.forexConversion → []`, `LotLedger.key → nil`) carrying both signed legs + the implied native→AUD rate as additive canonical-`Decimal` attributes, read by BOTH `CashProjection` and the `Div775Engine` (the D74 single source of truth). (b) `ProjectionConfig.forexCostBaseMethod` default `.weightedAverage` (FIFO is the recorded alternative; the chosen method is recorded on the result); configurable (D76), invalidates via `dropAll()` (D22). (c) Release 0.7.0(8) on `Tg`; App-Store wave stays FROZEN (D51).
- **Reversible?** Yes. **Blast radius?** Internal.
- **Rationale.** An `InstrumentKind.currency` would invite a phantom currency "position" — the very mishandling TICKET-10 removes; a distinct `FactKind` (like `.depositWithdrawal`, D40) routes cleanly to no parcel. Fungible currency has no natural parcel identity → a running weighted-average is the simplest defensible FRE-1 model.
- **Outcome-harness.** The exhaustive no-`default:` `FactKind` switch forces every consumer to handle it; the cash + Div 775 oracles (D75) are computed off the one fact model.

### D79 — 2026-05-26 — Div 775 v1 SCOPE = "conversions-only" FRE-1; USD-settled SHARE/income trade legs EXCLUDED (REVERSED by D84)
- **Decision.** Feed the Div 775 cost-base ledger from ONLY the pure `.forexConversion` legs, excluding the 8 USD `.trade` legs, for FY26.
- **Reversible?** Yes — **REVERSED by D84** (boss directive: recognise forex realisation on ALL USD currency disposals, including every USD-stock PURCHASE). **Blast radius?** Boss-facing/strategic (escalated for ratification at the FXC boundary).
- **Rationale (at the time).** Folding the USD `.trade` legs in would DOUBLE-COUNT the FX already embedded in each share's per-event CGT; sound *specifically* because FY26 had no realised USD-settled share SALE and no foreign-cash income.
- **Outcome-harness.** Excluding the 8 USD `.trade` legs gave +1,291.30; folding them in gave −13,474.37 → the share-CGT-byte-identical + no-double-count tests went RED. The dormant `forexCurrencyMovements` seam carried the future case — which D84 then activated.

## Resolved by boss (2026-05-27) — on-device stale-store bug

### D80 — 2026-05-27 — Fact-store carries a RECONCILER/FACT-MODEL VERSION stamp; an out-of-date stamp triggers a CLEAN store WIPE + full re-pull/re-reconcile (NOT an append)
- **Decision.** A `modelVersion` marker persisted alongside each source's `facts.jsonl`; on `load()`/`resyncNow` for the REAL source, if the stamped version < the current reconciler-model version → `emptyFactLog(source:.realStatement)` + a fresh `ingestReal` + stamp. The demo source is wiped + re-ingested from the bundled fixture on the same check (idempotent). Bump the model version for the FXC fact model. App-layer/provisioning (like D49); `LedgerCore` untouched.
- **Reversible?** Yes. **Blast radius?** Internal; SAFE per D49 (real data is always re-pullable, no loss).
- **Rationale.** TICKET-11: the device replayed the OLD 0.6.0(7) fact log (phantom `.trade` AUD.USD facts, zero `.forexConversion`) so the v2 `CashProjection` found none and still computed +93,627.88; a plain "Sync now" APPENDS (resurrecting the phantom) — only a wipe+re-reconcile fixes it.
- **Outcome-harness.** On first 0.7.1(9) launch `Tg` auto-wipes the 0.6.0(7) phantom log + re-pulls a clean store → cash corrects to +25,616.73 AND no phantom. (Manual workaround pre-0.7.1: disconnect+reconnect, NOT a plain Sync now.) Generalized to per-account migration in D83.

## Resolved by boss (2026-05-27) — three more on-device bugs (the 0.7.1(9) patch)

### D81 — 2026-05-27 — Daily-P/L overlay precedence fix: "current" mark = `positions.priceSource ?? closeOverlay`, never the historical statement close (TICKET-12)
- **Decision.** Invert the precedence at `AppContainer.swift:1399` (`refreshDailyPL`) so "current" is the freshest live/delayed mark; a symbol with no live mark → honest empty state, not the stale close. Add a DIRECTIONAL integration test.
- **Reversible?** Yes. **Blast radius?** Internal (app wiring).
- **Rationale.** It passed `closeOverlay ?? positions.priceSource`, using the OLD statement close as "current" while the snapshot was TODAY's price → `(old − new)` inverted the sign for every LONG.
- **Outcome-harness.** A directional test (live price > snapshot → positive/green TODAY) is RED under the old precedence; the DPC unit tests missed it because they tested the calculator in isolation, never the AppContainer overlay selection.

### D82 — 2026-05-27 — Lift the share-sheet presentation to the single app-level presenter; `ExportButton` emits result URLs upward (TICKET-13)
- **Decision.** `ExportButton` emits its result URLs via callback → `AppModel.autoShareURLs` (the single presenter at `RootView.swift:74`) instead of owning its own `.sheet`; one canonical share site removes the SYNC-tab collision. Add a presentation test.
- **Reversible?** Yes. **Blast radius?** Internal (app presentation).
- **Rationale.** The SYNC tab already carried the FlexQuerySheet + the GAPS sheet; iOS allows one active sheet per context, so `ExportButton`'s own `.sheet` was suppressed (pure no-op).
- **Outcome-harness.** A presentation test (tap EXPORT on SYNC → the share presenter fires); the export engine itself was always correct/tested. (Also note: TICKET-16 later scrubbed a literal token that had leaked into `0.8.0-boss-confirmation.md` — D31 enforced.)

## Resolved by boss (2026-05-27) — Account/Entity/Connection model (RATIFIED; build DEFERRED)

### D83 — 2026-05-27 — The Account/TaxEntity/Connection domain model is RATIFIED; BUILD DEFERRED until a real 2nd account/entity/connection exists ("bank the design, no code")
- **Decision.** Ratify (full design `account-entity-connection-design.md`): **Account is the SPINE** (accountId + entityId already on every fact); **Connection 1→N Account**, **Account N→1 TaxEntity** (Connection–TaxEntity is N:M through Account). **TaxEntity OWNS the tax profile + the Div 775 election + option treatment PER-TAXPAYER** (keyed by `entityId` — **supersedes D73's "account-level" phrasing**). Connection/Account owns the credential/ingest/FactStore/re-pull/migration; the D49 per-source partition generalizes to per-account under per-connection via a D80 model-version migration. Scope switcher filters by TaxEntity. Account→entity assignment is three-tier with a **SAFE-FLAGGED fallback (R8/D45 hardening of D39): an unrecognized real account NEVER silently gets the 0.5 Individual discount — conservative 0% until assigned.** Nothing is built (N=1).
- **Reversible?** Yes (design banked; nothing in production). **Blast radius?** Strategic/boss-owned; the boss explicitly accepted the latent N>1 risk (a 2nd account today → self-entity fallback → SILENT 50% discount, the D39/R8 hazard).
- **Rationale.** N=1 today (one connection → one Acme account → one Company entity), so nothing is broken; the real account is an `Institution Master`, so the multi-account door is ajar.
- **Outcome-harness.** The design doc is the durable spec; QUEUED (do FIRST when N>1): (1) the R8 safe-flagged fallback (never silent-50%); (2) the per-entity election copy fix (D73); (3) model Connection as first-class; (4) per-account FactStore partition; (5) the persisted account→entity directory + assignment UI. R8 holds throughout.

## Resolved by boss (2026-05-27) — forex CGT scope reversal

### D84 — 2026-05-27 — REVERSES D79: Div 775 forex realisation is recognised on ALL USD currency disposals, INCLUDING every USD-stock PURCHASE (boss directive, ATO-grounded)
- **Decision.** The per-currency (USD) weighted-average cost-base ledger consumes ALL USD movements — the `.forexConversion` legs AND the USD `.trade` cash legs (`netCash × fxAtEvent`); each USD outflow (stock buy, conversion-out, USD fee) realises a FRE = AUD-value-at-disposal − AUD-cost-of-USD-disposed; per-purchase events are SHOWN on the Tax→Forex tab (a NEW surface → D77 UX validation). Re-baseline the assessable line +1,291.30 → **−1.76 AUD**.
- **CRITICAL build instruction — the net-short fix (load-bearing):** ship `forexCurrencyMovements` including USD `.trade` legs **WITH** the `WeightedAveragePool` same-day INFLOWS-before-OUTFLOWS ordering (tie-break stableSourceID). A naïve reversal (strict (eventDate, stableSourceID) fold) ships **−13,474.37 = a FABRICATED ARTIFACT** because the 2026-05-19 same-day 150k-AUD round-trip is ordered DISPOSAL-before-FUNDING → the USD pool goes economically NEGATIVE → corrupts the running average.
- **Reversible?** Strategic/boss-owned. **Blast radius?** Strategic (moves a tax number; Company entity → no CGT discount on the FRE).
- **Rationale.** The boss (taxpayer) directed — repeatedly — that buying a USD stock disposes USD → a forex realisation (FRE-2). D79's exclusion was wrong. No double-count this window (ZERO share SELLs — all buys; the 2 SELLs are option WRITES), so share CGT stays byte-identical at $0.
- **Outcome-harness.** Per-event FRE list is the D75 oracle (SGOV 0/0, 150k round-trip +33.20, MSFT −25.40, BN −0.05/−4.72/−4.79; pool ends +1,269.53 USD = the cash oracle). **CORRECTED TOTAL = −1.76 AUD** (cross-check: all-inflows-first gives +8.49, also ≈0; only economically-impossible orderings produce the large artifacts). A20 (share CGT byte-identical) + A20 (no double-count) stay green. 12-month-rule boundary noted for a future FY that SELLS a USD share.

## Resolved by boss (2026-05-28) — retire shipping sample data

### D85 — 2026-05-28 — The SHIPPING app NEVER shows sample/demo data; `flex_dummy.xml` becomes a TEST-ONLY artifact
- **Decision.** Not-connected/no-token/degraded → a clean CONNECT IBKR / honest-empty state, NEVER demo holdings (close the two production demo-ingest leaks: `LedgerSource.resolve` step 3, and `AppModel.load()`'s degrade branch calling `ingestW1`; production `load()`/`disconnectReal` must NEVER call `ingestW1`). Fresh install → straight to CONNECT IBKR (drop the "USE DEMO CREDENTIALS" button + bundled-sample import; KEEP real user-file import). Retire the demo chrome (SAMPLE DATA chip, Tax SAMPLE banner, "clear sample data" / D53). `.bundledFixture`/`ingestW1` stay compiled-in but reachable ONLY via `LEDGER_SOURCE=demo` (dev/CI screenshots). Device-on-demo migration is FREE (the demo store is simply no longer read; reuses D49/D80).
- **Reversible?** Yes. **Blast radius?** Strategic/boss-owned (product direction); app-layer only, `LedgerCore` byte-UNTOUCHED, R8/tax untouched. **Supersedes** the demo-default of D9/D47 + the demo-fallback; **retires** D53/M23 chrome.
- **Rationale.** Boss: "Why do we have sample data still? We've moved way beyond that." Demo caused the "still 0 CGT" + "demo numbers" confusion (the boss landing on the demo source).
- **Outcome-harness (D85-SHIP, released 0.10.0(15)).** App tests asserting the OLD demo-fallback are REWRITTEN to the new contract (the rewrite IS the mutation-proof); `D85-IMPORTFILE-FIX` is mutation-proven (reverting to `ingestW1` leaks NVDA/AAPL/TSLA/MSFT/VAS → RED). 3-round reviewer gate (FAIL on the demo banner, FAIL on the disconnect button, PASS after an independent grep confirmed no rendered "demo" survives) → independent QA (both suites green; the `LEDGER_SOURCE=demo` dev override still loads the fixture; D84 −$1.76 still on the MAIN Tax summary). **694 core / 480 app green, token-free (D31), no `aps-environment` (D46).** LESSON: a "no demo" invariant must be enforced by copy-truth guards on EVERY user-facing surface, not only the data path.

---

## Logged by architect (W6 backend-lift design)

### D-seam — FactStore-protocol decision — extract a `FactStore` protocol seam so LedgerCore lifts verbatim into AWS Lambda over a multi-tenant DynamoDB fact store
- **Decision.** Define a `FactStore` protocol (the append/read/digest contract over the canonical JSONL fact bytes of D5) as the storage seam. The on-device build keeps the file-backed implementation; the W6 backend supplies a DynamoDB-backed implementation (per-tenant partition) behind the **identical** protocol. LedgerCore — the tax/projection math — is moved **verbatim** into Lambda; only storage, secrets, schedule, notify, and transport change. "A lift, not a rebuild." Path A (single-tenant, unattended) ships before Path B (multi-tenant SaaS).
- **Reversible?** Yes — the protocol is the reversal point (a different backend, or a return to file-backed, is another conforming implementation). **Blast radius?** **Locks future waves** — the *direction* (lift-not-rebuild; Path A before Path B; multi-tenant DynamoDB) is in the roadmap the founder owns; the *seam mechanics* are decided autonomously here.
- **Rationale.** The product was built Foundation-only (D4) precisely to make this lift possible; re-implementing the tax engine server-side would re-open every oracle-verified number to regression.
- **Outcome-harness.** **Byte-identical replay across hosts** (`server-tax-byte-identical` / `FactStore-seam-preserves-replay` in `specs/backend-lift.md`; the byte-identical-replay property generalised across hosts): the same fact set replayed through LedgerCore behind the file-backed `FactStore` (device) and the DynamoDB `FactStore` (Lambda) yields the **same projection digest and the same tax cells to the cent** (`web-source==file-source`). If the DynamoDB store reorders/drops/re-formats a fact, the digest diverges and the acceptance goes RED.

---

## Escalated → founder answered

### E1 — 2026-05-25 — Public "Account" naming KEPT (NOT renamed to "Wallet") — ESCALATED → founder
- **Decision.** Keep the public, user-facing term **"Account"** for a brokerage account throughout the UI, reports, and any future read API. A proposed rename to **"Wallet"** was **escalated to the founder, who chose to keep "Account."**
- **Reversible?** No, in practice. Once shipped in the UI, exported PDFs/CSVs, and (W6) a public read API, a public rename is expensive and trust-damaging to walk back.
- **Blast radius?** **External** — user-facing vocabulary + brand + documented surface. Precisely the class `references/decision-escalation.md` reserves for the founder ("wallet" implies stored value → unwanted financial-product/compliance connotations for a portfolio-valuation tool).
- **Rationale (why escalated, not decided).** Not an engineering choice with an outcome that can arbitrate it — there is no test that says "Account" is correct and "Wallet" is wrong. It significantly changes how the product presents itself, so the autonomy bar says **escalate**. To keep the milestone unblocked while pending, the term was kept everywhere behind a single internal label so the founder's answer would be a one-line flip, not a rewrite.
- **Outcome-harness.** The harness here is **the escalation process itself, not a test** — a public-naming decision has no falsifiable runtime outcome, which is exactly why it is a founder call. The proof it was handled correctly: it was **routed to the founder, staged ready-to-flip, and the milestone proceeded uninterrupted** while pending. Founder answer: **keep "Account."** (Contrast: had the CEO auto-renamed "Account"→"Wallet", that would be quietly making a founder-level call — the "autonomy trap.")
