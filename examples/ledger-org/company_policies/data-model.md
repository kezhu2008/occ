# Ledger — Data Model

> Owned by the **Architect** (`ledger-architect`). Reviewed by `ledger-architect-reviewer`. The
> authoritative type/schema design: every executor implements against this and every reviewer checks
> code against it. CO-OWNS `specs/` with the Product Manager — this doc supplies the **types and
> invariants**; the spec supplies the `R#`/`A#` behaviour that folds over them.
>
> **Companion docs:** `architecture.md` (current-system seams/invariants, ARCHITECT-owned),
> `system-design.md` (W6 AWS deployment/API/migration, TECH-LEAD-owned), `use-cases/` (PM).
> **Grounding:** the logical layer is the proven on-device `LedgerCore` (Swift 6.2, Foundation-only),
> preserved verbatim — this is the W6 server-lift seam. The physical layer (multi-tenant DynamoDB) is
> new. The conceptual layer formalises the account/entity/connection spine that today's code runs at
> N=1.

This is the centerpiece of the W6 backend lift. Tax/CGT correctness, byte-identical determinism,
multi-tenant isolation, and the AWS storage shape all fall out of getting it right. It is written in
three layers — **conceptual** (entities + relationships), **logical** (exact fields, event schema,
invariants), **physical** (multi-tenant tables + access patterns) — so the move to Lambda is *a lift,
not a rebuild*: storage/secrets/schedule/notify/transport move behind the seams, the core types do not.

---

## 1. Invariants (the rules the model must never break)

Load-bearing. Every type and field below serves these; the reviewers FAIL a change that breaks one.

- **I1 — Facts are the only truth.** Lot states, positions, tax, and reports are a pure projection
  folded over `[Fact]`. Any stored derived state is a rebuildable cache, never a source of truth.
- **I2 — Native-only facts.** A `Fact` stores the signed amount in the instrument's **native currency**
  plus the raw FX rate observed; it NEVER stores an AUD/marked/derived value. AUD conversion happens at
  projection time via an injected FX resolver. `fxAtEvent == nil` is the FX-gap signal — absent FX is
  never fabricated.
- **I3 — Display never touches tax.** Price marks, daily P/L, and the `netCash` attribute feed
  positions/valuation only. No lot-ledger / CGT / income input reads them. Swapping the quote feed can
  move market value, never cost base or tax.
