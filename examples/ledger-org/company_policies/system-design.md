# Ledger System Design — W6 Backend Lift (AWS, Path A single-tenant first)

> Owned by the **Tech Lead**. The **"can we build & run it?"** document for the next deployment phase. Reviewed by `ledger-tech-lead-reviewer` for delivery soundness, and consulted with the **Architect** (`ledger-architect`) for design-fit — the architecture↔system-design seam is a consultation edge, not a merge.
>
> **Distinct from `architecture.md`.** Architecture answers *"is it right?"* — the event-sourcing invariants, the seams, the deterministic byte-identical-replay property (`specs/domain-core.md` A3), the current-state shape. This answers *"can we operate it?"* — the AWS topology, identity, API surface, iOS-client changes, and the staged migration we have to stand up and run. Where this design leans on an invariant (Decimal-only money, Foundation-only core, FactID idempotency, byte-identical replay) it **points to `architecture.md` / `data-model.md` / `specs/`** rather than restating it. Where it needs a *new* invariant (the `FactLogStore` seam, the `revision`/`supersedes` restatement path, the conservative 0% CGT fallback) that is a **consultation back to the Architect**, logged in §10.
>
> Version-stamped, scoped to W6, supersedes the W5 design note. Open decisions that are irreversible / external / strategic / cost-incurring are **flagged for the founder** (`decision-escalation.md`) — staged ready-to-flip, never auto-decided.

**Phase:** W6 — Backend Lift ("a lift, not a rebuild": move `LedgerCore` verbatim into AWS Lambda; move storage/secrets/schedule/notify/transport behind the seams; thin API in front). **Path A (single-tenant, unattended)** deploys first as a strict configuration subset of the multi-tenant-capable target.
**Backing specs:** `specs/backend-lift.md` is the W6 contract — sync (§E R12–R16/A8–A10: scheduled+triggered pull, idempotent ingest), the fact-store seam (§A R1 + §C R7–R8 / A1, A3, A16: `FactLogStore` seam, append-only, FactID idempotency; the byte-identical-replay property `FactStore-seam-preserves-replay` A1), projections/read (§G R19/A12: versioned+digest-stamped rebuild served at any `Scope`), server tax (§B R4–R5 / A2 `server-tax-byte-identical-to-device`: CGT/franking/FX/Div775 byte-identical to the on-device oracle), and the read API (§G R19–R21 / A12–A13: read API contract, identity-scoped). The on-device math it preserves verbatim lives in `specs/domain-core.md` (tax-to-the-cent A1, byte-identical-replay A3). Domain shapes (Fact/FactKind/LotLedgerState/Scope) are the source of truth in `data-model.md`.
**Status:** in-review

---

## 1. Purpose & goals

W6 moves Ledger's sync and compute off the single iOS device onto an always-on AWS backend so that (a) IBKR Flex pull + price/snapshot refresh runs **unattended on a schedule and on demand** with no phone open — the literal pilot pain; (b) the data model is **multi-tenant-capable** (Path B later) while deploying single-tenant now; and (c) the iOS app becomes a **thin client** over a server API. The deterministic Foundation-only core (`Sources/LedgerCore`) is preserved **verbatim**; every device coupling already has a named protocol seam pointing at its server replacement (the W5 swap-points). Now, because the lift unlocks scheduled sync against the existing seams with low blast-radius.

**Goals (this phase delivers):**
- **Unattended scheduled sync** — IBKR Flex pull + Sydney-midnight snapshot refresh run server-side on an EventBridge cron and can be triggered on-event, independent of any client (FR1).
- **Byte-identical correctness** — server-computed CGT / franking / FX / Div775 / EOY numbers are **byte-identical** to the current on-device build for the same facts + config (the byte-identical-replay property — `domain-core.md` A3 — holds across the lift, proven by `backend-lift.md` A2; verified vs the committed golden).
- **Multi-tenant-capable data model** — per-tenant isolation, per-connection/account partitioning, N entities/accounts (closes the D83 single-account wiring limit) — deployed single-tenant at Path A.
- **Core reused unchanged** — zero behavioural change to `LedgerCore`; the lift swaps storage / secrets / schedule / notify / transport only.
- **Remote alerts** — fire `FiredAlert` push notifications server-side via SNS→APNs (the deferred D46 path).
- **Operable & cheap at N=1** — scale-to-zero economics for a single user; a few dollars/month.

