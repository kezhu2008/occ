# Ledger — Phase-Next Review: Backend Migration, Event-Sourcing & Data-Model Audit

**Author:** ledger-ceo (One Consultancy) · **Reviewer:** ledger-board-reviewer → founder · **Date:** 2026-05-29 · **Status:** RESEARCH / PROPOSAL (not a build)
**Mandate (founder):** "During the pilot we discovered the ledger app cannot be scaled further than
myself due to many restrictions. Refactor or rebuild. Research and propose the next phase. Key
changes: (1) move the backend to AWS — sync/price fetching scheduled + event-triggerable; (2)
review the algorithm and data-model design — how is the event-sourcing system designed, what
events are supported, did we use state-machine concepts, what are the main states."

This review is grounded in the actual `LedgerCore` SwiftPM code and the `App/LedgerApp` layer
(every claim cites `file:line`), produced by four parallel deep-reads of `Sources/LedgerCore`, the
app container, and the decision log. It is the W6 entry artifact: it audits the current system, then
proposes the lift. The companion `system-design.md` (ledger-tech-lead owned) turns the recommendation
here into the deployable AWS design; `data-model.md` and `architecture.md` (ledger-architect owned)
are the authoritative model/invariant sources this review summarises.

---

## 0. Executive summary

**The verdict in one paragraph.** The *math* is already a server. The *plumbing* is welded to a
phone. Ledger's domain core (`Sources/LedgerCore`) is a pure, deterministic, Foundation-only
event-sourcing engine with no clock, no locale, no UI, and no Apple-only dependency — it compiles
and runs unchanged in an AWS Lambda. Everything that makes it single-user lives *around* that core:
one on-device `facts.jsonl`, one Keychain credential, one hand-coded account directory, and a sync
loop that only runs while the app is open. The "many restrictions" the founder hit in the pilot are
not algorithmic — they are five device couplings, each of which already has a named protocol seam
pointing at its server replacement (this was designed in from W1, the "server-lift swap-points"). So
the next phase is **a lift, not a rebuild**: keep the core verbatim, move storage/secrets/schedule/
notify/transport to AWS behind the existing seams, and put a thin API in front.

**The one decision that shapes everything:** does "scale beyond myself" mean (A) *get my own backend
off my phone* — a single-tenant always-on server that syncs unattended — or (B) *a real multi-user
product*? These are very different builds. The recommendation below is **staged**: build (A) first
(it is small, unlocks unattended scheduled sync, and is the literal pilot pain), then (B) as a
later wave only if the product goal is multi-user. Section 7 asks the founder to confirm.

---

## 1. The algorithm & data model — how the event-sourcing system is designed

### 1.1 The shape: two pure folds over an append-only log

```
IBKR Flex XML / file ─▶ Reconciler ─▶ [Fact]  (append-only canonical JSONL log)
                                         │
                                         ▼
   FactInterpreter.routedMutations  ─▶ [Mutation]
                                         │
                                         ▼
   LotLedger.buildAll (FIFO fold)   ─▶ LotLedgerState per key  (parcels)
                                         │
   + injected PriceOverlay / FXResolver / ProjectionConfig
                                         ▼
   ProjectionEngine.rebuild ─▶ Positions · Tax · Cash · Income · Optimise · Gaps · Stats
```

This is textbook event-sourcing: **facts are the only truth; everything else is a derived,
disposable projection.** Two properties make it unusually clean:

1. **Facts store only what happened, in native currency** — never a derived/marked/AUD value
   (`Facts/Fact.swift:6-18`). All AUD conversion and price marking happen at *projection* time
   from injected inputs (`Projection/Projection.swift:42-73`), never baked into the log. This is
   the "R7/R8" firewall: changing the price feed or FX source moves market value but can never
   move cost base or tax (`Pricing/DelayedQuoteOverlay.swift:9-11`).
