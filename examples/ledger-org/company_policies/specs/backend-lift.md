# Backend Lift (W6) Backend Spec

> **Co-owned by Product Manager + Architect.** PM frames the required behaviour from the W6
> use-cases (`use-cases/backlog-backend.md`); the Architect makes it a precise, invariant-bearing
> contract grounded in `backend-migration-review.md` + `system-design.md`. `pm-reviewer` **and**
> `architect-reviewer` both gate this file. Lives in `company_policies/specs/backend-lift.md`.
>
> This is the **behavioural contract a developer implements against and a test asserts against** —
> no guessing. Organized **by subsystem, not by use-case** (one `R#`/`A#` backs many use-cases).
>
> **The governing principle: "a lift, not a rebuild" (review §0, system-design §Framing).** The
> deterministic, Foundation-only event-sourcing core (`Sources/LedgerCore`) is preserved **verbatim**
> and runs server-side unchanged. This spec is the contract for everything that lands *around* it:
> the `FactStore` protocol seam, multi-tenant durable storage, server-held secrets, unattended
> scheduled + event-triggered sync, the read/write API, and the binding that none of it moves a tax
> cell. The **acceptance bar for the whole wave** is the boss's W6 acceptance: *end-to-end on real
> data, tax numbers BYTE-IDENTICAL to the on-device build* (review §7-M7; system-design G2/NFR1).
>
> **Two parts, both load-bearing:**
> - **`R#` requirements** — what the system MUST do/hold. Behaviour and invariants,
>   **architecture-agnostic** (state the *what*, not the *how*; DynamoDB/Cognito/EventBridge are the
>   *design's* choices in `system-design.md`, not requirements here — a requirement says "per-tenant
>   isolation", the design says "partition prefix").
> - **`A#` acceptance** — **executable** facts. A test asserts each *without a human judging*.

## Key acceptance anchors

> **The stable concept → `A#` map for this spec (the W6 server lift).** Other files (`milestones/W6.md`, `tasks/W6-M*.md`, the W6 job-descriptions) cite the `A#` by looking up the concept here. **This file owns its own FLAT `A#` namespace** — its `A1`/`A2`/`A6`/… are W6-lift anchors and are **distinct** from the same-numbered on-device anchors in `domain-core.md`; a W6 file resolves a citation through THIS table, an on-device file through `domain-core.md`'s table. The shared **byte-identical-replay / A6** *property* is carried across the seam by `FactStore-seam-preserves-replay` (A1) and `server-tax-byte-identical-to-device` (A2) below.

| Concept | A# | One-line assertion (a test can assert this) |
|---|---|---|
| `FactStore-seam-preserves-replay` | **A1** | With `LedgerStore` reaching the log **only** through `FactLogStore`, the on-device tax-oracle + rebuildability suites pass **unchanged** and the `TaxProjection` digest **== golden `7f6784e1…`** by exact string equality; a path that bypasses the protocol FAILs. |
| `server-tax-byte-identical-to-device` | **A2** | Server `FactLogStore` and device file store feed the same engine ⇒ full `TaxProjection` (EOY+CGT+income+forex) via `CanonicalJSON` is **byte-identical** (1¢/1-byte delta FAILs); FY26 Div775 line **== −$1.76 AUD** on the server path. **(THE wave bar.)** |
| `multi-tenant-isolation` | **A5** | A request authenticated as T1 returns **only** T1's data and cannot read/mutate any T2 resource; a forged body `tenantId` is ignored (verified identity is the key); T2 leaked-byte count **== 0**. |
| `unattended-scheduled-sync` | **A8** | The server sync handler wired with server collaborators (canned credential, fake transport returning the real recorded bytes, fixed clock) invoked `reason=.scheduled` then `.onDemand` runs the **same** `runRefresh` body to the **same** ingest result; the chosen instant flows through `SydneyMidnightBoundary` to a deterministic snapshot key. |
| `server-R8-relock-byte-identical` | **A4** | Through the **server** path, the full `TaxProjection` is **identical** across `{no daily-P/L}` vs `{daily-P/L + cash + two snapshots}` while MV/daily-P/L move; a leak of `snapshot.mark`/`currentFX`/`netCash` into a tax input ⇒ RED. |
| `idempotent-server-ingest-no-cursor` | **A3** | Re-ingesting the same statement bytes ⇒ `append` returns 0 newly-persisted, `factCount` unchanged, digest unchanged (no since-date/cursor). |
| `N-connections-isolated-removal` | **A6** | Removing connection C2 deletes only C2's credential + stops only C2's sync; C1's credential/facts/schedule unchanged; `Scope.all` drops exactly C2's contribution. |
| `safe-entity-fallback-server` | **A7** | An unassigned discovered account computes with a **0%** CGT discount (never 50%) and is flagged; assigning it applies 50% and clears the flag; defaulting to 50% FAILs. |
| `token-never-leaks` | **A11** | The IBKR token appears in **no** log / error type / API response / fact JSONL / projection / export (grep-clean); the read/API role has no path to a credential. |
| `append-only-exportable-server` | **A16** | The store exposes **only** `append`/`allFacts`/`digest` (no update/delete); `allFacts()` is canonical `(eventDate, stableSourceID)` order; a per-tenant export round-trips to the same `digest()`. |

> When a W6 file says **"the A6 byte-identical-replay suite stays green"** it means the **on-device** replay property (`domain-core.md` A3, concept `byte-identical-replay`) re-run through the lifted seam — proven here by **A1** (`FactStore-seam-preserves-replay`) and **A2** (`server-tax-byte-identical-to-device`). When it says **"R8"** it means `domain-core.md` R23 (`display-never-moves-tax`), re-locked across the lift by **this file's R23 / A4**. See **`## Alias map (old → canonical)`** at the bottom for the cross-file resolution.

## Subsystem

The **server lift** of LedgerCore: the seam, storage, secrets, schedule, transport, and API layers
that move sync + compute off the single iOS device onto an always-on backend, **without changing the
domain core**. It **owns**: the `FactLogStore`/`TokenStore`/`ProjectionStore` protocol seams and
their server implementations; the per-tenant durable fact log; server-held per-connection
credentials; the scheduled + event-triggered sync worker (a thin shell over the existing
`runRefresh`); the multi-tenant isolation boundary; the read/write API contract and its payload
types; and the binding that server compute is byte-identical to device compute. It **calls out to**
(does not own, must not modify): the entire `Sources/LedgerCore` math — `Reconciler`, `FactKind`
taxonomy, `LotLedger`/`FactInterpreter`, every projection, every tax engine (`CGTEngine`,
`Div775Engine`, `FXGainEngine`, `FrankingEngine`, `FITOEngine`, `OptionTaxEngine`,
`WashSaleDetector`, `EOYSummary`), `DailyPLCalculator`, the quote/snapshot overlay lane, and
`CanonicalJSON`. Those are the asset; this subsystem is the plumbing the original architecture
pre-placed seams for (the "W5 swap-points", architecture.md §11/D4).

## Backed use-cases

From `use-cases/backlog-backend.md` (the W6 future-state set; all `(to be implemented)`):

- Identity / account: `UC-ACCT-1`, `UC-ACCT-2`, `UC-ACCT-3`, `UC-ACCT-4`
- Connections: `UC-CONN-1`, `UC-CONN-2`, `UC-CONN-3`, `UC-CONN-4`, `UC-CONN-5`, `UC-CONN-6`, `UC-CONN-7`
- Sync & freshness: `UC-FRESH-1`, `UC-FRESH-2`, `UC-FRESH-3`
- Alerts (server / push): `UC-PUSH-1`, `UC-PUSH-2`
- Reading round-trip: `UC-READ-1`, `UC-READ-2`, `UC-READ-3`

> Trace is bidirectional: every UC above references the `R#`/`A#` below; every `A#` below appears in
> at least one UC **or** is a named invariant carried from `decisions.md` (R8/D8/D31/D45/D49). A UC
> with no backing spec item — or a spec item no UC/invariant uses — is a gap the reviewers FAIL.

---

## Requirements (R#)

> Each `R#` is a behaviour or invariant the system MUST hold, stated **architecture-agnostic** (no
> AWS service names where a capability word will do; the *how* is `system-design.md`). One
> requirement, one line. Number monotonically; never renumber — append.

### A. The seam (the critical pre-refactor — must land first, no behaviour change)

R1. **FactStore protocol seam.** The durable fact log is reached **only** through a `FactLogStore`
protocol — `append(_:) -> Int` (returns # newly persisted), `allFacts() -> [Fact]` in canonical
`(eventDate, stableSourceID)` order, `digest() -> String` — so the concrete on-device file store and
a server store are interchangeable behind one interface. Today `LedgerStore` holds a **concrete**
`FactStore` (review §1.5, §4.2); introducing this protocol is "the single most important pre-refactor"
and is an in-app refactor with **zero behaviour change** (R8-locked, all determinism tests stay green).

R2. **Secrets seam.** Per-connection IBKR credentials `{token, queryId}` are reached **only** through
the existing `TokenStore` protocol (the documented W5 swap-point, `TokenStore.swift:22-25`), so the
device Keychain implementation and a server secret-store implementation are interchangeable with
**nothing upstream changing**.

R3. **Projection-store seam preserved.** Server-side projection caching uses the existing
`ProjectionStore.loadOrRebuild` contract (version + fact-digest stamped); a stamped projection is
reused while `version` and `factDigest` match, else rebuilt by replay (R5 of the W1 spec, unchanged).

### B. Determinism & correctness (the lift's whole point — G2/NFR1)

R4. **Core runs server-side verbatim.** `Sources/LedgerCore` is deployed unmodified; no tax/CGT/
income/cash/option/Div775/projection logic is re-implemented, re-ordered, or ported to another
language. The `PortabilityLockTests` (Foundation-only, no UIKit/SwiftUI/SwiftData) remains green as
the gate that the core is server-liftable by construction.

R5. **Server replay is byte-identical to device replay.** For the same facts + the same
`ProjectionConfig`, server-computed projections — and in particular the full `TaxProjection` (EOY
cells + CGT disposals + income + forex + `forexRealisations`) serialised via `CanonicalJSON` — are
**byte-identical** to the on-device build's output (the "A6" determinism property holds across the
lift). This is the wave's acceptance bar.

R6. **No clock/locale/randomness leaks in the lift.** The server lift introduces no `Date()`,
`TimeZone.current`, `Locale.current`, `UUID()`, or randomness into any core fold or its server
wiring; `today`, FX, marks, the Sydney-midnight instant, and the Div775 election remain **explicit
inputs** threaded in by the handler (D8). FactID stays the seed-free FNV-1a hash of
`(stableSourceID, kind)`; money stays exact `Decimal`; JSON stays sorted-keys/fixed-Decimal.

### C. Idempotent ingest & the append-only log (FR2, NFR3, NFR6)

R7. **Content-based idempotent ingest, no cursor.** Ingesting a pulled statement dedupes on FactID;
re-pulling or re-ingesting the same statement (overlapping windows, repeated schedules, a re-seed) is
a **no-op** on the fact set. There is no since-date/cursor to track — the IBKR `queryId` defines the
window and FactID dedup absorbs the overlap (review §1.2; system-design §5).

R8. **Append-only durable log.** The server fact log is structurally append-only — no update/delete
of a persisted fact; corrections are new facts (the supersede/revision path is a flagged backlog
item, system-design §5.1, **out of scope** for this lift unless OD4 opens it). The per-tenant log is
exportable and a content digest over its canonical bytes is computable on demand (NFR6).

### D. Multi-tenant isolation (FR8, NFR2, G3)

R9. **Per-tenant data isolation.** Every stored fact, projection, connection, credential, snapshot,
and alert is owned by exactly one tenant; no request can read or write another tenant's data. The
isolation key is a **server-verified tenant identity** (derived from the authenticated session),
**never a client-supplied value**.

R10. **N connections / N accounts / N entities per tenant.** A tenant may hold multiple connections,
each with its own credential and its own sync schedule/status; multiple discovered accounts; and
multiple tax entities. Removing one connection stops only that connection's sync and deletes only its
credential, leaving other connections and their already-ingested facts intact (closes D83-T7 "wipe
nukes everything"). This replaces the device's single-credential Keychain upsert (review §3-e).

R11. **Safe entity-assignment fallback (D83-T5 / D83-S3).** A discovered account that cannot be
auto-classified to a tax entity is flagged "needs assignment" and computed with a **conservative 0%
CGT discount** until assigned — **never a silent 50%**. Assigning an account triggers a tax recompute
for the affected entities.

### E. Unattended scheduled + event-triggered sync (G1, FR1)

R12. **Scheduled sync, no client open.** Each healthy connection is pulled from IBKR on a schedule
(default: hourly on trading days **plus** a guaranteed pull just after **00:00 Australia/Sydney** to
freeze the daily snapshot) with no iOS app open. The `SydneyMidnightBoundary` math runs server-side
unchanged. Prices/quotes + the snapshot refresh on the same schedule.

R13. **Event-triggered sync.** An on-demand trigger (a user "refresh now" request or an external
event) runs the **same** sync handler with `reason = .onDemand` — "scheduled" and "on-event" are two
invocation sources of one function, not two code paths (system-design §4.2/§9.2).

R14. **The sync handler is `runRefresh` verbatim.** Server sync wires the **server** collaborators
(server `TokenStore`, server `FactLogStore`, shared `QuoteService`, server notifier, server clock)
into the existing fully-injected `BackgroundRefreshController.runRefresh(...)` pipeline and calls it;
only the `#if canImport(BackgroundTasks)` device shell is dropped (review §4.1; system-design §9.1).
No sync logic is rewritten.

R15. **Concurrency guard + rate-limit throttle.** A per-connection job lock prevents overlapping
scheduled + on-demand pulls from double-pulling (replaces the device `resyncInFlight`, A5). A
per-connection last-pull throttle gates how often invocations actually hit IBKR, respecting the
per-token rate limit (IBKR error 1018) — replacing the device's 30s `rateLimitBackoff`. A throttled
on-demand request degrades to a "synced a moment ago — try again shortly" outcome, not an error.

R16. **Degrade-safe sync.** A failed pull leaves the prior fact set + last-good snapshot intact and
surfaces a **token-free** reason (extends D31); the connection moves to a "needs attention" state
(`ConnectionState.error`/`.degraded`), and the last-good data stays readable, honestly labeled
"last synced <time>" — never demo, never a fabricated zero.

### F. Secrets handling (NFR2, system-design §8.2)

R17. **Server-held, write-only credentials.** A connection's `{token, queryId}` is stored
**server-side only**, one secret per connection; it is **never** persisted on the device beyond the
auth session token, **never returned** by any read (write-only), and **never logged** or placed in
any error type, response body, fact, projection, or export (extends D31). The token is read only to
build the Flex request URL.

R18. **Least-privilege credential access.** Only the sync path may read a connection's credential;
the read/API path has **no** access to credentials (it never needs the token to serve projections).

### G. The read/write API contract (FR4, FR6; the thin-client seam)

R19. **Read API serves projections at any scope.** A read endpoint returns each projection
(positions+totals+weights, tax/CGT/income/EOY/Div775, cash, optimise, gaps, stats) at a requested
`Scope` (`all` / `entity:<id>` / `account:<id>`), tenant-scoped from the verified identity. Response
bodies are the **existing `Codable` projection `Output` types** encoded via `CanonicalJSON` — the
client decodes the **same Swift types** (`LedgerCore` package dependency) it renders today (FR4;
system-design §12).

R20. **Freshness on every read.** Every data read carries an accurate, server-sourced freshness stamp
(`lastPullAt` / "as of" timestamp); stale data (schedule slipped / connection failing) is labeled as
stale (age + amber semantics) and never presented as current (UC-FRESH-1).

R21. **Sync-health read surface (the inverted FSM).** The device "Sync now" button-machine inverts to
a read-only `SyncHealth` view — `{lastPullAt, lastPullStatus, nextScheduledAt, factCount, digest,
staleness}` per connection — and one server-owned `ConnectionState`
`{notConfigured | active | degraded | error(reason)}` (system-design §6.3). The client *observes*; it
no longer drives sync.

R22. **Write API round-trips through the server.** Small writes — add/remove a connection, set
account→entity assignments + `TaxKind`, manage alert rules, resolve a cost-basis gap
(`costBasisFill`), trigger an on-demand sync, register an APNs device — go to the server, which
persists them and recomputes affected projections so the result reflects on **every** device of the
tenant (UC-READ-1, UC-CONN-6, UC-PUSH-1). A config/entity edit invalidates the tenant's cached
projections (`dropAll()` semantics; config is deliberately not in the fact digest, review §1.4).

### H. R8 RE-LOCK — the overlay firewall holds across the lift (binding)

R23. **R8 re-lock (binding).** Moving compute to the server moves **zero tax cell**. Display/cash/
daily-P/L/snapshot/overlay/quote inputs still never enter the fact log, the lot ledgers, or any
tax/CGT/income/FXGain/EOY/optimise/Div775 input — exactly as on device (architecture.md §6, R8/D67).
The **only** intentional tax mover remains `Div775Engine` (D74/D84). Proven NON-VACUOUSLY by a
mutation-checked byte-identical-tax test (A4 below + the device's existing `DailyPLR8RelockTests` run
server-side).

### I. Remote push (G5, FR5)

R24. **Server-evaluated alerts, push-delivered.** Alert rules persist per tenant and are evaluated
server-side on every sync via the existing `AlertFiringCoordinator` (armed ⇄ fired, state threaded
value-in/value-out); a firing delivers a remote push to the tenant's registered devices through the
existing `FiredAlert` seam (the deferred D46 APNs path), with the de-dupe/re-arm semantics preserved
(a still-crossed rule does not spam). Permission-denied degrades to today's in-app banner behaviour.

---

## Acceptance (A#) — offline-testable; this is what tests assert

> Each `A#` is a **provable fact**: a test asserts it with no human judgement. Every `A#` is proven
> **offline** the way the existing wave harnesses are (`Div775AcceptanceTests`/`DailyPLR8RelockTests`
> over the real on-disk `.flex-local` bytes + a **fake transport** / canned snapshot JSON, never live
> network — D3/D45), RED-first + mutation-checked (D45), against an **independent oracle** where a
> value is asserted (D75 — never the projection's own sum). Each names its `R#`(s) and `UC`(s).

A1. **Seam swap is behaviour-neutral** — *Given* the on-device build refactored so `LedgerStore`
reaches the log only through `FactLogStore` (concrete file store injected), *when* the full A2 tax
oracle + the A6 rebuildability suite run, *then* every test passes **unchanged** and the
`TaxProjection` canonical digest equals the pre-refactor golden `7f6784e1…` by exact string equality.
A mutation that lets `LedgerStore` bypass the protocol and touch the concrete store FAILs the seam
test. — *(proves R1, R3; serves UC-CONN-1, UC-READ-1)*

A2. **Server replay == device replay (THE wave bar)** — *Given* the real `.flex-local` fact set + the
committed `ProjectionConfig`, *when* the **server** `FactLogStore` implementation feeds
`ProjectionEngine` and the **device** file store feeds the same engine, *then* the full
`TaxProjection` (EOY + CGT disposals + income + forex + `forexRealisations`) serialised via
`CanonicalJSON` is **byte-identical** across the two stores (exact string equality; a 1¢ or 1-byte
delta FAILs). The FY26 Div775 line equals the committed real-bytes oracle (**−$1.76 AUD**, D84) on
the server path. — *(proves R4, R5, R7; serves UC-READ-1, UC-FRESH-1)*

A3. **Idempotent ingest, no cursor** — *Given* a server store seeded by ingesting the real statement
once (factCount = N), *when* the **same** statement bytes are ingested again (a re-pull / re-seed),
*then* `append` returns 0 newly-persisted and factCount is still N; the canonical digest is unchanged.
A second, **different** statement with overlapping rows adds only the genuinely-new FactIDs. — *(proves
R7, R8; serves UC-CONN-7, UC-FRESH-2)*

A4. **R8 byte-identical tax across the lift (the re-lock, non-vacuous)** — *Given* the real
`.flex-local` bytes run through the **server** path, *when* the full `TaxProjection` (canonical JSON)
is assembled in two worlds — `{no daily-P/L / no cash / no snapshot}` vs `{daily-P/L + cash + two
different non-degenerate snapshots present}` — *then* the tax bytes are **identical** across all worlds
while MV/daily-P/L move. **Negative control:** a deliberate leak of `snapshot.mark` / `currentFX` /
`netCash` into a tax input turns the test RED. — *(proves R23, R5; serves UC-READ-1)*

A5. **Tenant isolation** — *Given* two tenants T1 and T2 each seeded with a distinct fact set, *when*
a request authenticated as T1 reads/writes any resource (facts, projections, connection, secret,
snapshot, alert), *then* it returns **only** T1's data and **cannot** read or mutate any T2 resource;
a request that supplies a forged/other `tenantId` in the body is ignored — the isolation key is the
**verified** identity. A test asserting a cross-tenant read returns T1-only (T2 byte-count of leaked
data == 0). — *(proves R9; serves UC-ACCT-2, UC-CONN-3)*

A6. **N connections, isolated removal** — *Given* a tenant with two connections C1, C2 each ingested,
*when* C2 is removed, *then* C2's credential is deleted and C2 stops syncing, **and** C1's credential,
facts, and schedule are unchanged (C1 factCount before == after; C1 secret still present); the
consolidated `Scope.all` portfolio drops exactly C2's contribution. — *(proves R10; serves UC-CONN-2,
UC-CONN-5)*

A7. **Safe entity fallback** — *Given* a discovered account with no `EntityDef`/assignment, *when* the
tax recompute runs, *then* that account's disposals use a **0%** CGT discount (never 50%) and the
account is flagged "needs assignment"; *when* the user assigns it to an Individual entity, *then* the
recompute applies the 50% discount and the flag clears. A mutation that defaults the unassigned
account to 50% FAILs. — *(proves R11; serves UC-CONN-6)*

A8. **Unattended sync wiring (offline)** — *Given* the server sync handler wired with the server
collaborators (server `TokenStore` returning a canned credential, fake `FlexTransport` returning the
real recorded statement bytes, server clock fixed to a chosen instant), *when* the handler is invoked
with `reason = .scheduled` and again with `reason = .onDemand`, *then* both invoke the **same**
`runRefresh` body and produce the **same** ingest result for the same bytes (one shared code path);
the chosen instant flows through `SydneyMidnightBoundary` to a deterministic snapshot key. — *(proves
R12, R13, R14, R6; serves UC-FRESH-1, UC-FRESH-2, UC-FRESH-3)*

A9. **Job lock + throttle** — *Given* a connection with an in-flight pull (job lock held), *when* a
second concurrent invocation fires, *then* it does **not** start a second IBKR pull (lock denies it),
and the fact set after one logical pull equals the single-pull result (no double-ingest — guaranteed
anyway by A3's FactID dedup, but the lock prevents the wasted call). *Given* a last-pull timestamp
within the throttle window, *when* an on-demand refresh is requested, *then* it returns a
"try again shortly" outcome and makes **no** IBKR call. — *(proves R15; serves UC-FRESH-2)*

A10. **Degrade-safe failed pull** — *Given* a seeded connection, *when* the next scheduled pull fails
(fake transport returns an IBKR error / network failure), *then* the prior fact set + last-good
snapshot are unchanged, `ConnectionState` is `.error`/`.degraded` with a **token-free** reason, and a
subsequent read still returns the last-good projections with an honest stale stamp. — *(proves R16,
R20; serves UC-CONN-4, UC-READ-2)*

A11. **Token never leaks** — *Given* a full sync + read cycle, *when* every produced artifact is
inspected — logs, `FlexServiceError` (and any error type), every API response body, the fact JSONL,
every projection output, and every export — *then* the IBKR token string appears in **none** of them
(grep-clean, extends D31). *Given* a read-path request, *then* the read/API role has **no** path to a
credential secret. — *(proves R17, R18; serves UC-CONN-1, UC-ACCT-4)*

A12. **API payloads decode to the same core types** — *Given* a read endpoint's response for a known
fact set, *when* the bytes are decoded with the **`LedgerCore` `Codable` `Output` types** the client
renders, *then* decoding succeeds and the decoded `TaxProjection`/`PositionsProjection`/`CashProjection`
equals the directly-computed projection (round-trip equality); the response is `CanonicalJSON`
(sorted-keys, exact `Decimal`). — *(proves R19; serves UC-READ-1, UC-READ-3)*

A13. **Freshness + SyncHealth fields present and accurate** — *Given* a connection last pulled at a
known instant, *when* `SyncHealth` is read, *then* `lastPullAt` equals that instant, `factCount` +
`digest` equal the store's, `nextScheduledAt` is in the future, and `staleness` is computed from
(asOf − lastPullAt); a connection past its stale threshold reports `staleness = stale`. — *(proves
R20, R21; serves UC-FRESH-1, UC-FRESH-3, UC-CONN-3)*

A14. **Write round-trip recomputes** — *Given* a tenant projection cached at digest D, *when* a config
write (e.g. set an entity's `Div775Election` to non-`.standard`, or set parcel policy, or resolve a
gap via a `costBasisFill`) is applied through the write API, *then* the affected projections are
invalidated and recomputed, the new result differs from the cached one **iff** the change is
economically material (e.g. the gap resolution moves CGT; the Div775 election toggle changes the
Div775 line per A19 of domain-core.md), and a second device reading after the write sees the new
result. — *(proves R22; serves UC-READ-1, UC-CONN-6, UC-PUSH-1)*

A15. **Server-evaluated alert fires once** — *Given* an armed price-alert rule and a snapshot/quote
that crosses the threshold delivered to the server sync, *when* the sync runs, *then* the
`AlertFiringCoordinator` transitions armed→fired and emits exactly one `FiredAlert` to the push seam;
*when* the next sync runs with the rule **still** crossed (not reset), *then* **no** second
`FiredAlert` is emitted (re-arm/de-dupe holds). The push path is asserted against a fake notifier
capturing emitted `FiredAlert`s — no live APNs. — *(proves R24, R6; serves UC-PUSH-1, UC-PUSH-2)*

A16. **Append-only + exportable** — *Given* a seeded tenant log, *when* the store API is exercised,
*then* there is **no** update/delete-fact operation (the protocol exposes only `append`/`allFacts`/
`digest`), `allFacts()` returns canonical `(eventDate, stableSourceID)` order, and a per-tenant export
of the canonical bytes round-trips to the same `digest()`. — *(proves R8, R1; serves UC-ACCT-4,
UC-READ-3)*

> **Tracing checklist (reviewers enforce):**
> - [ ] Every `R#` (R1–R24) is proven by ≥1 `A#`.
> - [ ] Every `A#` names its `R#`(s) and ≥1 `UC` (or is an explicit named invariant — R8/D8/D31).
> - [ ] Every `A#` is assertable with **no human judging** — exact byte/value equality, stated
>       tolerance (**zero**), RED-first + mutation-checked (D45), independent oracle (D75).
> - [ ] No business vision (single-vs-multi-tenant *decision*) or tap-by-tap user-flow prose leaked
>       in — those live in `backend-migration-review.md` / `use-cases/backlog-backend.md`.

---

## Out of scope / non-goals

- **Re-implementing any tax/domain logic** in another language — the Swift core is the asset; this
  spec forbids it (R4), it does not re-spec it (the W1 `requirements.md` R9–R17 still own the math).
- **The supersede/revision path** for IBKR restatements (`revision`/`supersedes` on `Fact`,
  system-design §5.1) — additive backlog item, **not** on this lift's critical path; gated by OD4.
  This lift keeps the existing `(stableSourceID, kind)` identity (R8 note).
- **Billing / subscriptions / App-Store launch / a web frontend / real-time streaming quotes** — the
  frozen W5/W7 product scope, explicitly deferred (system-design §1.3).
- **The choice of concrete AWS services** (DynamoDB vs Aurora; Cognito vs device-key; EventBridge
  Scheduler) — those are `system-design.md` decisions behind these seams, swappable without touching
  this contract. A test asserts a *capability* (idempotent put, tenant isolation, scheduled invoke),
  not a service.
- **Multi-tenant account journey UI** beyond its server contract — the `[MT]` sign-up/delete screens
  (UC-ACCT-1/3/4) are designer/UI-engineer surfaces; this spec owns only their server behaviour
  (isolation, write-only credentials, delete-removes-everything).

## Open questions

These block the `A#` they touch; do not let a developer guess (mirror `use-cases/backlog-backend.md`
§8 + `system-design.md` §16):

- **OD1 — single-tenant vs multi-tenant (the headline fork).** Determines whether the full identity
  journey (UC-ACCT-1/3/4) and its `A5` multi-tenant assertions are in this wave or degraded to a
  device-pinned single-tenant key (`tenantCount == 1`). The R9 isolation invariant holds either way
  (single-tenant is the `tenantCount == 1` cut); only the **auth** surface degrades. **Blocks the
  scope of A5.**
- **OD3 — store engine** (DynamoDB single-table vs Aurora Postgres). Swappable behind `FactLogStore`
  (R1); pick the default before A2/A3 fixtures are written so the conformance test targets the chosen
  implementation. Does **not** change any `A#` text (capability-level).
- **OD-CONN5 — remove-connection data retention.** On `UC-CONN-5` removal, are the connection's
  already-ingested facts retained read-only or deleted with the credential? **Blocks the exact
  post-condition of A6** (does C2's history survive removal?). Tax-history implications — boss call.
- **OD4 — supersede/revision now or backlog** (system-design §16 OD4). If opened, adds an `R#`/`A#`
  for restatement handling; currently **out of scope** (above).
- **OD6 — preflight hard gates** (none verified; system-design §8/§16): AWS creds usable from this
  environment; a Swift-on-Linux/Lambda container build proven; the IBKR Flex token re-issued for
  unattended server use with the rate-limit cadence (error 1018) confirmed; the APNs key +
  `aps-environment` entitlement (absent today, D46) — **A15's live-push leg and A8's deploy leg are
  blocked until these pass.** Offline acceptance (A1–A16 as written, fake transport) is **not** gated
  on OD6.

## Alias map (old → canonical)

> W6 files (`milestones/W6.md`, `tasks/W6-M1.md`, the W6 job-descriptions) cite a few IDs whose **canonical home is `domain-core.md`**. This table tells a fixer how to translate each citation. No `R#`/`A#` defined in THIS file was renamed (the flat W6 namespace is stable: `R1–R24`, `A1–A16`); the entries below are the **cross-file** resolutions.

| Old / ambiguous citation (as written in W6 files) | Canonical target | Where it lives | Note |
|---|---|---|---|
| **"A6" (byte-identical replay)** | **A3** (`byte-identical-replay`) | `domain-core.md` | The on-device replay property re-run server-side; this file proves the lift preserves it via **A1**+**A2**. The W6-local **A6** (`N-connections-isolated-removal`) is a DIFFERENT anchor — resolve by concept, not number. |
| **"R8" (display-never-moves-tax)** | **R23** (`domain-core.md`) | `domain-core.md` | Re-locked across the lift by **this file's R23** + proven by **A4**. The W6-local **R8** is `append-only durable log` — resolve by concept. |
| **"A2" (tax oracle / golden digest)** | **A1** (`tax-to-the-cent-vs-oracle`) on device; **A2** (`server-tax-byte-identical-to-device`) on server | `domain-core.md` A1 / this file A2 | W6 "byte-identical to the on-device build" = this file's **A2** asserting equality against `domain-core.md`'s **A1** golden. |
| snapshot-freeze (overlay) | **A28** | `domain-core.md` | snapshot-freeze on the same Sydney day; this file's **A8** drives the same `SydneyMidnightBoundary` server-side. |
| snapshot reconstruction (overlay) | **A27** | `domain-core.md` | snapshot reconstruction (overlay provider) — lifted onto the scheduled server path (this file's **A8**/R12). |
| on-device Sync-now re-pull | **A29** (→ this file **A11**) | `domain-core.md` / this file | on-device Sync-now re-pull; the server lift is this file's **A11** (`token-never-leaks`) + **A8** (`unattended-scheduled-sync`). |
| Div 775 forex realisation | **A19** (`div775-vs-independent-oracle`) | `domain-core.md` | the Div 775 line referenced by this file's **A2**/A14 ("Div775 line per A19 of domain-core.md") resolves to `domain-core.md` A19. |

> The W6-local flat anchors (`A1` seam-swap, `A2` server-replay, `A3` idempotent, `A4` server-R8-relock, `A5` isolation, `A6` N-connections, `A7` safe-fallback, `A8` unattended-sync, `A9` lock+throttle, `A10` degrade-safe, `A11` token-never-leaks, `A12` API round-trip, `A13` freshness, `A14` write-recompute, `A15` alert-fires-once, `A16` append-only) are **never dotted** and never collide *within* this file. Cross-file collisions with `domain-core.md` are resolved by the concept anchor tables, never by raw number.

## Source

(to be implemented) — W6 build per `system-design.md` §15 milestones (pre-refactor seam → infra →
seed+verify → cut sync → thin client → push → acceptance). The developer's tests and QA's acceptance
tests both point back to the `A#` ids above; the wave's exit gate is **A2** (server tax bytes ==
device golden) plus the boss's end-to-end-on-real-data leg (review §7-M7).
