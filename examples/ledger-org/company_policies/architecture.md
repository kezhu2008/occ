# Ledger Architecture

> Owned by the **Architect** (reviewed by `ledger-architect-reviewer`). The authoritative **current-system** technical design: the shape the running software actually has, the invariants that realize the charter's non-negotiables, and the seams that keep the W6 backend lift a *lift, not a rebuild*. Version-stamped; supersedes the per-wave design notes it links. **The canonical authority for decisions is `decisions.md`** (the load-bearing pointers below: D4, D5, D7, D8, D31, D45, D46, D48, D49, D58, D67, D74, D75, D80, D84, D85); the canonical authority for the spec contract is `specs/domain-core.md` (the display-never-moves-tax invariant is **R23**, proven by **A18**). When this doc and a per-wave plan disagree, **the code wins** — drifts are called out inline.
>
> Scope boundary (do NOT merge with `system-design.md`): this doc answers *"is the current system right?"* — the invariants the system enforces on itself. The next-phase AWS topology/API/migration is the Tech-Lead's `system-design.md`. The seam **between** the two (the `FactStore` protocol) is the consultation edge and is described here as the pre-refactor it is.
>
> Stack: Swift 6.2 (strict concurrency). Foundation-only `LedgerCore` SwiftPM library + a thin SwiftUI app (`com.solo.ledger`). State: W1 core+tax DONE; W2 real-data path IN PROGRESS; W6 backend lift designed against the seams below.

---

## System shape

The pipeline is one direction: broker statement bytes become immutable **facts**, facts fold into **projections**, projections render **reports**. Everything below the fact log is a pure, disposable fold.

```
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  INGEST          IBKR Flex Web Service        user-picked Flex XML / file  │
   │  (Foundation)    (FlexTransport seam,         (FileRawRecordSource, D85)   │
   │                   Keychain token, D31/D48)                                 │
   └─────────────────────────┬───────────────────────────────┬────────────────┘
                             │ raw XML bytes                 │ raw XML bytes
                             ▼                               ▼
                   ┌───────────────────────────────────────────────┐
                   │  Sources/LedgerCore/Ingest                     │
                   │  FlexXMLParser → [RawRecord]                   │  (Foundation only)
                   │  RawRecordSource  (the W6 server seam ↑)      │
                   └─────────────────────────┬─────────────────────┘
                                             │ RawRecord (deterministic)
                                             ▼
                   ┌───────────────────────────────────────────────┐
                   │  Sources/LedgerCore/Facts                      │
                   │  Reconciler → [Fact]   (append-only, FactID    │
                   │                          content-dedupe, A5)   │
                   │  13 FactKinds (closed taxonomy, exhaustive)    │
                   │  FactStore  → canonical JSONL, per-source dir  │
                   │     (the W6 FACTSTORE-PROTOCOL pre-refactor)   │
                   └─────────────────────────┬─────────────────────┘
                                             │ THE ONLY SOURCE OF TRUTH (D5)
                                             ▼
        ┌────────────────────────────────────┴────────────────────────────────┐
        │  Sources/LedgerCore/Projection   — pure folds, version+digest stamped │
        │  LotLedger / FactInterpreter (ground state)                          │
        │  Positions · Cash · Income · Gaps · LedgerStats · Optimise · Tax     │
        │  Tax engines: CGT · Div775 · FXGain · Franking · FITO · Option ·     │
        │               WashSale · EOYSummary                                  │
        │  OUTSIDE the tax pipeline (R23/D67): DailyPLCalculator (pure fn)      │
        └────────────────────────────────────┬─────────────────────────────────┘
                                             │ projection outputs (Codable, Sendable)
                                             ▼
        ┌──────────────────────────────────────────────────────────────────────┐
        │  App/LedgerApp  (SwiftUI thin client)  —  reads outputs, draws cells   │
        │  AppContainer / AppModel · per-screen VMs · BG refresh                │
        └──────────────────────────────────────────────────────────────────────┘

              ─── OVERLAY LANE (display only, R23 — NEVER enters the fact log) ──
   QuoteTransport ─► YahooQuoteProvider / YahooDailySnapshotProvider
                       ├─ delayed marks (TieredPriceOverlay) → Positions rows (display)
                       └─ per-Sydney-day snapshot → DailyPLCalculator (frozen-FX) → TODAY column
```