2. **Deterministic replay ("A6")** — same facts + same config ⇒ **byte-identical** projections.
   No `Date()`, `UUID()`, `Locale.current`, or randomness anywhere in the core; `today`, FX, and
   marks are explicit inputs (`Support/ProjectionConfig.swift:102`; `Support/FY.swift:9-12`).
   FactID is a seed-free FNV-1a hash (`Support/Ids.swift:14-40`), parcel ids are deterministic
   `P-<symbol>-<seq>`, money is exact `Decimal` with rounding only at the edges via bankers'
   rounding (`Support/Money.swift:86-111`), and everything serialises through `CanonicalJSON`
   (sorted keys, exact Decimal — `Support/CanonicalJSON.swift:21-30`).

### 1.2 What a Fact is, and how identity/dedup works

A `Fact` (`Facts/Fact.swift:27-95`) carries: `id`, `stableSourceID`, `entityId`, `accountId`,
`instrument?`, `kind`, `eventDate` (day-granular `DateComponents`), `nativeAmount` (`Money`),
`quantity?`, `fxAtEvent?`, `fee?`, and an open `attributes: [String:String]` bag.

- **Identity is content-addressed on `(stableSourceID, kind)`** (`Support/Ids.swift:63-64`) — *not*
  the whole body. Re-importing the same statement row is a byte-stable no-op; dedup is an O(1)
  `Set<FactID>` guard on append (`Facts/FactStore.swift:98-105`). This is what makes sync idempotent
  with **no since-date/cursor** — the IBKR `queryId` defines the window server-side, and FactID
  dedup absorbs the overlap.
- **The store is structurally append-only** — no update/delete API exists (`Facts/FactStore.swift:32-147`).
  Tamper-detection is a single SHA-256 **content digest** over the whole sorted canonical log
  (`Facts/FactLog.swift:33-35`), recomputed on demand — not a per-fact hash chain.
- **Persistence:** one file, `facts.jsonl`, one fact per line, the *entire file re-sorted and
  rewritten atomically on every append* (`Facts/FactStore.swift:96-138`). SHA-256 is hand-vendored
  in pure Foundation (`Facts/SHA256.swift`) specifically to avoid Apple-only CryptoKit.

### 1.3 The event vocabulary — every `FactKind`

The full supported event set (`Facts/FactKind.swift:15-71`). Raw values are on-disk tokens and part
of the canonical artifact:

| # | FactKind | Meaning |
|---|----------|---------|
| 1 | `trade` | Buy/sell of share/ETF/option; direction = sign on `quantity`. |
| 2 | `dividend` | Cash dividend (AU franked or foreign). |
| 3 | `distribution` | Trust/ETF distribution. |
| 4 | `interest` | Cash interest. |
| 5 | `depositWithdrawal` | Capital-account cash in/out — audit only, **never assessable**. |
| 6 | `withholding` | Foreign tax withheld at source (split from a dividend; FITO). |
| 7 | `corporateAction` | Split / consolidation / DRP-restate (ratio/amount in `attributes`). |
| 8 | `optionExercise` | A long option exercised. |
| 9 | `optionAssignment` | A written option assigned. |
| 10 | `optionExpiry` | A long option expired OTM (premium → capital loss). |
| 11 | `transferIn` | External transfer-in; may have no cost basis → gap candidate. |
| 12 | `costBasisFill` | **User-appended** fact recording a resolved cost-basis gap (audit, A7). Not emitted by the reconciler. |
| 13 | `forexConversion` | AUD↔USD cash conversion leg (the most recently added, FXC/D74). Carries **both** signed legs + implied rate as canonical-Decimal attributes; no instrument, no parcel. Single source of truth for `CashProjection` *and* the Div-775 forex engine. |

Interpretation (`Mutations/FactInterpreter.swift:59-91`): trade→open/close, transferIn→open,
corporateAction→cost-base adjust, option lifecycle→close-option (+open-underlying for
exercise/assignment), costBasisFill→fill; income kinds and `forexConversion`→`[]` (they feed
projections directly, not the lot fold).