**Non-goals (explicitly out of scope — named so reviewers don't expect them):**
- **No tax-engine re-implementation** in another language — the Swift core *is* the asset; re-deriving it would re-incur the entire on-device verification.
- **No web frontend** — the iOS client stays the only client this phase; the API is client-agnostic so a web client is a later option.
- **No billing / subscriptions / App-Store launch** — the frozen W5 scope absorbs into Path B (W7) if multi-tenant goes public.
- **No real-time / streaming quotes** — delayed quotes + the daily Sydney-midnight snapshot is the contract, unchanged (R8).
- **No public/third-party API** — the API is client↔server only this phase.

---

## 2. Target architecture (component diagram)

> The runnable AWS topology after W6 ships. Two Lambdas, **one Swift binary** embedding `LedgerCore`; they differ only by handler entrypoint and IAM role (no logic fork). Everything marked NEW lands on a pre-placed seam.

```
                  ┌──────────────────────── EventBridge ────────────────────────┐  ← NEW
                  │  Scheduler (cron): per-connection pull + 00:00 Australia/     │
                  │  Sydney snapshot freeze                                       │
                  │  Rule (event): on-demand "Sync now" / webhook                 │
                  └───────────────────────────────┬──────────────────────────────┘
                                                  │ invoke {tenant, connectionId, reason}
   Secrets Manager  ◀──token (read)──┐            ▼
   (per connection)        NEW       │   ┌────────────────────────────────┐  ← NEW (handler shell)
   ledger/<tenant>/<conn>            └───│  Lambda: SyncWorker (Swift)     │
                                         │  LedgerCore VERBATIM:           │──append (idempotent)──▶  Fact store  ← NEW
   Shared quote cache  ◀──prices───────│   runRefresh(...) + ingestReal   │                          (DynamoDB single-table:
   (DynamoDB, short TTL)     NEW        └───────────────┬────────────────┘                           PK T#tenant#C#conn, SK F#…#factId)
                                                        │ FiredAlert
                                                        ▼
                                                 SNS  ──▶  APNs  ──▶  iOS device (push)   ← NEW (lights up D46)

   iOS thin client ──HTTPS (Cognito JWT)──▶ API Gateway (HTTP API) ──▶ Lambda: API (Swift, LedgerCore)  ← API edge NEW
     observes projections / sync-health / reports          ├─ read:  replay + cache → projections @ Scope
     triggers sync; manages connections/config/alerts       └─ write: connections, config, gaps, rules, trigger
                                                                       │
                                                       Projection cache (StampedProjection in Dynamo)  ◀── version + digest stamped
```

| component | new/existing | responsibility | runs where |
|-----------|--------------|----------------|-----------|
| iOS app (thin client) | existing (re-thinned) | observe server projections / sync-health; trigger sync; manage connections/config/alerts; receive APNs | iPhone |
| API Gateway (HTTP API) | NEW | TLS edge; validate Cognito JWT; route to API Lambda | AWS managed |
| Lambda: `API` | NEW (shell) | read path (replay → projection @ Scope, cache) + small writes; **no Secrets Manager access** | AWS Lambda (container image) |
| Lambda: `SyncWorker` | NEW (shell) | write path: pull → ingest → fold → fire alerts; calls `runRefresh` verbatim | AWS Lambda (container image) |
| `LedgerCore` | existing (verbatim) | Reconciler, Fact model, all tax engines, LotLedger, projections | inside both Lambdas |
| EventBridge Scheduler + Rule | NEW | cron heartbeat + Sydney-midnight pull; on-demand invoke | AWS managed |
| Fact store (DynamoDB single-table) | NEW | per-tenant append-only fact log + materialised state + projection cache | AWS managed |
| Secrets Manager | NEW | per-connection IBKR `{token, queryId}` | AWS managed |
| Shared quote cache (DynamoDB) | NEW | one quote/snapshot fetch per symbol serves all tenants | AWS managed |
| SNS → APNs | NEW | deliver `FiredAlert` as remote push | AWS managed |
| S3 (statements/exports) | NEW | raw pulled XML (audit) + generated PDF/CSV; SSE-KMS, lifecycle-expired | AWS managed |
| Cognito user pool | NEW | identity; JWT carries verified `tenantId` | AWS managed |

**Trust boundaries.** (1) **Client↔API:** TLS + Cognito JWT at API Gateway; the Lambda derives the storage prefix `T#<tenant>` **only** from the verified `tenantId` claim, never a client-supplied value (this is the multi-tenant isolation boundary, FR8). (2) **Compute↔secrets:** only `SyncWorker`'s IAM role can read Secrets Manager; the token never enters a log, a response, or `API`'s role. (3) **Tenant↔tenant:** every Dynamo item is prefixed by `T#<tenant>`; a query is physically scoped to one partition prefix.

---

## 3. Data & storage

> What is persisted *for operations*. The domain shapes (Fact, FactKind, LotLedgerState, Scope) live in `data-model.md` — linked, not restated.

- **System of record:** the **per-tenant append-only fact log** in DynamoDB. Facts are the only truth; every projection is a derived, disposable, rebuildable fold (`architecture.md` event-sourcing invariant; shapes in `data-model.md`).
- **Storage engine & why — DynamoDB single-table (recommended, internal/reversible → log in `decisions.md`).** Access patterns are key-based and tenant-scoped; on-demand billing gives scale-to-zero (NFR4); **conditional puts give FactID idempotency and the daily-snapshot "freeze" for free**. One table, item types:

  | Item | PK | SK | Notes |
  |------|----|----|-------|
  | Fact | `T#<tenant>#C#<conn>` | `F#<eventDateKey>#<factId>` | Append-only; **`PutItem` with `attribute_not_exists(SK)`** ⇒ FactID idempotency natively (replaces the in-memory `Set<FactID>`). `eventDateKey` in the SK preserves the deterministic `(eventDate, stableSourceID)` query order. |
  | LotLedgerState | `T#<tenant>` | `S#<entity>#<account>#<instrument>` | Materialised ticker state (parcels + disposals); range-query by `S#<entity>#<account>` prefix for account/portfolio roll-ups. Cache; rebuildable. |
  | StampedProjection | `T#<tenant>` | `P#<name>#<scope>` | Versioned + digest-stamped projection output (positions/tax/cash/income/optimise/gaps/stats); reused while `version`+`digest` match (the existing `ProjectionStore` contract). |
  | Connection | `T#<tenant>` | `CONN#<conn>` | `{accountIds, queryId ref, schedule, lastPullAt, lastStatus, jobLock}`. Secret stored separately (§4). |
  | EntityDirectory | `T#<tenant>` | `DIR` | account→entity assignments + `TaxKind` per entity (the persisted `ProjectionConfig`; closes D83). |
  | AlertRule / AlertState | `T#<tenant>` | `ALERT#<id>` / `ALERTSTATE` | rules + the armed/fired set threaded into `AlertFiringCoordinator`. |
  | DailySnapshot | `T#<tenant>#C#<conn>` | `SNAP#<sydneyDay>` | `saveIfAbsent` = conditional put ⇒ frozen-through-the-day (the `SydneyMidnightBoundary` snapshot key driven server-side, `backend-lift.md` A8) for free. |

  Alternative kept open: **Aurora Serverless v2 Postgres** (one append-only `facts` table unique on `(tenant, conn, factId)`) if richer ad-hoc reporting is wanted later — swappable behind the `FactLogStore` seam without touching the core (OD3, §9).
- **The `FactLogStore` seam (critical pre-refactor).** Today `LedgerStore` holds a **concrete** `FactStore` (the durable log has no interface to swap — review §1.5). Introduce one protocol, an in-app refactor with **no behaviour change** (R8-locked, byte-identical-replay determinism tests `domain-core.md` A3 stay green; seam neutrality is `backend-lift.md` A1):
  ```swift
  protocol FactLogStore {            // server: DynamoFactLogStore; device: file-backed FactStore
      func append(_ facts: [Fact]) async throws -> Int   // # newly persisted (idempotent)
      func allFacts() async throws -> [Fact]             // canonical (eventDate, stableSourceID) order
      func digest() async throws -> String               // SHA-256 over the sorted canonical log
  }
  ```
  This is the single most important code change enabling the lift.
- **On-wire / at-rest format:** projection responses are the existing `Codable` `Output` types encoded via `CanonicalJSON` (sorted keys, exact `Decimal` — the determinism invariant in `architecture.md`); the client decodes the **same Swift types** it renders today.
- **Durability / backup / retention:** Dynamo PITR on the table; the append-only fact log + SHA-256 content digest remain the exportable source of truth (NFR6). Raw pulled statement XML and exports → S3 (`s3://ledger-<env>/<tenant>/<conn>/statements/<pulledAt>.xml`), SSE-KMS, lifecycle-expired after a retention window; facts (the canonical truth) live in Dynamo, never expired.
- **Tenancy / isolation:** every item is prefixed `T#<tenant>`; a query is physically scoped to that prefix derived from the verified Cognito claim. Path A deploys with `tenantCount == 1` (a fixed `tenantId` constant) — the same code path, isolation degraded to a device-pinned key.

---

## 4. Identity, auth & secrets

> Who may do what, and how the system proves it. Identity-provider choice and APNs are **founder-flagged** (§9) — staged, not decided.

- **Identity model:** Amazon **Cognito** user pool; each user = one tenant; the JWT carries `tenantId`. **Phase-1 single-tenant cut:** Cognito with one user, or a device-pinned API key + a fixed `tenantId` constant — same code path, auth degraded. This is the "get my backend off my phone" deployment.
- **AuthN:** Cognito-issued JWT, short-lived access token + refresh; API Gateway's JWT authorizer validates issuer + signature before any Lambda runs. The client obtains the token at sign-in and presents it as a bearer header.
- **AuthZ:** default-deny. The Lambda derives `T#<tenant>` from the **verified** claim and scopes every storage op to it; no cross-tenant op is expressible. `SyncWorker` may read secrets; `API` may not (IAM-enforced).
- **Secrets handling:** IBKR Flex `{token, queryId}` → **AWS Secrets Manager**, one secret per connection `ledger/<tenant>/<conn>`, behind `SecretsManagerTokenStore: TokenStore` (the documented W5 swap-point — nothing upstream changes). Supports **multiple connections per tenant** (fixes the single-item Keychain upsert blocker, D83 T1). The token is read **only** to build the Flex URL; it is **never** logged, never returned in a response, and error types carry no token (already true in `FlexServiceError`; extends D31). Rotation is a Secrets Manager update, no code change.
- **Transport security:** TLS everywhere (API Gateway, AWS SDK to Dynamo/SNS/Secrets, IBKR/Yahoo egress). Least-privilege IAM per Lambda (`SyncWorker`: read its tenant's secrets + RW Dynamo + write SNS + RW quote cache + RW statement S3; `API`: RW Dynamo + read SNS topic ARNs, **no Secrets Manager**).

> **Founder flags here (see §9):** the identity provider (Cognito vs device-key-forever) and APNs (Apple push key + `aps-environment` entitlement, deliberately absent today per D46) are escalation-class — a default is proposed and staged behind config.

---

## 5. Sync subsystem

> The heart of the lift. The device sync FSMs **invert**: the server pulls on cron/event; the client only *observes* a read-only `SyncHealth` view. Idempotency is content-based, so every step is safely re-runnable.

- **Sync model:** **server-driven pull**, not client push. EventBridge **Scheduler** (cron) and an EventBridge **rule / API route** both invoke `SyncWorker` with `{tenant, connectionId, reason}` — "scheduled" and "on-event" are two invocation sources of **one** handler (FR1). The handler is a thin shell wiring the server collaborators (`SecretsManagerTokenStore`, `DynamoFactLogStore`, shared `QuoteService`, `SNSAlertNotifier`, server clock) and calling `BackgroundRefreshController.runRefresh(...)` — already the fully-injected, `LedgerCore`-style headless pipeline (review §6); only the `#if canImport(BackgroundTasks)` device shell drops. Cadence: hourly on ASX/US trading days + a guaranteed pull just after **00:00 Australia/Sydney** to freeze the daily snapshot (the `SydneyMidnightBoundary` math runs server-side, unchanged).
- **Conflict resolution:** there is no last-writer-wins conflict — the log is **append-only** and ingest is **idempotent by FactID** (content hash of `(stableSourceID, kind)`). Re-pulling a statement is a no-op (FR2); there is **no since-cursor** (the IBKR `queryId` defines the window; FactID dedup absorbs overlap). The one real conflict — IBKR **restating** a row under the same id+kind with a corrected amount — is handled by the new `revision`/`supersedes` path (Architect consultation, §10): a higher-revision fact supersedes its predecessor in the fold; the predecessor stays in the log, marked superseded (additive, version-bumping, replay stays deterministic).
- **Ordering & identity:** facts fold in canonical `(eventDate, stableSourceID)` order (`FactLog`); identity is the seed-free FNV-1a FactID (`data-model.md`). The Dynamo SK `F#<eventDateKey>#<factId>` preserves that order on query.
- **Offline behavior (server side):** a failed pull degrades gracefully — the last-good materialised state and last frozen snapshot are still served (NFR3); the append-only model means no single sync failure loses data. Client-offline behavior is in §7.
- **Failure & resume:** a per-connection **`jobLock`** (conditional update on the Connection item, with TTL) prevents overlapping scheduled + on-demand invocations from double-pulling (replaces the device `resyncInFlight`). A per-connection **`lastPullAt` throttle** in Dynamo gates how often EventBridge invocations actually hit IBKR, respecting the per-token rate limit (error 1018; replaces the device's 30s `rateLimitBackoff`). The Flex failover/polling (SendRequest→ReferenceCode→GetStatement, `ndcdyn`→`gdcdyn`) is in-core, unchanged. A crash mid-sync leaves a consistent state because each fact is an idempotent conditional put — re-invoking replays cleanly.

**Acceptance hooks:** byte-identical replay (drop projections + replay ⇒ byte-identical, server-side, against the committed golden `.flex-local`) — the on-device `byte-identical-replay` property (`domain-core.md` A3) re-run through the lifted seam, proven by `backend-lift.md` A1/A2; idempotent ingest (re-pull ⇒ `append` returns 0 newly-persisted, digest unchanged — `backend-lift.md` A3); snapshot freeze (second pull same Sydney day ⇒ snapshot unchanged — the `SydneyMidnightBoundary` math driven server-side by `backend-lift.md` A8). These trace to `specs/backend-lift.md` §E (sync, A8–A10) and §A/§C (the fact-store seam, A1/A3/A16).

---

## 6. API design

> The client↔server contract. Response bodies are the existing `Codable` projection `Output` types via `CanonicalJSON`, so the thin-client migration is mechanical. All paths tenant-scoped from the verified Cognito claim.

| method | path | purpose | auth | request → response (shape) | backing R#/A# (`[BL]` = `specs/backend-lift.md`) |
|--------|------|---------|------|----------------------------|---------------|
| GET | `/portfolio?scope=all\|entity:<id>\|account:<id>` | positions + totals + weights | tenant | `()` → `PositionsProjection` (rows/groups/totals) | R19, A12 [BL]; replay A2 [BL] |
| GET | `/tax?entity=<id>&fy=<fy>` | tax summary / CGT / income / EOY / Div775 | tenant | `()` → `TaxProjection` + `EOYSummary` | R5, A2 [BL] (server byte-identical) |
| GET | `/cash?scope=…` | cash balances per `(entity,account,currency)` | tenant | `()` → `CashProjection` | R19, A12 [BL] |
| GET | `/optimise?…` · `/gaps` · `/stats` | optimise / cost-basis gaps / portfolio stats | tenant | `()` → respective `Output` | R19, A12 [BL] |
| GET | `/sync/health?connection=<id>` | `{lastPullAt, lastPullStatus, nextScheduledAt, factCount, digest, staleness}` | tenant | `()` → `SyncHealth` | R21, A13 [BL] |
| GET | `/reports/<kind>?…` | PDF/CSV export | tenant | `()` → presigned S3 URL | R19, A12 [BL] |
| POST | `/connections` | store credential + Connection item | tenant | `{queryId, token}` → `{connectionId}` (token write-only, never returned) | R17, A11 [BL]; NFR2 |
| POST | `/sync` | trigger an on-demand `SyncWorker` run | tenant | `{connection}` → `202 {jobId}` | R13, A8 [BL] (FR1) |
| PUT | `/entities` | account→entity assignments + `TaxKind` | tenant | `EntityDirectory` → `204` | D83; R22, A14 [BL] |
| POST/DELETE | `/alerts` · `/alerts/<id>` | manage alert rules | tenant | `AlertRule` → `204` | R24, A15 [BL] |
| POST | `/gaps/<id>/resolve` | append a `costBasisFill` fact | tenant | `{costBasisFill payload}` → `204` | R22, A14 [BL] |
| POST | `/devices` | register APNs token for push | tenant | `{apnsToken}` → `204` | R24, A15 [BL] (D46) |

- **Versioning:** path-prefixed `/v1`; a v1 client survives a v2 server because the `CanonicalJSON` `Output` types are the shared `LedgerCore` package dependency — additive fields decode, the client ignores unknowns.
- **Errors:** a small JSON error envelope `{code, message}` with HTTP status; `5x`/throttle are retryable, `4xx` (auth, validation) are fatal. The IBKR rate-limit (1018) is surfaced as a `SyncHealth` `degraded` status, not a hard API error.
- **Idempotency:** writes that append facts (`POST /gaps/.../resolve`, ingest via `POST /sync`) are idempotent by FactID — replaying is a no-op. `POST /sync` is guarded by the per-connection `jobLock`.
- **Compatibility contract:** the client and server share the `LedgerCore` `Codable` types as a package dependency; the backward-compat window is "additive-only fields within a major path version."

> No **public / third-party** surface this phase (non-goal). Internal client↔server contracts are the Tech Lead's to decide and log; user-facing sign-in vocabulary is founder-flagged (§9).

---

## 7. Client changes

> The bridge the migration walks across. The app becomes a **thin client**; most deleted code is the parts that move server-side — a large but mechanical reduction.

- **New capabilities:** a Cognito auth/token store; an API client decoding the same `LedgerCore` `Output` types; APNs registration (`POST /devices`); a `SyncHealth` observer.
- **Changed flows:** the `ResyncControlVS` button machine (`ResyncResult {idle|updated|upToDate|error|noToken}`) is **replaced** by a read-only `SyncHealth` view (observe `lastPullAt`/`nextScheduledAt`/`status`) plus a "Sync now" that calls `POST /sync`. Alerts arrive via **APNs** instead of local notifications. Coordinate the affected use-cases (sync, alerts) with the UX manuals.
- **Local migration:** drops on-device ingest, `FactStore`, projection compute, `BGTaskScheduler`, and the Keychain token (the token now lives in Secrets Manager; the device authenticates to the API via Cognito). Keeps the SwiftUI views + the `LedgerCore` `Codable` types.
- **Graceful degradation:** with no network / expired token / server down, the app shows the last cached API responses read-only and a stale `SyncHealth` banner; sign-in refresh recovers. (Offline write-queue is Phase-2, not W6.)
- **Backward compat:** during rollout the on-device ingest/compute path stays behind a flag; the app can run old-local or server-backed until parity is verified, then the flag flips default to server.

---

## 8. Migration plan

> Today's on-device system → the AWS target, **reversible at every step**. The append-only, FactID-idempotent model makes every step re-runnable: re-seeding or re-pulling never corrupts state.

| step | action | reversible? | rollback | verify (observable) |
|------|--------|-------------|----------|---------------------|
| 1 | **Pre-refactor in-app** (no behaviour change, R8-locked): introduce `FactLogStore`/`TokenStore` server-ready seams; consolidate the ~8 scattered connection booleans into one `ConnectionState {notConfigured\|active\|degraded\|error}`; close the D83-T5 CGT-fallback hazard (conservative **0%**, never silent 50%). | yes | revert the branch | byte-identical-replay determinism tests (`domain-core.md` A3) + full suite stay green; seam stays behaviour-neutral (`backend-lift.md` A1) |
| 2 | **Stand up infra** (Terraform, no traffic): Dynamo single-table, Secrets Manager, EventBridge, SNS, S3, Cognito, API Gateway, the two Lambdas (container image). | yes | `terraform destroy` the stack | healthcheck green; empty-tenant API returns 200 |
| 3 | **Seed + verify (the NFR1 gate):** one-shot loader imports the on-device `real/facts.jsonl` into the tenant's Dynamo fact partition (FactID dedup ⇒ safe/repeatable). Replay server-side; **diff the tax digest against the on-device golden**. | yes | re-seed is idempotent; tear down on mismatch | server tax digest == on-device golden, **byte-identical** (`backend-lift.md` A2) |
| 4 | **Cut sync over:** enable the EventBridge schedule; the server becomes the source of truth for new pulls. | yes | disable the schedule; device sync resumes | a scheduled run appends new facts; `SyncHealth` advances |
| 5 | **Thin the client:** point the app at the API behind the flag; verify parity (server projections == on-device); flip the flag default to server. | yes | flip the flag back to on-device | per-screen parity check passes; tax cells match |
| 6 | **Enable push:** register APNs; route `FiredAlert` → SNS → device. | yes | stop the SNS publish; alerts fall back silent | a fired rule delivers a push to the device |

- **Data backfill:** one-time `facts.jsonl` → Dynamo loader; integrity proven by the **byte-identical tax-digest parity check** (step 3, `specs/backend-lift.md` A2 `server-tax-byte-identical-to-device`), the same gate that proves the lift.
- **Cutover strategy:** flag-gated client cutover (step 5) after the seed+verify gate passes; the schedule cutover (step 4) is a single EventBridge enable.
- **Rollback trigger:** any tax-cell or digest mismatch at the parity gate ⇒ roll back (flip the flag / disable the schedule) and re-investigate — a mismatch is never accepted.
- **Zero/low-downtime story:** the on-device app keeps working throughout; the server runs in parallel until parity, so there is no user-facing downtime window.

---

## 9. Risks & open decisions

> **Risks** = things that could go wrong, with a mitigation/detection. **Open decisions** = forks tagged decide-now (logged) or **FOUNDER** (escalated per `decision-escalation.md`: irreversible / external / strategic / costs money / locks future waves). FOUNDER items are **staged ready-to-flip** so the run is never blocked.

**Risks**
| risk | likelihood | impact | mitigation / detection |
|------|-----------|--------|------------------------|
| Server replay diverges from on-device (a cent off) | low | critical | the byte-identical tax-digest parity gate (step 3) blocks cutover; a **determinism canary** Lambda replays the golden `.flex-local` on a schedule and alarms on any digest drift (NFR1 in production) |
| Yahoo blocks/limits a fixed server IP (ToS-gray) | med | high | backoff + rotating egress; the `QuoteProvider`/`QuoteTransport` seam keeps a licensed feed a one-line swap (OD2); overlays are display-only so a stale quote never moves tax (R8) |
| IBKR per-token rate limit (error 1018) under cron cadence | med | med | per-connection `lastPullAt` throttle in Dynamo gates real IBKR hits; surface as `SyncHealth.degraded`, not data loss |
| Cold full-replay exceeds Lambda timeout as history grows | low | med | realistic single-user fact count (thousands) replays well within timeout; the materialised `LotLedgerState` cache lets reads skip the fold and aggregate stored ticker states (the optimisation target if needed) |
| Swift-on-Lambda container build not proven here | med | high | Preflight hard gate (OD6) — prove the Linux/Lambda container build before any infra spend; `PortabilityLockTests`/D30 already assert Foundation-only |
| Token leaks into a log/response | low | critical | token read only to build the Flex URL; never logged, never returned; `API` role has no Secrets Manager access; error types carry no token |
| IBKR restatement silently dropped as a duplicate | med | high | the `revision`/`supersedes` path (OD4 / §10) — a higher-revision fact supersedes in the fold; predecessor retained for audit |

**Open decisions**
| id | decision | recommendation (staged default) | owner | escalate? |
|----|----------|---------------------------------|-------|-----------|
| OD1 | Scope: multi-tenant-capable, Phase-1 single-tenant (vs single-tenant-forever, which drops Cognito for a device key) | **multi-tenant-capable, deploy single-tenant** | founder | **FOUNDER** — strategic / locks future waves |
| OD2 | Price feed: keep Yahoo (server-IP ToS risk) vs a licensed feed (ongoing cost) | **keep Yahoo behind the `QuoteProvider` seam**, licensed feed staged as a one-line swap | founder | **FOUNDER** — external surface + costs money |
| OD3 | Store engine: DynamoDB single-table vs Aurora Serverless Postgres (richer reporting later) | **DynamoDB single-table** (swappable behind `FactLogStore`) | tech-lead | decide-now → log in `decisions.md` |
| OD4 | Restatement handling: build the `revision`/`supersedes` path now vs backlog | **backlog** for a clean first lift; build before any unattended-restatement reliance | founder | **FOUNDER** — locks data-model future; Architect consult (§10) |
| OD5 | AWS account / region / budget (the `aba`/`smut-lambda` accounts exist) | **reuse an existing account, ap-southeast-2, a low budget ceiling** | founder | **FOUNDER** — costs money / account ownership |
| OD6 | Preflight hard gates (block any build): AWS creds usable here; Swift-on-Lambda container build proven; IBKR token re-issued for unattended server use + rate cadence confirmed; APNs key + `aps-environment` | **batch into the CEO founder pre-flight; do not start the build until all four are green** | founder | **FOUNDER** — external / credentials / costs money |

> Every **FOUNDER** row is staged behind an abstraction/flag (a one-line flip on answer) and does not block the rest of the phase: the seam refactors (step 1) and infra stand-up (step 2) proceed while OD1/OD2/OD4/OD5 are answered. A fork discovered mid-run is a frontloading miss — note for HR. Batch OD1/OD2/OD4/OD5/OD6 into the CEO's founder pre-flight.

---

## 10. Decision & consultation pointers

- **Logged decisions (to `decisions.md`):** OD3 store engine = DynamoDB single-table (internal, reversible, behind `FactLogStore`); the `ConnectionState` consolidation; the conservative-0% CGT fallback for unrecognized accounts (closes D83-T5, replaces the silent-50% hazard).
- **Architect consultations (new invariants raised back to `ledger-architect`):** (1) the **`FactLogStore` protocol seam** — extracting the durable log behind an interface without behaviour change (R8-locked, byte-identical-replay `domain-core.md` A3 must stay byte-identical, seam-neutral per `backend-lift.md` A1); (2) the **`revision: Int` / `supersedes: FactID?`** additive fact-model change for IBKR restatements — version-bumping, must preserve deterministic replay (`data-model.md` impact, OD4); (3) the **conservative 0% CGT fallback** for accounts with no `EntityDef` — an invariant that must never silently apply the 50% discount.
- **Spec impact (co-owned PM + Architect):** all in `specs/backend-lift.md` — §E (scheduled+triggered pull, idempotent ingest, `SyncHealth`: R12–R16/R21, A8–A10/A13), §A/§C (the `FactLogStore` seam, append-only, FactID idempotency, the byte-identical seed-parity bar: R1/R7/R8, A1/A2/A3/A16), and §G (the read API contract + identity scoping: R19–R21, A12–A13). Each `A#` traces to a use-case (server sync `UC-SCHED-1/2`, sync-health view `UC-HEALTH-2`, remote alert `UC-HEALTH-3`) or to a preserved invariant (byte-identical-replay `domain-core.md` A3 / `backend-lift.md` A2, the R8/R23 native-currency display firewall).