**Three structural invariants the diagram encodes** (and that every reviewer enforces):
1. **The fact log is the only truth (D5).** Everything beneath it is a pure fold and is disposable — projections are a rebuildable cache, never authored state.
2. **Display never moves tax (R23/D67).** Quotes, marks, daily P/L, snapshots, and the `netCash` display attribute feed valuation only. The *only* engine that intentionally moves a tax number is `Div775Engine` (forex realisation, D74/D84).
3. **The overlay lane is app/provider-layer and display-only.** A price feed swap can move market value; it can never touch cost base, CGT, income, or EOY.

The same vertical pipeline is what the **W6 backend lift** re-homes: `RawRecordSource` gets a server implementation, `FactStore` moves behind its protocol onto DynamoDB, the projection fold runs verbatim in Lambda, and a thin read API replaces the SwiftUI client. The core (Fact → fold → engines) moves *unchanged* — that is the "lift, not rebuild" promise, and it is only possible because of the invariants below.

---

## Technical invariants

> The engineering realizations of the charter's business non-negotiables. A change that violates one of these is a FAIL at review regardless of test status.

- **Money is `Decimal`, never `Double` (D7).** `Money = {amount: Decimal, currency: String}`, scale 2, bankers' rounding at the edges only, arithmetic traps on currency mismatch. Realizes the charter's *"accurate to the cent"* promise; `Double` is banned in the money path by construction.
- **`LedgerCore` imports Foundation only (D4).** No UIKit, SwiftUI, SwiftData, or any UI/persistence framework crosses into the core. Enforced by a standing portability test (`PortabilityLockTests`). This is **the W6 server-lift seam**: the core compiles into a Lambda by construction, because it never depended on the device.
- **Native-only facts (R3).** A `Fact` stores the signed amount in the instrument's **native currency** plus the raw FX rate observed; it NEVER stores an AUD/marked/derived value. AUD conversion happens at projection time via the injected `FXResolver`. `fxAtEvent == nil` is the FX-gap signal — **absent FX is never fabricated** (an empty cell is `—`, never `0`).
- **Facts are the only persisted truth (D5).** Lot states, positions, tax cells, and reports are all pure projections folded over `[Fact]`. Derived state is always a rebuildable cache.
- **Append-only + content-addressed identity (A5).** The log is append-only; `FactID = hash(stableSourceID, kind)` makes re-ingest idempotent with **no cursor** — re-pulling an overlapping statement is a byte-stable no-op.
- **Entity + account on every fact (R4).** Tax is per-entity and accounts are isolated, so `entityId` + `accountId` are stamped at reconcile time on every fact, never bolted on later. The keystone join (D39): `EntityDef.id` MUST equal the stamped `Fact.entityId`; an unmatched account fails SAFE to the most conservative 0% discount, never a silent 50% (`data-model.md` I9, spec R28/A26).
- **Deterministic serialization (D8).** The JSONL log is written via `CanonicalJSON` — sorted keys, fixed `Decimal` formatting, `(eventDate, stableSourceID)` order. Same facts ⇒ **byte-identical** bytes. This makes the SHA-256 `digest()` a meaningful projection stamp and makes byte-identical-replay (the **A3** property, proving R8/R11) a cheap test rather than a heroic one.
- **Tax never moves on display inputs (R23).** Bound by a non-vacuous, mutation-checked test (below) — the byte-identical-tax anchor **A18**. The single intentional mover is `Div775Engine`.

These invariants are what make the W6 acceptance bar — *server tax numbers BYTE-IDENTICAL to the on-device build* — achievable: byte-canonical facts + deterministic folds mean the same `[Fact]` produces the same cents on a phone and in a Lambda.

---

## Determinism / concurrency contract

**Determinism (D8) — what `Projection.build` may read:**
- Only `[Fact]` + an explicit `ProjectionConfig`. **No `Date()`, `Locale.current`, `TimeZone.current`, `UUID()`, or randomness** inside a fold.
- `today`, the FX resolver, `ParcelPolicy`, and the Div-775 election are *explicit* `ProjectionConfig` inputs — not ambient reads.
- The total fold order is `(eventDate, stableSourceID, FactID)`; day-granular, same-day ties broken on id (a deliberate, deterministic trade-off). The `Div775Engine` weighted-average pool additionally folds **same-day inflows before outflows** (D84) — a load-bearing rule, not an accident of sort order.
- The Sydney-midnight instant is computed **once** in the app (`SydneyMidnightBoundary`, DST-aware, injected zone + clock) and threaded into the core as an explicit `Date` — no implicit clock ever enters a fold.