### 1.4 Projection versioning & rebuild

Each projection has an integer `version` (positions v2, tax v2, cash v2, stats v2, income v1,
optimise v1, gaps v1). A `StampedProjection` records `{name, version, factCount, factDigest,
output}` (`Projection/ProjectionVersion.swift:21-40`). A rebuild fires when the **version bumps**
(code change) **or the fact digest changes** (any fact added/changed), or on an explicit
`dropAll()` after a config/overlay/FX change (config is deliberately *not* in the digest —
`Projection/LedgerStore.swift:329-337`). The single `actor LedgerStore` owns all mutable state and
is the natural server request-handler boundary.

### 1.5 Data-model weaknesses (what to fix when we lift)

The *engine* is excellent; the *substrate* is single-device. Concrete smells, with the fix the
backend wants:

- **Single-file, whole-set-rewrite log.** `facts.jsonl` is read entirely into memory and fully
  rewritten on every append (`FactStore.swift:96-138`); the digest re-hashes the whole log on every
  projection read (`InMemoryProjectionStore.swift:72`). O(N) per write *and* per read. Fine for one
  IBKR history; wrong for a server. → **Partitioned/streamed durable store** (per tenant, per
  connection/account).
- **No FactStore seam.** `ProjectionStore` is a protocol (clean swap to SQLite/DB), but the actor
  holds a *concrete* `FactStore` (`LedgerStore.swift:35`) — the durable log has no interface to
  swap. → **Introduce a `FactStore` protocol** before the server lift; this is the single most
  important pre-refactor.
- **Identity = `(stableSourceID, kind)`, no revision concept.** If IBKR *restates* a row under the
  same id+kind (corrected amount), dedup silently drops the correction (`FactStore.swift:100`). →
  Add a supersede/revision path for restatements.
- **Content digest, not a hash chain.** Detects change but can't prove append-only history — weak
  for a tax-audit artifact. → Per-fact `prevHash` chain or Merkle root if audit-grade provenance
  matters.
- **`attributes: [String:String]` as an open union.** Economically significant data (franking %,
  FX legs, ratios, premiums, `netCash`) is stringly-typed; a key typo is a silent miss, not a
  compile error (`Reconciler.swift:226-232`). → Typed payloads per kind when convenient.
- **Day-granular ordering.** Same-day events tie-break on `stableSourceID` lexicographically, not
  execution time (`FactLog.swift:41-47`) — deterministic but not guaranteed economically correct
  for same-day FIFO/wash-sale edges. Acceptable; note it.
- **"Incremental" isn't cheap.** The fast path re-folds the entire combined fact set per update and
  discards untouched keys (`ProjectionEngine.swift:104-109`) — correctness-first, O(all facts).

---

## 2. The state model — what the states actually are

**Headline (corrected per founder, 2026-05-29):** the system's state is the **materialised aggregate
the event fold produces, at three nested levels — per-ticker, per-account, portfolio.** Each ticker
holds a state that the event stream transitions forward; account and portfolio "stats" are roll-ups
of those ticker states under a `Scope`. (A fourth axis, the tax **entity**, sits between account and
portfolio.) This — not the App-layer connection/sync enums — is the state machine of the system. An
earlier draft of this section mis-framed "state" as the device sync/connection FSMs; that is
plumbing, demoted to §2.4.

### 2.1 The unit of state — per-ticker `LotLedgerState`

The materialised state of one ticker is `LotLedgerState`, keyed by
`LotLedgerKey(entityId, accountId, instrumentSymbol)` — i.e. **one state object per
(entity × account × instrument)** (`Support/Ids.swift:83-91`; `LotLedger/LotLedgerState.swift:62-79`;
key built at `LotLedger/LotLedger.swift:193-196`). It holds:

- **open parcels** — each with its *preserved original* `quantity` + `costBaseNative`, its live
  `remaining`, `ParcelFlags`, and `ParcelOrigin`;
- **disposal lines** — each carrying the disposal terms + the FIFO `consumed` breakdown of which
  parcels supplied the units (`LotLedgerState.swift:31-49`).

`LotLedger.buildAll` produces the **set of all ticker states** — one per key — by folding the facts
in deterministic `(eventDate, stableSourceID)` order (`LotLedger.swift:64-92`). That set is the
ground state; every projection reads from it.

### 2.2 The per-ticker state machine — events drive the transitions

Each ticker's state is advanced by the events (`FactKind` → `Mutation` via
`Mutations/FactInterpreter.swift:59-91`, applied in `LotLedger.apply`). The transitions:

| Event | Transition on the ticker's state |
|-------|----------------------------------|
| `trade` BUY / `transferIn` | open a new parcel (`transferIn` may flag `missingBasis` → a gap) |
| `trade` SELL | FIFO-consume `remaining` across parcels, record a `DisposalLine` |
| `corporateAction` FS/RS (split) | **supersede** parcels with re-based quantity/cost (`splitAdjusted`) |
| `corporateAction` DRP-restate | add a cost-base `delta` to the latest open parcel (`costBaseRestated`) |
| `optionExpiry` | close the option parcel (premium → capital loss) |
| `optionExercise` (long) | close the option parcel **and** open the underlying (premium into cost base) — a cross-key transition |
| `optionAssignment` (written) | close the option parcel **and** open the underlying (premium reduces cost base) — cross-key |
| `costBasisFill` | clear the `missingBasis`/gap flag on the targeted parcel |

The lifecycle of each parcel *within* a ticker is itself a small machine read off `remaining`:
**open** (`remaining == quantity`) → **partially-consumed** (`0 < remaining < quantity`) →
**closed** (`remaining == 0`); or **written-short** (`remaining < 0`) winding back toward 0. Originals
are superseded, never mutated — the CGT-audit guard (`LotLedger.swift:18-22`). So "no stored enum"
does **not** mean "no state machine": the machine is the event-driven fold over each ticker's parcels.

### 2.3 The roll-up levels — account & portfolio stats

The same ticker states aggregate upward under a `Scope` (`Support/Scope.swift:11-43`,
`{ all | entity(id) | account(id) }`):

- **Per-ticker row** — `PositionsProjection.Row`, one per lot-ledger key, with netQty, market value,
  cost basis, unrealised, weight, flags (`Projection/Models/PositionsProjection.swift:69-96`). Cash
  has its own per-ticker-analogue: `CashProjection` folds a signed balance per
  `(entity, account, currency)`.
- **Account-level stats** — the roll-up of all tickers in one account:
  `PositionsProjection.Group` (grouping `.account`) with its own market-value total
  (`PositionsProjection.swift:158-166`); `Scope.account(id)`; cash balances per account.
- **Portfolio-level stats** — the roll-up across everything: `PositionsProjection.Totals`
  (market value, cost basis, unrealised, position count — `:130-137`); `Scope.all`; scope-relative
  weights computed against the portfolio total.
- **Entity level (the 4th axis)** — the tax-filing unit owning N accounts; `Scope.entity(id)`,
  grouping `.entity`. The tax engines (`TaxProjection`, `EOYSummary`) materialise their state at
  entity scope (`TaxKind` = individual / SMSF / company drives discount + treatment).

So the full state cube is **instrument × account × entity × portfolio**, all derived from the one
ticker-level ground state by deterministic aggregation.

### 2.4 The App-layer connection/sync enums (plumbing, NOT the domain state)