- **I4 — Determinism.** Projections read only `[Fact]` + an explicit `ProjectionConfig` — no `Date()`,
  `Locale.current`, `UUID()`, or randomness. Same facts + same config ⇒ **byte-identical** output (the
  byte-identical-replay property, spec anchor **A3**; the spec's R8 determinism requirement). The total
  fold order is `(eventDate, stableSourceID, FactID)`.
- **I5 — Append-only + content-addressed identity.** The fact log is append-only;
  `FactID = hash(stableSourceID, kind)` makes re-ingest idempotent with **no cursor**.
- **I6 — Entity + account on EVERY fact.** Tax is per-entity and accounts are isolated, so `entityId`
  and `accountId` are stamped at reconcile time on every fact, never bolted on later.
- **I7 — Preserved originals.** A `Parcel`'s `quantity` and `costBaseNative` are written once and never
  mutated; only `remaining` decrements. Splits supersede (new parcel), they do not edit. So the CGT
  engine can re-match parcels↔disposals under any `ParcelPolicy`.
- **I8 — The entity join.** `EntityDef.id` MUST equal the reconciler-stamped `Fact.entityId`. An
  unmatched account fails SAFE (I9), never silently getting the 50% individual CGT discount.
- **I9 — Safe fallback (binding now).** An unrecognised/unassigned real account degrades to a **flagged
  data-gap** treated as un-assessable until assigned, or at minimum the **most conservative 0% CGT
  discount** — never a silent 50%. A gain is never under-stated.
- **I10 — Tenant isolation.** Every persisted item's tenant boundary is derived from the **verified**
  Cognito JWT `tenantId`, never a client-supplied value. One tenant can never read or write another's
  partition.
- **I11 — Reconciliation is EXACT.** Reconciled cash / closing balances are compared by **Decimal exact
  equality** — never a tolerance, never an epsilon. A 1-cent delta is a FAIL, not a rounding pass.

---

## 2. Conceptual model (entities & relationships)

### 2.1 The spine

```
 Tenant ─1──< Connection ─1──< Account >──N──1── TaxEntity
 (user)        (IBKR Flex      (broker acct       (taxpayer:
               token+queryId)   Uxxxxxxx)          IND 50% / SMSF 33⅓% / Company 0%)
                                   │
                                   │ owns (every Fact carries entityId + accountId)
                                   ▼
                                  Fact ──folds──▶ LotLedgerState  (one per (entity,account,instrument))
                                 (event)          = open Parcels + DisposalLines  ← THE UNIT OF STATE
                                   │                          │
                                   │                          └─roll up by Scope─▶ Account stats
                                   │                                              ─▶ TaxEntity stats
                                   └─income / cash kinds feed projections directly ─▶ Portfolio stats
```

### 2.2 The seven core concepts

| Concept | What it is | Identity | Today (N=1) |
|---------|-----------|----------|-------------|
| **Tenant** | A user — the multi-tenant axis. NEW for W6. | `tenantId` (from the verified Cognito JWT) | implicit: one device = one tenant |
| **Connection** | An IBKR Flex data source = `{token, queryId}`. | `connectionId` | a single Keychain credential; no `Connection` type yet |
| **Account** | A broker account (`U0000000`) with type/currency. **The spine** — carries both `connectionId` and `entityId`. | `accountId` | hand-wired single account |
| **TaxEntity** | A taxpayer (CGT %, tax kind, elections). | `entityId` | `EntityDef`; one real entity today |
| **Fact** | An immutable economic event in native currency. | `FactID = hash(stableSourceID, kind)` | append-only `facts.jsonl` |
| **LotLedgerState** | The materialised per-ticker state (parcels + disposals). | `LotLedgerKey(entityId, accountId, instrumentSymbol)` | rebuilt in memory |
| **Projection** | A versioned, digest-stamped read model at a `Scope`. | `(name, version, factDigest)` | in-memory projection store |

### 2.3 Cardinalities (the account/entity/connection spine)

- **Connection 1 : N Account** — one login can carry many broker accounts.
- **Account N : 1 TaxEntity** — many accounts roll up to one taxpayer (W1 already had two accounts → one `IND`).
- **Connection ↔ TaxEntity = N : M through Account** — neither owns the other; **`Account` is the
  join.** It carries `connectionId` (which login delivered it) and `entityId` (which taxpayer owns it).
- **Ownership rule:** the *tax profile* (CGT %, kind, Div-775 election, option treatment) lives on
  **TaxEntity** and survives regardless of which connection delivered the account; *ingest / fact log /
  re-pull / credential* live on **Connection (+ Account)**.

---

## 3. Core types

The authoritative types. Money is **always `Decimal`**, never `Double` (I2, §8). All fact-layer types
are immutable, `Codable`, and byte-canonical.

### 3.1 The Fact — the event atom (logical model)

Immutable, `Codable`, canonical JSON via `.sortedKeys` + fixed Decimal formatting.

| Field | Type | Notes / invariants |
|-------|------|--------------------|
| `id` | `FactID` | dedupe key = `hash(stableSourceID, kind)` (I5) |
| `stableSourceID` | `String` | verbatim IBKR id (tradeID / transactionID / transferID / actionID) |
| `entityId` | `String` | taxpayer entity (I6) — e.g. `"IND"`, `"Acme Holdings Pty Ltd"` |
| `accountId` | `String` | IBKR account (I6) — e.g. `"U0000000"` |
| `instrument` | `Instrument?` | `nil` for pure-cash facts |
| `kind` | `FactKind` | economic classification — the 13-value vocabulary (§4) |
| `eventDate` | `DateComponents` | AU calendar y/m/d, **no time-of-day** (drives FY + holding period) |
| `nativeAmount` | `Money` | signed, native currency (I2) |
| `quantity` | `Decimal?` | signed (buy +, sell/write −); `nil` for cash |
| `fxAtEvent` | `Decimal?` | raw native→AUD rate; `nil` ⇒ FX gap (I2) |
| `fee` | `Money?` | commission / fee when present |
| `attributes` | `[String:String]` | kind-specific extras — the open per-kind schema (§4.2) |

### 3.2 Instrument

`Instrument = { symbol: String, kind: InstrumentKind {equity | etf | option}, currency: String,
exchange: String, option: OptionSpec?, conid: String? }`. ETF classification is a hardcoded set
`{VAS, VGS}`, else equity.

`OptionSpec = { underlyingSymbol: String, strike: Decimal, expiry: DateComponents, putCall: {P | C},
multiplier: Int }`. **Long vs short is NOT in the spec — it is the sign on `quantity`.** The OCC symbol
is a fixed 21-char `root(6) + YYMMDD(6) + C|P(1) + strike×1000(8)` with exact round-trip.

### 3.3 LotLedgerState — the unit of state (what the fold produces)

One per `LotLedgerKey(entityId, accountId, instrumentSymbol)`:

- **`parcels: [Parcel]`** — acquisition lots, oldest first.
- **`disposals: [DisposalLine]`** — realised disposals with their FIFO `consumed` breakdown, kept
  **separate** from parcels so the CGT engine re-matches under the user's `ParcelPolicy`.

**`Parcel = { id: ParcelID, acquiredOn: DateComponents, quantity: Decimal (preserved),
remaining: Decimal (var), costBaseNative: Money (preserved), fxAtAcq: Decimal?, origin: ParcelOrigin,
flags: ParcelFlags (var) }`** (I7: `quantity` + `costBaseNative` written once, only `remaining` moves).

- **`ParcelOrigin`** = `buy | transferIn | assignment | drp | writtenOption`.
- **`ParcelFlags`** (all default-false, confidence signals that NEVER change numbers — I3):
  `missingBasis`, `partialGap`, `fxUnverified`, `splitAdjusted`, `costBaseRestated`, `writtenOption`.

### 3.4 TaxEntity / config types (data that is not facts — §9)

`EntityDef = { id: String, taxKind: TaxKind {individual | smsf | company}, cgtDiscount: Decimal
(STORED: 0.5 / ⅓ / 0), accountIds: [String] }`. `cgtDiscount` is stored, not derived; `id` must equal
the stamped `Fact.entityId` (I8). The full config bundle (`ProjectionConfig`) is detailed in §9.

### 3.5 Projection

A versioned, digest-stamped read model produced by folding facts at a `Scope`. Identity =
`(name, version, factDigest)`. Reused while version + digest match the current fact set; otherwise
rebuilt by replay (I1, I4). Examples: positions, valuation, CGT, income, EOY tax summary,
reconciliation.

---

## 4. The event vocabulary & per-kind attribute schema

### 4.1 The 13 `FactKind`s

`trade` · `dividend` · `distribution` · `interest` · `depositWithdrawal` (capital in/out, **never
assessable**) · `withholding` (foreign tax split out, for FITO) · `corporateAction` (split /
consolidation / DRP-restate) · `optionExercise` · `optionAssignment` · `optionExpiry` · `transferIn`
(may arrive basis-less → gap) · `costBasisFill` (**user-appended** gap resolution; not
reconciler-emitted) · `forexConversion` (AUD↔USD cash leg; **no instrument, no parcel** — economics
live in attributes).

**Two-fact rule:** a foreign dividend emits **both** a `dividend`/`distribution` income fact **and** a
separate `withholding` fact sharing `stableSourceID` but distinct in `kind` (so distinct `FactID`).

### 4.2 The attribute schema (the stringly-typed bag, made explicit)

`Fact.attributes` is a `[String:String]` bag so the fact stays `Codable`/canonical without a union
type. The hidden schema, per producing kind:

| Kind(s) | Key | Format | Meaning |
|---------|-----|--------|---------|
| `trade` | `netCash` | canonical Decimal, signed | commission-folded cash leg; **DISPLAY-ONLY (I3)** — prefers IBKR `<Trade netCash>` |
| `forexConversion` | `fxBaseCurrency` / `fxBaseAmount` | ISO ccy / signed Decimal | BASE leg (e.g. AUD) currency + amount |
| `forexConversion` | `fxQuoteCurrency` / `fxQuoteAmount` | ISO ccy / signed Decimal | QUOTE leg (e.g. USD) |
| `forexConversion` | `fxRateQuoteToBase` | canonical Decimal | implied quote→base rate, sourced never fabricated |
| `forexConversion` | `fxCommissionCurrency` / `fxCommissionAmount` | ISO ccy / signed Decimal | commission leg |
| `forexConversion` | `forexGap` | `"true"` | degrade flag when a leg is unusable — never a phantom |
| option events | `transactionType` | raw Flex (`Exercise`/`Assignment`/`Expiration`) | always written |
| option events | `strike` / `premium` | raw strings | strike link; per-contract premium folded into underlying basis |
| `transferIn` | `costBasisGap` | `"true"` | no basis supplied → flagged gap, never fabricated |
| `transferIn` | `transferType` | raw Flex | provenance of the transfer |
| `corporateAction` | `caType` | raw Flex (`FS`/`RS`/`DRP`) | drives split-rebase vs delta-restate |
| `corporateAction` | `ratio` / `amount` | raw strings | split ratio / DRP cash component (native) |
| cash kinds | `cashType` | raw Flex `type` | always written |
| `dividend`/`distribution` | `frankingPct` / `frankingCredit` | raw % / native Decimal | franking data when supplied |
| `withholding` | `dividendSource` | source `stableSourceID` | links withholding → its dividend for FITO pairing |
| any with notes | `note` | raw string | passthrough IBKR notes |

> **Backend treatment:** these keys are an implicit schema. Persisted as-is they are portable, but the
> backend treats them as a **versioned per-kind contract** — validate on ingest, document here. The
> supersede proposal (§6.3) and any future typed payloads attach here without changing the fold.

---

## 5. Relationships

- **Tenant 1 : N Connection 1 : N Account N : 1 TaxEntity.** `Account` is the join carrying both
  `connectionId` and `entityId` (§2.3). Connection ↔ TaxEntity is N:M *through* Account.
- **Account 1 : N Fact** — every `Fact` is stamped `accountId` + `entityId` at reconcile time (I6).
- **Fact (many) folds to → LotLedgerState (one)** per `(entityId, accountId, instrumentSymbol)`. Income
  and cash kinds feed positions/valuation directly without a lot.
- **LotLedgerState 1 : N Parcel** and **1 : N DisposalLine**; disposals reference the parcels they
  consume but are stored separately (I7) so re-matching under any `ParcelPolicy` is possible.
- **TaxEntity 1 : N Account**; the entity owns the tax profile, the account owns the ingest lineage.
- **LotLedgerState rolls up by `Scope`** to Account / TaxEntity / Portfolio stats (§7.3). The natural
  state cube is **instrument × account × entity × portfolio**, all derived by deterministic aggregation
  from the one ticker-level ground state — which is why the physical partition key is
  `(tenant, entity, account, instrument)` (§10).

---

## 6. Identity, ordering, idempotency

- **`FactID`** = 16-char hex of a **seed-free FNV-1a 64-bit** hash over a **length-delimited** encoding
  of `[stableSourceID, kind]` (so `("a","bc") ≠ ("ab","c")`). Seed-free because a per-process random
  hasher would break cross-run reproducibility (I4).
- **`LotLedgerKey`** = `{entityId, accountId, instrumentSymbol}` — the ticker-state key.
- **`parcelID(key, seq)`** = `"P-<instrumentSymbol>-<seq>"`, deterministic.
- **Total order** for the fold: `(eventDate, stableSourceID, FactID)`. Day-granular — same-day events
  tie-break on id, not execution time (a known, deterministic trade-off).
- **Idempotency:** append rejects an existing `FactID` (an O(1) `Set<FactID>` on device; a DynamoDB
  conditional put server-side). No since-cursor — the IBKR `queryId` defines the window and dedup
  absorbs the overlap.

### 6.3 The one required change — restatement / supersede (currently a gap)

Identity is `(stableSourceID, kind)`, so an IBKR **restatement** of a row (same id + kind, corrected
amount) is silently dropped as a duplicate. Server-side, with repeated unattended pulls, this matters.
**Proposed:** add `revision: Int = 0` + `supersedes: FactID?` to the fact; the fold treats the highest
revision as authoritative, keeping predecessors in the log (marked superseded) for audit. Additive,
version-bumping, replay stays deterministic. (Boss decision — build now vs backlog: OD-DM1.)

---

## 7. Derived / projected state (the fold)

### 7.1 The per-ticker state machine (events → transitions)

`trade` BUY / `transferIn` → open parcel · `trade` SELL → FIFO-consume `remaining` + record
`DisposalLine` · `corporateAction` FS/RS → supersede (rebased) parcels (`splitAdjusted`) · DRP-restate
→ cost-base delta on latest open parcel (`costBaseRestated`) · `optionExpiry` → close option parcel ·
`optionExercise`/`optionAssignment` → close option **+ open underlying** (cross-key) · `costBasisFill`
→ clear gap flag. Parcel sub-lifecycle off `remaining`: **open → partially-consumed → closed**, or
**written-short** (negative) winding to 0.

### 7.2 Roll-ups by `Scope`

`Scope = { all | entity(id) | account(id) }`. The same ticker states aggregate up:

- **Account stats** = all tickers in an account (positions group + total; cash per
  `(entity, account, currency)`).
- **TaxEntity (tax) stats** = the taxpayer roll-up; tax engines run at entity scope under that entity's
  `TaxKind` / discount / elections.
- **Portfolio stats** = across all (totals, scope-relative weights).

### 7.3 Rebuild & idempotence guarantee

Every projection is a pure function of `[Fact]` + `ProjectionConfig` (I1, I4). Drop any cached
projection or any cached `LotLedgerState` and replay ⇒ **byte-identical** output (the byte-identical-replay
property, spec anchor **A3** — this is what QA asserts). Config is NOT in the fact digest: a config edit
drops the tenant's projection cache so the next read rebuilds.

---

## 8. Money & determinism

`Money = { amount: Decimal (full precision, signed), currency: String }`; display `scale = 2`.
Arithmetic **traps** on currency mismatch (`+`/`-`); non-trapping variants return `nil`. Rounding is
**explicit at the edges only** — bankers' (half-to-even) at scale 2; canonical form `"1234.50 AUD"`,
`en_US_POSIX`, no grouping. **Never `Double`.** This is what makes I4 (byte-identical recompute) and
I11 (exact reconciliation) hold.

---

## 9. Configuration model (the tax-position data — per ENTITY)

This is *data that is not facts* but drives the fold — the user's tax position. It is **per-entity**,
byte-canonical, and a change invalidates cached projections.

| Type | Shape | Notes |
|------|-------|-------|
| `ProjectionConfig` | `{today, parcelPolicy, perEntityOptionTreatment, entities:[EntityDef], lodgedFYs:Set<String>=[], div775Election:[String:Div775Election]=[:], forexCostBaseMethod=.weightedAverage}` | all explicit inputs (I4) |
| `EntityDef` | `{id, taxKind, cgtDiscount: Decimal, accountIds:[String]}` | `cgtDiscount` STORED not derived (0.5 / ⅓ / 0); `id` must equal stamped `entityId` (I8) |
| `TaxKind` | `individual | smsf | company` | company → **no 50% discount** (rate 0) |
| `ParcelPolicy` | `fifo | lifo | minGain | maxGain` | disposal parcel selection |
| `OptionTreatment` | `capital | revenue` | per-entity option premium treatment |
| `Div775Election` | `.standard | .balanceElection(thresholdAUD: Decimal)` | per-entity forex-realisation election; absent ⇒ `.standard` |
| `ForexCostBaseMethod` | `weightedAverage | fifo` | fungible-currency cost-base method for Div 775 |
| `lodgedFYs` | `Set<FY.id>` | a lodged FY is pinned to its snapshot policy, not recomputed under live policy |

> **Backend consequence:** `ProjectionConfig` becomes the per-tenant **EntityDirectory** item (§10),
> editable via the read API. The Div-775 election is **per-entity** — the data model is already correct.

---

## 10. Physical model — multi-tenant AWS storage (W6)

Single-table DynamoDB, tenant-isolated by partition prefix. (Aurora Postgres is the swappable
alternative behind the `FactStore` seam.) Every item's tenant boundary is derived from the **verified**
JWT `tenantId`, never a client value (I10).

### 10.1 Item types & keys

| Item | PK | SK | Idempotency / notes |
|------|----|----|---------------------|
| **Fact** | `T#<tenant>#C#<conn>` | `F#<eventDateKey>#<factId>` | append-only; **conditional put `attribute_not_exists(SK)`** = FactID dedup (I5) for free; SK ordering preserves `(eventDate,…)` |
| **LotLedgerState** | `T#<tenant>` | `S#<entity>#<account>#<instrument>` | the ground state (§3.3); range-query by `S#<entity>#<account>` prefix → account/portfolio roll-ups; rebuildable cache |
| **StampedProjection** | `T#<tenant>` | `P#<name>#<scope>` | versioned + `factDigest`-stamped; reused while version + digest match, else rebuilt |
| **Connection** | `T#<tenant>` | `CONN#<conn>` | `{label, queryId, schedule, lastPullAt, lastStatus, jobLock(ttl)}`; **secret NOT here** — token lives in Secrets Manager |
| **EntityDirectory** | `T#<tenant>` | `DIR` | the persisted `ProjectionConfig` (entities, account→entity, TaxKind, elections, policy) — closes the spine |
| **AlertRule** / **AlertState** | `T#<tenant>` | `ALERT#<id>` / `ALERTSTATE` | rules + the threaded armed/fired set |
| **DailySnapshot** | `T#<tenant>#C#<conn>` | `SNAP#<sydneyDay>` | **conditional put** = `saveIfAbsent` freeze-through-the-day for free |
| **DeviceToken** | `T#<tenant>` | `DEV#<apnsToken>` | for SNS→APNs push fan-out |

### 10.2 Shared (cross-tenant) tables

- **QuoteCache** — `PK=symbol`, `SK=asOfBucket`, TTL on delayed quotes; one fetch per symbol serves all
  tenants (removes per-device duplication). Quotes feed display only (I3).
- **S3** — raw pulled statement XML (`<tenant>/<conn>/statements/<pulledAt>.xml`, SSE-KMS,
  lifecycle-expired) + generated PDF/CSV exports (presigned URLs). Facts in Dynamo remain the truth.

### 10.3 Access patterns (the table is designed around these)

1. Ingest facts for a connection → batch conditional-put under `T#<tenant>#C#<conn>`.
2. Replay a tenant's facts (optionally scoped) → query `T#<tenant>#C#*` (or filter by account).
3. Serve a projection at a scope → get `P#<name>#<scope>`; on miss/stale → rebuild from facts + cache.
4. Account/portfolio position roll-up → query `S#<entity>#<account>` prefix, aggregate.
5. Sync health → get `CONN#<conn>`.
6. Freeze daily snapshot → conditional put `SNAP#<sydneyDay>`.
7. Alert evaluation → get rules + `ALERTSTATE`, thread, put back.
8. Edit config → put `DIR`, then drop `P#*` for the tenant (config not in the fact digest).

### 10.4 Migration & versioning

- **Seed:** load the existing on-device `real/facts.jsonl` into `T#<tenant>#C#<conn>` (FactID dedup
  makes re-seeding safe). **Gate:** server replay's tax digest == on-device golden — byte-identical
  (the W6 acceptance bar).
- **Model-version bumps:** a fact-schema change (e.g. the §6.3 `revision` field) is a clean re-pull, not
  an in-place migration — real data is re-pullable from the stored credential, so no data-loss path.
  Projection version bumps rebuild by replay.
- **Per-account partition:** `C#<conn>` already namespaces facts; `accountId` is in every fact, so an
  account-scoped wipe/migrate is a query + delete on a prefix — no data reshape.

---

## 11. Persisted shape (serialization)

**On-device / on-wire fact log:** append-only JSONL. Each line is one `Fact` encoded as `Codable` JSON
with `.sortedKeys` and fixed Decimal formatting (canonical `"1234.50 AUD"`, `en_US_POSIX`, no grouping).
Determinism is the contract: the same fact set serialised twice is **byte-identical**, and a replay
round-trips byte-for-byte (I4). `DateComponents` carry y/m/d only (no time-of-day). `Money.amount` and
all Decimal fields are serialised at full precision via the fixed formatter — never as floating-point.

**On-wire (DynamoDB):** the same canonical fact body stored as an item attribute under the keys in
§10.1. The append-only + content-digest log is the per-tenant exportable audit artifact.

**Versioning:** additive, version-bumping changes only (the §6.3 supersede fields are the model case).
A schema bump is a clean re-pull + replay, never a destructive in-place edit.

---

## 12. Validation rules (constraints on construction / ingest)

- **Money is `Decimal`.** Any `Double` in a money path is invalid by construction (I2, §8).
- **Entity + account present.** A fact missing `entityId` or `accountId` is invalid — the reconciler
  stamps both (I6). An account with no matching `EntityDef.id` fails SAFE (I8/I9): flagged data-gap or
  0% discount, never a silent 50%.
- **No fabricated FX.** `fxAtEvent == nil` is a legal, meaningful value (FX gap); ingest never
  back-fills a rate (I2).
- **No fabricated basis.** A basis-less `transferIn` is flagged (`costBasisGap`/`missingBasis`), never
  given a guessed cost base (I7/I9).
- **Idempotent append.** Re-appending an existing `FactID` is rejected (a byte-stable no-op), not an
  error path that duplicates (I5).
- **Reconciliation is EXACT (I11).** Reconciled cash == statement closing balance by **Decimal exact
  equality**. There is **no tolerance and no epsilon**: a 1-cent delta is a reconciliation FAILURE that
  must surface as a gap, never silently absorbed. (This is the offline-testable acceptance the spec's
  `A#` and QA assert.)
- **Tenant boundary from verified JWT.** A persisted item whose tenant prefix does not match the
  verified `tenantId` is rejected (I10).
- **Determinism preconditions.** A projection input that reads wall-clock/locale/randomness is invalid —
  only `[Fact]` + `ProjectionConfig` may drive the fold (I4).

---

## 13. Idempotency & multi-source reconciliation

### 13.1 Idempotency (single-source — what the model guarantees today)

Identity is content-addressed: `FactID = hash(stableSourceID, kind)` (I5). Ingest computes the id from
content; the store **rejects an already-held id** (O(1) `Set<FactID>` on device; a DynamoDB conditional
put `attribute_not_exists(SK)` server-side, §10.1). Consequences:

- **No sync cursor.** The IBKR `queryId` defines the pull window; re-pulling the overlapping statement
  every cycle is a byte-stable no-op — this is what makes scheduled re-sync safe.
- **Determinism extends it.** Same fact set → byte-identical projections (I4), so re-sync never drifts a
  tax number through the pipeline.
- **Guarantee boundary:** keyed on `(stableSourceID, kind)` *only*, not the body. Same source + row →
  deduped. **Restatement** (same id + kind, new amount) → silently dropped (the §6.3 supersede gap).
  **Same event from a *different* source** → different id → **counts twice** (→ §13.2).

### 13.2 Multi-source reconciliation (a NEW subsystem)

Today the model is single-source by construction: one truth (IBKR Flex) plus a file-import of the *same*
IBKR XML, sharing one id space, so dedup just works. Ingesting the same trade from a second source
(email confirmation, a different broker's report, manual entry) would **double-count**. Reconciliation
reuses three things the architecture already has: the `RawRecordSource`/`FactStore` seam, the
gap-detect→user-resolve pattern (the `costBasisFill` mechanism), and the projection/replay model.

**Two-layer identity:**

- **Layer 1 — source-record idempotency (keep every source record in the log).** Namespace the id by
  source: `SourceRecordID = hash(sourceSystem, sourceNativeId, kind)`. Same-source re-ingest is
  idempotent (§13.1); different sources produce different facts — no accidental collision, no accidental
  merge. The append-only log retains all source records → full provenance.
- **Layer 2 — canonical economic identity (collapse duplicates in a PROJECTION, never in the log).** A
  deterministic `economicFingerprint = hash(entity, account, instrument, kind, eventDate, normalizedQty,
  normalizedAmount)`. A **`ReconciliationProjection`** groups facts by fingerprint within a date-window
  and emits the canonical, deduped economic view that tax/positions fold over.

**Matching rules (deterministic → replay-safe, I4):**

- Exact fingerprint match across sources → **auto-link** (one canonical event; others corroborating).
- Near match (qty equal, amount within a configured window, date within N days) → raise a
  **`ReconciliationGap`**, surfaced like a cost-basis gap; the user's confirm/reject is recorded as a
  **resolution fact** (à la `costBasisFill`). The auto-part stays pure; the ambiguous part becomes an
  explicit recorded decision — replay stays byte-identical.

**Source precedence / trust:** each source has a trust rank — *authoritative* (broker Flex / daily
statement) > *provisional* (email, manual). The canonical event is the highest-trust corroborating
record; a provisional fact is **superseded** when the authoritative one arrives (reuses §6.3). Tax
always reads the authoritative canonical; provisional sources give early display only (I3).

> **Reconciliation comparison is EXACT (I11):** the *amount* check that decides auto-link vs gap is the
> only place a configured near-match *window* exists, and it exists precisely so that anything not an
> exact match becomes an explicit, user-confirmed gap — it is never silently merged. Cash/closing-balance
> reconciliation remains Decimal **exact equality with no tolerance**.

**New data-model elements introduced:** a `sourceSystem` + per-source trust rank on
ingest/`Connection`; the `economicFingerprint` derivation + window config (a deterministic
`ProjectionConfig` input, I4); a `ReconciliationProjection` (canonical-view read model);
`reconciliationResolution` as a user-resolution fact kind (mirrors `costBasisFill`); provenance — the
canonical event references its backing source-record FactIDs.

---

## 14. Open data-model decisions (boss)

- **OD-DM1 — Restatement/supersede (§6.3):** add `revision`/`supersedes` now, or backlog? Affects audit
  fidelity when IBKR corrects a row.
- **OD-DM2 — Store engine:** DynamoDB single-table (recommended; idempotency + freeze for free) vs Aurora
  Postgres (richer reporting). Swappable behind the `FactStore` seam.
- **OD-DM3 — Attribute schema hardening:** keep the stringly-typed bag (portable, minimal change) vs typed
  per-kind payloads. Recommend: keep the bag, add ingest-time validation + this doc as the contract.
- **OD-DM4 — Spine build depth:** Phase-1 (Path A, single-tenant unattended) can run with `Connection` as
  data (not a rich type) and N=1; confirm whether to build the full N-connection/N-account model +
  assignment UI now (Path B) or stage it. I9 safe-fallback is binding regardless.
- **OD-DM5 — Multi-source ingest (§13.2):** in scope this phase, or is IBKR the sole source (then §13.2 is
  a documented future extension and only §13.1 is built)? The window, auto-merge-vs-confirm threshold, and
  source trust ranking are policy choices for the boss if it is in scope.

---

## 15. Summary

A per-tenant, append-only event log of native-currency `Facts` (13 kinds; identity = content hash of
`(stableSourceID, kind)`; idempotent; no cursor) that folds **deterministically** into the ticker-level
`LotLedgerState` ground state and rolls up by `Scope` to account, tax-entity, and portfolio stats. Tax
position is per-entity config data (`ProjectionConfig`/`EntityDef`). Multi-tenancy and N-accounts come
from the account/entity/connection spine — **`Account` is the join** between a `Connection` (data
source/credential) and a `TaxEntity` (tax profile) — under the binding safe-fallback invariant (I9) that
an unassigned account never gets a silent discount. Physically this maps onto a DynamoDB single table
partitioned on `(tenant, connection)` for facts and `(tenant, entity, account, instrument)` for state,
where conditional puts give idempotency and snapshot-freeze for free. Money is **always `Decimal`** and
reconciliation is **exact equality, never a tolerance** (I11). The logical layer is today's proven core,
unchanged; only the storage and the tenant/connection spine are new — a lift, not a rebuild.