`FactID` uses a **seed-free FNV-1a** hash (not Swift's per-process-randomized `Hasher`) so identity is reproducible across runs, devices, and the W6 server.

**Concurrency (Swift 6.2 strict):**
- Every value the core produces — `Fact`, `FactKind`, all projection `Output`s, every tax engine result — is `Sendable` by being an immutable value type or an `enum` namespace of pure free functions. The tax engines have no shared mutable state to protect.
- The single serialization point for mutation is the app-layer `actor LedgerStore`; the `FactStore` itself is **append-only, single-owner, no internal lock** — it is mutated only through `append`/`append(contentsOf:)` behind the actor.
- Gates that mutation-probe the **same source file run sequentially, not concurrently (D58)** — a same-file race produced a reviewer false-negative; serial probing (or per-worktree isolation) is the ratified fix.

---

## Persistence strategy

State is an **append-only Codable JSONL fact log**; everything else is rebuilt from it.

- **The log.** `FactStore` writes `facts.jsonl` — one canonical JSON fact per line, rewritten in `(eventDate, stableSourceID)` order on every mutation. `digest()` (SHA-256 over the canonical bytes) is the change-detection stamp.
- **Per-source isolation (D49).** The store root is `…/Library/Application Support/Ledger`; the active source appends a subdirectory — `…/Ledger/demo` vs `…/Ledger/real` — so demo facts and real facts never share a `facts.jsonl`. The shipping app never resolves to demo (D85); a fresh/no-token/disconnected launch loads the empty *real* store and shows the CONNECT IBKR CTA, never fabricated holdings.
- **Rebuild by replay.** Each projection output is stamped `(name, version, factDigest)`. A version bump or a digest mismatch ⇒ rebuild-by-replay (R11) — the **A3** byte-for-byte guarantee. A model-version bump (D80) is a clean-wipe + full re-pull (real data is re-pullable from the Keychain token), not an in-place migration.
- **Config is not facts.** The user's tax position (`ProjectionConfig` / `EntityDef`: entities, account→entity, `TaxKind`, CGT discount, Div-775 election, option treatment) is per-entity byte-canonical data that drives the fold; a config change invalidates the projection cache but is not in the fact digest.

The exact field shapes, the 13-kind vocabulary, the attribute schema, and the physical (DynamoDB) layout live in **`data-model.md`** — this section is the strategy, that doc is the schema.

---

## Seams for future waves

> Swappable boundaries that keep later waves — above all **W6 (backend lift)** — non-blocking. Each is a real protocol in the current tree, exercised today by a fake in tests, so the server implementation drops in without touching the core.

| Seam | Today (on-device) | Future wave (target) |
|------|-------------------|----------------------|
| **`RawRecordSource`** | `FileRawRecordSource` + `FlexWebServiceRawRecordSource` (device-side Flex pull) | W6: a server-side `SyncWorker` pull (EventBridge-scheduled/triggered) → same `[RawRecord]` |
| **`FactStore` (the key pre-refactor)** | append-only JSONL behind `actor LedgerStore`, per-source dir | W6: a `FactStore` **protocol** over multi-tenant DynamoDB; `attribute_not_exists(SK)` conditional put = FactID dedupe for free |
| **`ProjectionStore`** | `InMemoryProjectionStore` (rebuilt each launch) | W6: a server `StampedProjection` cache keyed `(name, scope, factDigest)`; same contract |
| **`QuoteTransport` / `QuoteProvider`** | `URLSessionQuoteTransport` + `YahooQuoteProvider` (per-device fetch) | W6: a shared `QuoteCache` table (one fetch per symbol serves all tenants) |
| **`TokenStore`** | `KeychainTokenStore` (`WhenUnlockedThisDeviceOnly`, D48) | W6: per-tenant AWS **Secrets Manager** behind the same protocol |
| **`LocalAlertNotifier`** | `UNUserNotificationCenter` local notifications (D46; `aps-environment` ABSENT) | W6: SNS → APNs remote push behind the same notifier seam |
| **`FXResolver`** | `StatementFXResolver` (rates from the statement) | unchanged — already an injected, deterministic input |

**The `FactStore` seam is the load-bearing one for W6.** Because the store is already (a) the sole truth, (b) append-only, (c) content-addressed (idempotent, cursor-free), and (d) byte-canonical, moving it to DynamoDB is a storage swap, not a logic change: the conditional put gives idempotency for free, the partition key falls straight out of `(tenant, connection)` / `(entity, account, instrument)`, and the projection fold above it is untouched. This is the precise meaning of "**a lift, not a rebuild**," and it is the consultation edge between this doc (Architect: the protocol the system enforces) and `system-design.md` (Tech-Lead: the DynamoDB tables and Lambda topology that operate it).

---

## The R23 lock (how the centerpiece invariant is proven, not asserted)

R23 — *display/cash/daily-P/L/snapshot/overlay/quotes NEVER move tax* (the "R8 firewall" in older prose) — is the invariant most likely to rot under a refactor, so it is bound by a **non-vacuous, mutation-checked** test (the byte-identical-tax anchor **A18**), not a comment:

1. Assemble the full `TaxProjection` in two worlds — `{no daily-P/L}` vs `{daily-P/L + cash + snapshot}` — over the real `.flex-local` bytes, in canonical JSON.
2. Assert byte-identical across both worlds and across two *different* non-degenerate snapshots.
3. **Negative control (the mutation):** deliberately leak `snapshot.mark` / `currentFX` / `netCash` into a tax input — the test MUST turn RED. A test that only re-asserts the VM number it was handed is *vacuous* and is rejected at review (the 0.9.1 `RenderedMoney` "drawn == asserted" lesson).

Acceptance oracles are **independent ground truth (D75)** — first-principles re-derivations over the real bytes plus boss live confirmation — never the projection's own internal sum. The one accepted tax mover, `Div775Engine`, is validated against a hand-computed cost-base walk over the real legs (FY26 real-bytes total: −$1.76 AUD, D84).

---

## Decision pointers

The load-bearing entries in `decisions.md` (the canonical decision log; one-line rationale here only — do not treat this table as authority). The one spec-contract pointer (**R23**) lives in `specs/domain-core.md`, not `decisions.md`:

| Pointer | Home | Rationale |
|---------|------|-----------|
| **R23** | `specs/domain-core.md` | display/cash/daily-P/L/snapshot/overlay/quotes never move tax (proven by **A18**); the sole intentional mover is `Div775Engine`. |
| **D4** | `decisions.md` | two-target shape — Foundation-only `LedgerCore` + SwiftUI app; the W6 server-lift seam. |
| **D5** | `decisions.md` | append-only Codable JSONL fact log + in-memory projections behind a `ProjectionStore` seam. |
| **D7** | `decisions.md` | `Money` is `Decimal`, never `Double`. |
| **D8** | `decisions.md` | determinism contract — `(eventDate, stableSourceID, FactID)` order; no clock/locale/UUID inside a fold. |
| **D31/D48** | `decisions.md` | tokens live only in the Keychain `TokenStore` seam; never in repo/ledger/logs/UserDefaults/factstream. |
| **D45** | `decisions.md` | persona-skill discipline — design-time real-dependency probe, mutation/negative-control on every new test, paired independent oracle. |
| **D46** | `decisions.md` | W4 price alerts = local notifications; server-pushed APNs deferred to W6 on the same `FiredAlert` seam. |
| **D49** | `decisions.md` | per-source FactStore directory — demo and real never share `facts.jsonl`. |
| **D58** | `decisions.md` | gates that mutation-probe the same source file run sequentially. |
| **D67** | `decisions.md` | `DailyPLCalculator` is a pure fn in core but OUTSIDE the tax pipeline; never feeds tax inputs. |
| **D74** | `decisions.md` | the shared `.forexConversion` fact serves BOTH `CashProjection` (display/R23) AND `Div775Engine` (tax) — single source of truth. |
| **D75** | `decisions.md` | acceptance oracles are independent first-principles ground truth, never the projection's own sum. |
| **D80** | `decisions.md` | per-source `modelVersion` stamp → clean-wipe + full re-pull when stale (safe per D49). |
| **D84** | `decisions.md` | Div 775 scope = all USD currency disposals; weighted-average pool folds same-day inflows before outflows. |
| **D85** | `decisions.md` | shipping app NEVER shows demo; disconnect → honest-empty under `.realStatement`; copy-truth guards on every surface. |

Companion docs: `data-model.md` (types/schema/serialization — Architect-owned), `system-design.md` (next-phase AWS topology/API/migration — Tech-Lead-owned), `backend-migration-review.md` (the W6 Path A vs B proposal, 7 milestones).