For completeness — these *exist* and are real enums, but they are device/UI concerns, not the
system's state model, and a server replaces them: `LedgerSource` `{ bundledFixture | realStatement }`
(`AppContainer.swift:104-122`); `ResyncResult` `{ idle | updated | upToDate | error | noToken }` (the
manual "Sync now" status, `:168-181`); `SampleState`, `SourcePreference`, `AddSearchState`. The one
genuine *domain* FSM outside the fold is **alert firing** — `AlertFiringCoordinator` armed ⇄ fired
(`Alerts/AlertFiringCoordinator.swift:69-131`), state threaded value-in/value-out, `asOf` a
parameter — built W3/W4 with the server/APNs seam in mind.

### 2.5 Migration impact of the state model

- **The ground state and its aggregation are pure and lift unchanged.** Per-ticker `LotLedgerState`
  → account → entity → portfolio is deterministic replay with no device dependency. On AWS this is
  exactly the data to **partition and store per tenant**: the natural partition key is the
  ticker-state key `(tenant, entity, account, instrument)`, rolling up to account/portfolio reads —
  which maps cleanly onto a Dynamo composite key or a Postgres `lot_state` table.
- **The sync/connection enums (§2.4) invert under a scheduled server.** No "Sync now" button machine:
  the server pulls on cron/event and the client only *observes* a read-only "last sync / next sync /
  status" view; `resyncInFlight` becomes server-side job locking, `lastRealPullAt` becomes server
  scheduling. The alert FSM and all folds move unchanged.
- **Pre-refactor still worth doing:** the App-layer connection state is scattered across ~8 booleans
  mirrored across `AppModel` + `SyncViewModel` (divergence risk) and token state collapses
  locked-vs-absent (`(try? load()) ?? nil`). Consolidating into one `ConnectionState` enum is the
  cheap cleanup to do during the lift — but note this is the *plumbing*, not the ticker/account/
  portfolio state that is the actual subject of this section.

---

## 3. Why it can't scale beyond one user (the pilot's "restrictions")

Not algorithmic — five device couplings. Each row is the literal blocker plus its evidence:

| # | Blocker | Evidence |
|---|---------|----------|
| a | **On-device-only storage.** All truth is `~/Library/Application Support/Ledger/real/facts.jsonl` on the one device; no shared/remote store. | `FactStore.swift:53-64`; `AppContainer.swift:1712-1723,1897-1903` |
| b | **No server / no API.** The app calls IBKR and Yahoo directly (Foundation URLSession, "no backend", D41). The app *is* the whole system. | architecture.md §1; D41 |
| c | **No auth / identity / multi-tenant.** Zero user concept; the store path has no tenant key; facts carry `entityId`/`accountId` but **no userId**. | D83 tangles T1/T6 |
| d | **Sync requires the one device open.** Pull is driven by `load()` / `resyncNow()` / a best-effort `BGAppRefreshTask` *on the device*; alerts are LOCAL notifications only (`aps-environment` deliberately absent, D46). | `AppContainer.swift:778-815`; `BackgroundRefreshController.swift` |
| e | **Secrets in device Keychain, device-only.** One `WhenUnlockedThisDeviceOnly` item, no backup/migrate, upsert-overwrites → **at most one connection**, never shareable. | `KeychainTokenStore.swift:29,69`; D83 T1 |
| f | **Compute on device, no shared cache.** Projections run on-device (RAM cache only); every user would independently re-pull the same Yahoo prices; whole-source wipe/migration ops nuke the entire `real/` dir (would nuke all accounts at N>1). | `InMemoryProjectionStore`; D83 T7 |

**A subtlety worth the founder's attention:** even *single-user N>1 accounts* is currently blocked,
not just multi-*user*. `RealDirectory` is hand-wired to exactly one entity ("Acme Holdings Pty Ltd") and
one account (`U0000000`) (`RealDirectory.swift:25-54`); the domain (`Scope`, `ProjectionConfig`,
N-entity reconciler map) already supports many. The ratified-but-deferred **D83
Connection/Account/Entity model** is the prerequisite for N>1 *before* multi-user even enters. There
is also a latent hazard (D83 T5): a second discovered account with no `EntityDef` silently gets the
0.5 CGT discount — founder accepted this at N=1, but it must be closed before any account growth.

---

## 4. What lifts unchanged vs what must be built

### 4.1 Reusable as-is (the asset inventory — this is most of the value)

The architecture was *designed* for this lift (the "server-lift swap-points", architecture.md:353).
Lifts into a Lambda **verbatim**:

- **The entire `Sources/LedgerCore` Foundation-only core** — Reconciler, Fact model, FactStore
  (byte-canonical, FactID-deduped), all tax engines (`CGTEngine`, `Div775Engine`, `FXGainEngine`,
  `FrankingEngine`, `FITOEngine`, `OptionTaxEngine`, `WashSaleDetector`), `LotLedger`,
  `DailyPLCalculator`, all projections, reports/CSV. A `PortabilityLockTests` already asserts the
  core imports Foundation only; `Package.swift` notes "Linux ignores `platforms`, so server-lift is
  unaffected" (D30).
- **The Flex ingest pipeline** — `FlexWebServiceRawRecordSource` (two-step SendRequest→GetStatement
  with `ndcdyn`→`gdcdyn` failover + polling), `FlexXMLParser`, `FlexTransport` (injectable HTTP).
  Runs unchanged in Lambda; idempotency is content-based (FactID), no cursor to migrate.
- **The quote pipeline** — `YahooQuoteProvider`, `YahooDailySnapshotProvider`, `QuoteService` actor
  (TTL cache), `SydneyMidnightBoundary` (pure, injected clock), all overlays + `FXResolver`.
- **The injectable seams already named as server swap-points:** `TokenStore` protocol → per-user
  server secret store (documented verbatim in `TokenStore.swift:22-25`); `ProjectionStore` →
  server DB; `FlexTransport`/`QuoteTransport` → keep or swap; `LocalAlertNotifier` ↔ server APNs on
  the same `FiredAlert` seam; `BackgroundRefreshController.runRefresh(...)` is *already* the
  fully-injected, `LedgerCore`-style headless pipeline a Lambda handler calls — only the
  `#if canImport(BackgroundTasks)` shell drops.

### 4.2 Must be built for AWS

1. **A `FactStore` protocol** (pre-refactor) so the durable log can be swapped for a server store.
2. **Server-side multi-tenant, partitioned durable store** (per tenant, per connection/account) —
   replacing the single per-device `facts.jsonl`. (DynamoDB or Postgres; see §6.)
3. **Per-user/per-connection credential storage** behind `TokenStore` (AWS Secrets Manager),
   supporting **multiple connections** (today's single-item upsert is the blocker).
4. **Scheduled + event-triggered sync** — EventBridge cron (scheduled) + an event rule / API
   trigger (on-event) invoking a Lambda that calls `runRefresh`/`ingestReal`. Per-user
   "last pull at" throttle in Dynamo to respect IBKR's per-token rate limit (error 1018).
5. **Server-side projection compute or cache**, and a **shared price cache** (one Yahoo/licensed
   pull serving all clients) instead of per-device fetches.
6. **A client API** (the SwiftUI app becomes a thin client) — read-only "sync health / next sync"
   view replacing the `ResyncControlVS` button machine.
7. **(If multi-user) identity + auth** (Cognito or similar) and tenant-scoped data isolation.
8. **The D83 Connection/Account/Entity model** — first-class `Connection` owning credential +
   ingest + FactStore partition + re-pull + migration; per-account partition; account→entity
   directory + assignment UI; R8 **safe conservative 0% fallback** for unrecognized accounts
   (never silent-50%). Prerequisite for N>1.
9. **A licensed price feed decision** — Yahoo is ToS-gray; from a fixed server IP the risk profile
   is worse than per-device. (D41 flagged this as a server-time decision.)

---

## 5. The two paths (the decision that shapes the build)

**Path A — Single-tenant always-on backend ("get *my* backend off my phone").**
One AWS account, one tenant (you), the IBKR credential in Secrets Manager, EventBridge cron pulling
IBKR + prices on a schedule (and triggerable), facts in a server store, alerts via push. The iOS app
becomes a thin viewer of server-computed state. **This is the literal pilot pain** — unattended
scheduled sync, no "phone must be open." Small, low-risk, mostly wiring against existing seams. No
auth/identity needed (single tenant, API-key/device-pinned).

**Path B — Multi-tenant SaaS ("a real product other people use").**
Everything in A, plus identity/auth (Cognito), multi-tenant data isolation, per-user onboarding &
billing, a licensed price feed, App-Store + legal/privacy (the frozen W5 scope), and the full D83
model. Much larger; only justified if the goal is to sell/serve others.

**Recommendation:** **stage it — build A first.** A directly removes the pilot restriction
(unattended sync), is a few weeks of mostly-wiring work, validates the entire server lift on real
data, and is a strict subset of B. Decide B *after* A proves the lift, when you know whether the
product is for you or for a market. Do the two non-negotiable pre-refactors (FactStore protocol +
ConnectionState consolidation + close the D83 T5 CGT-fallback hazard) inside Path A regardless,
because they're cheap now and expensive later.

---

## 6. Proposed AWS architecture (Path A, B-ready)

```
                       ┌──────────────── EventBridge ────────────────┐
                       │  cron: pull IBKR + prices (e.g. hourly +     │
                       │  Sydney-midnight snapshot)                   │
                       │  event: on-demand trigger (API / rule)       │
                       └───────────────────┬──────────────────────────┘
                                            ▼
   Secrets Manager  ◀──reads token──   Lambda: SyncWorker  ──writes──▶  Fact store
   (per-conn creds)                    (LedgerCore verbatim:            (DynamoDB / Aurora
                                        runRefresh + ingestReal)         Serverless; per
        ▲                                    │                           tenant+connection)
        │                                    ├──prices──▶ shared quote cache (Dynamo/ElastiCache)
        │                                    └──FiredAlert──▶ SNS → APNs (remote push)
        │
   Lambda: API (read)  ◀── HTTPS/Cognito ──  iOS thin client (SwiftUI; views server state,
   projections / sync-health / reports        triggers on-demand sync, receives push)
```

- **Compute:** Lambda running the unmodified `LedgerCore` + `runRefresh`/`ingestReal`. (Swift on
  Lambda via the AWS Swift runtime, or containerized Swift.) Deterministic replay means a cold
  Lambda rebuilds projections identically — no shared mutable state hazard.
- **Storage:** facts in DynamoDB (partition key = `tenant#connection`, sort key = `factId`; the
  FactID dedup maps directly to a conditional put) or Aurora Serverless v2 Postgres (one `facts`
  table, append-only, unique on `(tenant, connection, factId)`). Snapshots + shared quote cache in
  Dynamo.
- **Secrets:** Secrets Manager, one secret per connection, behind the existing `TokenStore`
  protocol.
- **Schedule + event:** EventBridge Scheduler (cron) for the heartbeat pull + the Sydney-midnight
  snapshot; an EventBridge rule / API Gateway route for on-demand "sync now".
- **Notify:** the `FiredAlert` seam → SNS → APNs (this finally lights up the deferred D46 remote
  push; needs `aps-environment` + an APNs key — a real prerequisite to verify).
- **Reuse the existing AWS muscle:** the org's Lambda + IaC patterns (known Terraform stacks under
  known profiles) already run under OIDC; reuse the deploy patterns and the `.npmrc`/credential
  gotchas captured in persona MEMORY rather than greenfielding the pipeline.

---

## 7. Recommended next-phase waves (proposal — founder to confirm)

Single project, sequenced. W6 is the lift; W7+ is the optional product. (Numbering continues the
existing wave log; the frozen W5 App-Store leg folds into W7 if Path B is chosen.)

- **W6 — Backend Lift (Path A, single-tenant unattended sync).** *Opens when:* founder confirms
  Path A + an AWS account/region/budget + IBKR token re-issued for server use. *Goal:* IBKR + price
  sync runs on AWS on a schedule and on-demand, with no phone open; iOS app reads server state.
  *The 7 milestones (proposed):*
  1. **Pre-refactors** — `FactStore` protocol + `ConnectionState` consolidation + close the D83-T5
     CGT-fallback hazard (all in-app, no behavior change, R8-locked). *Persona chain:*
     ledger-architect designs the seam → ledger-architect-reviewer → ledger-backend-engineer
     implements → ledger-backend-engineer-reviewer → ledger-qa (A6/A8 determinism unchanged).
  2. **Server core on Lambda** — package `LedgerCore` for Linux/Lambda, `runRefresh` handler,
     determinism proven server-side against the real `.flex-local` bytes.
  3. **Durable store + secrets** — Dynamo/Aurora fact store behind the new protocol, Secrets Manager
     `TokenStore`, idempotent conditional put.
  4. **EventBridge schedule + on-demand trigger** + per-connection rate throttle (IBKR error 1018)
     and a per-connection job lock.
  5. **Read API + thin iOS client** — app views server-computed projections + sync health, triggers
     sync. *Persona chain:* ledger-ios-engineer (thin client) + ledger-backend-engineer (API) →
     respective reviewers → ledger-qa parity check.
  6. **Remote push** — `FiredAlert`→SNS→APNs (verify APNs key/`aps-environment`).
  7. **Acceptance + deploy** — end-to-end on real data, tax numbers **byte-identical** to the
     on-device build (the A6 acceptance gate; ledger-qa owns the cross-build digest diff).
- **W7 — Multi-tenant / product (Path B) — CONDITIONAL** on the founder wanting multi-user. *Opens
  when:* W6 done + founder commits to multi-user + licensed-feed budget + auth choice. *Goal:*
  identity/auth (Cognito), tenant isolation, full D83 Connection model + assignment UI, licensed
  price feed, App-Store + privacy/legal (absorbs frozen W5).

---

## 8. Prerequisites to verify before W6 build (Preflight — not yet run)

These are hard gates; none verified yet (this is a research turn). Per OCC automation policy, these
are the access pre-flight checks that gate the W6 build, owned by ledger-tech-lead:

1. **AWS account + region + credentials** usable from this environment (org Lambda/IaC profiles
   exist; confirm which account hosts Ledger and the budget ceiling).
2. **Swift-on-Lambda toolchain** — AWS Swift runtime or container build for the Linux `LedgerCore`
   target (the core is portable; the *deploy* path needs proving).
3. **IBKR Flex token re-issued for unattended server use** — confirm the per-token rate limit
   (error 1018) tolerates a scheduled cadence, and that a server IP isn't geo/policy-blocked.
4. **APNs** — Apple push key + `aps-environment` entitlement (W6-M6 remote push; deliberately absent
   today per D46).
5. **Price feed** — decide keep-Yahoo (server-IP ToS risk) vs a licensed feed (cost).

---

## 9. Bottom line

The pilot's "many restrictions" are five device couplings, not a flawed design — and the original
architecture pre-placed a named protocol seam at every one of them. The deterministic
event-sourcing core is the asset; it moves to AWS unchanged. The next phase is a **staged lift**:
W6 gets *your* backend off your phone (scheduled + event-triggered sync, the literal pilot pain),
doing the two cheap pre-refactors along the way; W7 turns it into a multi-user product only if that's
the goal. The single decision that gates the plan is **Path A vs A-then-B** (§5/§7). On founder
sign-off this review hands to ledger-tech-lead, who turns §6–§7 into `system-design.md` and the W6
milestone/tasks breakdown.
