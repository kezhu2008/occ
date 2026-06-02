# Ledger W6 — Backend Lift Test Strategy

> Owned by the **QA engineer** (`ledger-qa-engineer`). The acceptance + integration test plan for the
> **W6 Backend Lift (Path A, single-tenant unattended sync)**: which `A#` (from `specs/backend-lift.md`)
> each test asserts, the fixtures and oracle / byte-identical harnesses, cross-task integration
> coverage, and the Integrate-phase gate. Reviewed by `ledger-qa-engineer-reviewer` (tests truly assert the
> `A#`, non-vacuous, mutation-probed; cross-task integration real). Version-stamped per milestone —
> this revision covers **W6-M1** (the `FactLogStore` pre-refactor) and **W6-M2** (`LedgerCore` on
> Lambda, determinism proven); the later W6 milestones (durable store + secrets, schedule + trigger,
> read API + thin client, remote push, deploy acceptance) extend this matrix as they open.
>
> **Two-level QA — read this first.** This is **product QA**: holistic, against-the-spec, at the
> milestone Integrate phase — it guards the *product* (here: that the server lift moves **zero tax
> cell** and the server's tax bytes equal the device golden). It is **distinct from the reviewers**
> (`ledger-backend-engineer-reviewer`, `ledger-architect-reviewer`, `ledger-tech-lead-reviewer`), who
> are agentic-integrity QA: per-artifact PASS/FAIL gates that guard the *workflow* (catching
> hallucination, fabricated "done", vacuous tests, spec drift). Reviewers protect the process; I
> protect the product. Both run; neither replaces the other. The spec's `A#` is the shared harness.

## Scope

- **Milestone(s):** `W6-M1` — `FactLogStore` protocol pre-refactor (in-app, zero behaviour change); and
  `W6-M2` — `LedgerCore` on Lambda (server core, determinism proven). The W6 wave exit gate is **A2**
  (server tax bytes == device golden) plus the boss's end-to-end-on-real-data leg
  (`backend-migration-review.md` §7-M7; `milestones/W6.md`).
- **Specs under test:** [`specs/backend-lift.md`](specs/backend-lift.md) — R1–R24 / A1–A16; with the
  W1 [`specs/domain-core.md`](specs/domain-core.md) `A1` (tax oracle) / `A3` (replay) suites carried in
  as the byte-identical baseline the lift must **not** move (the W1 tax oracle, IND × SMSF, FY25 × FY26,
  golden `TaxProjection` digest `7f6784e1…`).
- **Tasks integrated:** [`tasks/W6-M1.md`](tasks/W6-M1.md) (the seam + `ConnectionState` consolidation
  + the closed D83-T5 0%-CGT-fallback hazard), [`tasks/W6-M2.md`](tasks/W6-M2.md) (Linux/Lambda
  package, the `SyncWorker` thin shell over `runRefresh`, the committed `.flex-local` golden +
  server-side replay in CI).
- **Use cases that must demonstrably work end-to-end** (from `use-cases/index.md` §6 + `backend.md`):
  `UC-CONN-1`, `UC-SCHED-1`, `UC-SCHED-2`, `UC-SCHED-3`, `UC-SCHED-4`,
  `UC-HEALTH-1`, `UC-HEALTH-2`, `UC-HEALTH-4` (the wave's headline UC). `UC-HEALTH-3`
  (A15 live-push leg) is gated by the OD6 APNs prerequisite and tested **offline** against a fake
  notifier in this revision; its live leg lands with the remote-push milestone.

## A#-to-test coverage matrix

> The core of this plan: **every `A#` in `specs/backend-lift.md` maps to at least one test here.** An
> `A#` with no test is a coverage gap the reviewer FAILs; a test that asserts no `A#` is scope creep.
> Each `A#` is executable by definition (a test asserts it without a human judging — `automation.md`:
> the spec is the harness). All `A#` are proven **offline** the way the existing wave harnesses run
> (`Div775AcceptanceTests` / `DailyPLR8RelockTests` over the real on-disk `.flex-local` bytes + a fake
> `FlexTransport` / canned snapshot JSON, never live network — D3/D45), RED-first + mutation-checked
> (D45), against an **independent oracle** where a value is asserted (D75 — never the projection's own sum).

| A# | what it asserts (one line) | test id / name | kind | fixtures | harness | task(s) covered |
|----|----------------------------|----------------|------|----------|---------|-----------------|
| A1 | Seam swap is behaviour-neutral — `LedgerStore` reaches the log only via `FactLogStore`; the on-device tax-oracle (`domain-core.md` A1) + replay/rebuildability (`domain-core.md` A3) suites pass **unchanged**; `TaxProjection` digest == golden `7f6784e1…` by exact string equality | `SeamSwapDigestParityTest` (+ re-run of `TaxAcceptanceTests`, `RebuildabilityTests`) | acceptance | `flex_dummy.xml` + `expected_tax.json`; real `.flex-local` bytes | byte-identical (digest exact-string) + oracle | W6-M1 |
| A2 | **THE wave bar** — server `FactLogStore` feeds `ProjectionEngine`; full `TaxProjection` (EOY + CGT disposals + income + forex + `forexRealisations`) via `CanonicalJSON` is byte-identical to the device file-store path; FY26 Div775 == −$1.76 AUD on the server path | `ServerVsDeviceReplayParityTest` | acceptance | real `.flex-local` golden + committed `ProjectionConfig`; `div775_oracle_fy26.json` (−$1.76) | byte-identical + oracle (exact equality, 1¢/1-byte FAILs) | W6-M2 (consumes W6-M1 seam) |
| A3 | Idempotent ingest, no cursor — re-ingest same statement ⇒ `append` returns 0, factCount unchanged, digest unchanged; a different statement with overlapping rows adds only genuinely-new FactIDs | `ServerIngestIdempotencyTest` | acceptance | real `.flex-local` golden + an overlapping-window second-pull fixture | oracle (count == 0 / digest equality) + boundary | W6-M2 |
| A4 | R8 byte-identical tax across the lift (the re-lock, non-vacuous) — full `TaxProjection` identical across `{no daily-P/L}` vs `{daily-P/L + cash + two snapshots}` on the **server** path while MV/daily-P/L move | `ServerDailyPLR8RelockTest` (server-side run of `DailyPLR8RelockTests`) | acceptance | real `.flex-local` + two non-degenerate canned `DailySnapshot` JSONs | byte-identical + boundary/failure (negative control) | W6-M2 |
| A5 | Tenant isolation — a request verified as T1 reads/writes only T1's resources, cannot touch any T2 resource; a forged body `tenantId` is ignored (isolation key = the **verified** identity); T2-leak byte-count == 0 | `TenantIsolationTest` | acceptance | two seeded fact sets `tenantA.jsonl` / `tenantB.jsonl` | property/invariant (no-cross-read) + boundary (forged id) | (durable-store milestone; **single-tenant cut** of A5 — see note) |
| A6 | N connections, isolated removal — remove C2 ⇒ C2 credential deleted + C2 stops syncing **and** C1's credential/facts/schedule unchanged; `Scope.all` drops exactly C2's contribution | `IsolatedConnectionRemovalTest` | acceptance | two-connection fixture `conn_C1.jsonl` / `conn_C2.jsonl` | oracle (C1 factCount before==after) + invariant | (durable-store milestone; seam landed W6-M1) |
| A7 | Safe entity fallback — unassigned account ⇒ disposals use **0%** CGT discount (never 50%) + flagged "needs assignment"; assign to Individual ⇒ 50% applies + flag clears | `EntityFallbackDiscountTest` | acceptance | `unassigned_account.flex.xml` (one account, no `EntityDef`) | oracle (discount==0%) + boundary (mutation-to-50% FAILs) | W6-M1 (closes D83-T5) |
| A8 | Unattended sync wiring (offline) — handler wired with server collaborators; `reason=.scheduled` and `reason=.onDemand` invoke the **same** `runRefresh` body, same ingest result for same bytes; fixed instant flows through `SydneyMidnightBoundary` to a deterministic snapshot key | `SyncWorkerSharedPathTest` | integration | real `.flex-local` (via fake `FlexTransport`) + fixed server-clock instant | property/invariant (one code path) + byte-identical (snapshot key) | W6-M2 (handler shell) |
| A9 | Job lock + throttle — concurrent invocation while lock held ⇒ no 2nd IBKR pull, fact set == single-pull result; last-pull within throttle window ⇒ "try again shortly", **no** IBKR call | `JobLockThrottleTest` | acceptance | seeded connection + fake transport with a call-counter | boundary/failure (assert call-count) + invariant | (schedule+trigger milestone; seam landed W6-M1 `ConnectionState`) |
| A10 | Degrade-safe failed pull — pull fails ⇒ prior facts + last-good snapshot unchanged, `ConnectionState=.error/.degraded` with a **token-free** reason, subsequent read still returns last-good with an honest stale stamp | `DegradeSafePullTest` | acceptance | seeded connection + fake transport returning an IBKR-error / network-failure | boundary/failure + oracle (digest unchanged) | (schedule+trigger milestone) |
| A11 | Token never leaks — across a full sync+read cycle, the IBKR token appears in **no** log / error type / response body / fact JSONL / projection / export (grep-clean, extends D31); read/API role has no path to a credential | `TokenNeverLeaksTest` (grep sweep over all emitted artifacts) | acceptance | a canned credential with a known sentinel token string | boundary/failure (grep-clean assertion) | W6-M2 (the `SyncWorker` shell) |
| A12 | API payloads decode to the same core types — read response decoded with the client's `LedgerCore` `Codable` `Output` types == the directly-computed projection (round-trip); body is `CanonicalJSON` | `APIRoundTripTypeEqualityTest` | acceptance | real `.flex-local`-derived projection outputs | byte-identical (CanonicalJSON) + oracle (round-trip equality) | (read-API milestone; types frozen by W6-M2 core) |
| A13 | Freshness + SyncHealth fields present & accurate — `lastPullAt` == known instant, `factCount`+`digest` == store's, `nextScheduledAt` in the future, `staleness` == f(asOf − lastPullAt); past-threshold reports `stale` | `SyncHealthFieldsTest` | acceptance | seeded connection pulled at a fixed instant + an injected `asOf` | oracle (field-exact) + boundary (stale threshold) | (read-API milestone) |
| A14 | Write round-trip recomputes — a config write (Div775 election, parcel policy, gap `costBasisFill`) invalidates + recomputes affected projections; result differs **iff** economically material; a 2nd device reads the new result | `WriteRoundTripRecomputeTest` | acceptance | real `.flex-local` + a `costBasisFill` payload + a Div775-election flip | oracle (material-change ⇔ value-change) + invariant | (write-API milestone) |
| A15 | Server-evaluated alert fires once — armed rule + crossing snapshot ⇒ exactly one `FiredAlert` to the push seam; next sync still-crossed ⇒ **no** second `FiredAlert` (re-arm/de-dupe holds); asserted against a **fake notifier** (no live APNs) | `ServerAlertFiresOnceTest` | acceptance | armed rule fixture + a crossing canned snapshot + a same-state re-pull | property/invariant (fire-once) + boundary | (remote-push milestone; live APNs gated by OD6) |
| A16 | Append-only + exportable — store API exposes only `append`/`allFacts`/`digest` (no update/delete), `allFacts()` returns canonical `(eventDate, stableSourceID)` order, a per-tenant canonical-bytes export round-trips to the same `digest()` | `AppendOnlyExportRoundTripTest` | acceptance | seeded tenant log (real `.flex-local`) | property/invariant (ordering) + byte-identical (export digest) | W6-M1 (protocol surface) |

- **kind:** `acceptance` = asserts one `A#` directly against the spec. `integration` = asserts an `A#`
  (or a use-case end-to-end) **across task boundaries** (see the integration section). `A8` is marked
  `integration` because it proves the W6-M1 seam (`FactLogStore`) and the W6-M2 handler shell compose
  through one shared `runRefresh` path.
- High-stakes `A#` (money, secrets, irreversible) get a positive case **and** a failure case: **A2/A4**
  (a 1¢/1-byte delta must FAIL — no tolerance on tax bytes), **A7** (the 50%-discount mutation must
  FAIL), **A11** (the token sentinel must FAIL the grep if present), **A3** (a non-zero `append` return
  on a re-pull must FAIL).
- **Single-tenant cut note (OD1):** W6 is Path A. `A5`/`A6` are asserted in their `tenantCount == 1`
  form — the **isolation invariant is exercised by construction** (the storage prefix is derived from a
  verified tenant identity even when there is one tenant) so the seam can't be wired against a
  client-supplied id; the full multi-tenant cross-read matrix opens with Path B (W7). This is recorded,
  not waived — A5's two-tenant fixture is committed now so the harness is ready.

## Fixtures

> Inputs and expected outputs that make each `A#` provable offline, with no human and no network.
> Deterministic, version-pinned, committed. The lift adds **no new fixture kind** — it reuses the W1/W2
> real-bytes harness so "server == device" is a like-for-like comparison.

| fixture | shape / source | feeds tests | owner / provenance |
|---------|----------------|-------------|--------------------|
| `Fixtures/flex_dummy.xml` + `Fixtures/expected_tax.json` | the W1 bundled multi-account/multi-entity statement + its hand-derived tax oracle (`Bundle.module`) | A1 | committed W1 artifact; the A2 oracle, IND × SMSF, FY25 × FY26 |
| real `.flex-local` golden bytes | the founder's real recorded IBKR statement (gitignored live; a **committed golden snapshot** of the bytes used as the server replay input) | A1, A2, A3, A4, A8, A14, A16 | captured + held per D31 (`.flex-local` gitignored); the golden snapshot is a reviewed artifact, scrubbed of the token (token never in the bytes — it is request-time only) |
| `golden/server_tax_projection/` | expected byte-identical `CanonicalJSON` `TaxProjection` snapshot (EOY + CGT + income + forex + `forexRealisations`) — the **device** build's output, the parity target | A2 | regenerated only on an intentional, reviewed core format change (re-baselined exactly like D70/D85) |
| `Fixtures/div775_oracle_fy26.json` | the hand-computed FRE-1 per-currency cost-base walk over the 13 USD legs → **−$1.76 AUD** (D84) | A2 | independent oracle (D75) — first-principles, NOT the projection's own sum |
| `Fixtures/quotes/` + two `DailySnapshot` JSONs (non-degenerate, distinct) | canned Yahoo chart JSON over a fake `QuoteTransport`; two different Sydney-day snapshots | A4, A8, A15 | committed; reconstruct the 00:00 Australia/Sydney instant deterministically (no live `Date()`/`TimeZone`) |
| `Fixtures/overlap_second_pull.flex.xml` | a second statement whose date window overlaps the golden (shared rows + a few new FactIDs) | A3 | derived from the golden; proves dedup absorbs overlap, adds only the new |
| `Fixtures/unassigned_account.flex.xml` | one account with **no** `EntityDef` / no assignment | A7 | crafted; the D83-T5 hazard input |
| `Fixtures/tenantA.jsonl` / `Fixtures/tenantB.jsonl`; `conn_C1.jsonl` / `conn_C2.jsonl` | two distinct seeded fact sets; a two-connection fixture | A5, A6 | crafted; isolation + isolated-removal harness (ready now, asserted in the single-tenant cut + Path-B) |
| canned credential with a sentinel token string | a deliberately recognizable fake token value | A11 | crafted; the grep-clean negative control |

- Fixtures are **deterministic and offline** — pinned, no live calls, no clock/locale/random leakage.
  The Sydney-midnight instant, `today`, FX, marks, and the Div775 election are **explicit inputs** (D8);
  a fixture that needs the network or a wall clock is not a fixture. (R6 forbids the lift introducing
  any `Date()`/`TimeZone.current`/`Locale.current`/`UUID()` — the fixtures enforce it by supplying every
  such value.)
- **Golden/baseline files are reviewed artifacts.** A diff to `golden/server_tax_projection/` is a
  behaviour change: it gets re-reviewed (architect + `ledger-qa-engineer-reviewer`), never silently re-blessed to make a
  test pass. The golden is the **device** output — so a server diff means the server diverged, which is
  exactly the A2 failure we are guarding.
- The real `.flex-local` golden is **scrubbed**: the IBKR token is request-time only and never appears
  in the statement bytes, the fact log, or any committed artifact (D31). The grep sweep (A11) is run
  over the committed fixtures too, as a standing guard.

## Harnesses (how the outcome is judged)

> Per `automation.md`, every acceptance is outcome-harnessed: the result, not a human, is the arbiter.
> Each `A#` names the harness it is bound to.

- **Byte-identical / replay harness (the wave's centrepiece).** Feed the **server** `FactLogStore`
  implementation and the **device** file store the *same* facts + the *same* `ProjectionConfig`, run
  `ProjectionEngine`, serialise both `TaxProjection`s via `CanonicalJSON` (sorted keys, exact
  `Decimal`), and assert **exact string equality** of the bytes (and of the `factDigest`). Any
  non-determinism the lift could introduce — an unsorted key, a float drift, a locale/clock leak in the
  server wiring, a re-ordered fold — turns this RED. Binds **A2, A1, A4, A8 (snapshot key), A16
  (export digest)**. This is the literal G2/NFR1 acceptance bar: server bytes == device golden.
- **Oracle / exact-equality harness.** An independently-computed expected value the system must match by
  exact equality — `expected_tax.json` (W1 hand-derived oracle, IND × SMSF, FY25 × FY26), the Div775
  **−$1.76 AUD** first-principles cost-base walk (`div775_oracle_fy26.json`, D75 — never the projection's
  own sum), the digest `7f6784e1…`. A 1¢ / 1-byte delta FAILs — **no tolerance on money** (R5/R23).
  Binds **A1, A2, A3, A7, A13, A14**.
- **Property / invariant harness.** Spec invariants asserted over inputs, not just one example: idempotent
  re-pull returns 0 (**A3**), `allFacts()` canonical order + no update/delete API (**A16**), one shared
  `runRefresh` code path for both `reason`s (**A8**), fire-once / re-arm (**A15**), tenant-prefix derived
  from the **verified** identity not the body (**A5**), isolated removal leaves C1 untouched (**A6**).
- **Boundary / failure harness (the negative control for high-stakes `A#`).** Prove the system FAILs the
  way the spec says it must: the **1-byte / 1¢ tax delta** must FAIL A2/A4; the **50%-discount mutation**
  on an unassigned account must FAIL A7; the **token sentinel** present in any artifact must FAIL A11;
  the **throttle bypass** (an IBKR call inside the window) must FAIL A9; a **failed pull that loses prior
  data** must FAIL A10.
- No suitable harness exists for an `A#` → **write one before testing**; an acceptance with no
  falsifiable outcome is not ready (escalate to PM + Architect as a spec gap). For W6 every `A#`
  reuses an existing harness shape, so none is unready — the lift deliberately keeps the *what we assert*
  identical to device and only swaps *where the bytes come from*.

## Cross-task integration coverage

> Per-task review proved each task in isolation (W6-M1's seam refactor; W6-M2's Lambda packaging).
> Integration proves the tasks **compose** — the seam between them and the use-cases that span both.
> This is where the milestone-level bug hides: a server store that conforms to `FactLogStore` but feeds
> the engine bytes that drift from the device path.

| use case / flow | tasks it spans | A# asserted end-to-end | integration test | seam exercised |
|-----------------|----------------|------------------------|------------------|----------------|
| `UC-HEALTH-4` (the headline) | W6-M1 → W6-M2 | A1, A2 (+ `domain-core.md` A3 replay re-run server-side) | `ServerVsDeviceReplayParityTest` (full pipeline: seam-swapped `LedgerStore` → server `FactLogStore` → `ProjectionEngine` → `CanonicalJSON` `TaxProjection` vs device golden) | `FactLogStore` (file ↔ server impl) → `ProjectionStore.loadOrRebuild` → `CanonicalJSON` |
| `UC-SCHED-1` | W6-M1 → W6-M2 | A8, A3, A4 | `SyncWorkerSharedPathTest` (`SyncWorker` shell wires server collaborators → `runRefresh` verbatim → server `FactLogStore` append → re-fold; `reason=.scheduled` then `.onDemand` produce the same result) | `TokenStore` (canned) → `FlexTransport` (fake) → `runRefresh` → `FactLogStore` → `SydneyMidnightBoundary` snapshot |
| `UC-HEALTH-1` | W6-M2 (+ read-API milestone) | A2, A12, A4 | `ServerReadStateParityTest` (the projection the read path would serve equals the device build's projection for the same facts; R8 holds) | `FactLogStore` → `ProjectionEngine` → `Output` `Codable` types (the thin-client contract) |
| `UC-SCHED-3` | W6-M2 | A8 (snapshot key), A4 | `SnapshotReconstructionDeterminismTest` (a fixed Sydney-midnight instant → the same persisted snapshot key + marks; overlay stays display-only, moves no tax cell) | `SydneyMidnightBoundary` → `YahooDailySnapshotProvider` (fake transport) → snapshot store (overlay lane, R8) |
| `UC-HEALTH-4` R8 leg | W6-M1 → W6-M2 | A4 | `ServerDailyPLR8RelockTest` (the device `DailyPLR8RelockTests`, re-plumbed through the server path: tax bytes identical with overlays on vs off) | overlay lane never enters `FactLogStore`/lot-ledger/tax inputs (the firewall) |

- Every W6 milestone **use case** has at least one integration test walking it end-to-end against its
  backing `A#`. `UC-CONN-1`, `UC-SCHED-2`, `UC-SCHED-4`, `UC-HEALTH-2` walk through
  A11/A8/A9/A13 and land fully when their owning milestones (secrets, schedule+trigger, read-API) open;
  their seam (`FactLogStore`, `ConnectionState`, `TokenStore`) is already integrated and asserted here.
- Exercise the **seams** named in `architecture.md` §11 / `system-design.md` §7.2/§9.1 — the swappable
  boundaries the original architecture pre-placed (`FactLogStore`, `TokenStore`, `ProjectionStore`,
  `FlexTransport`/`QuoteTransport`, the `FiredAlert` notifier, `runRefresh`). A contract honored by each
  side alone but broken **across** the seam (server store conforms but yields drifting bytes) is exactly
  what the per-task unit tests miss and what A2/A8 catch.
- Integration tests assert `A#`, same as acceptance — never a vague "it ran." Each names the `A#` /
  use-case it proves.

## Non-vacuity (mutation probe)

> A test that always passes proves nothing — and `ledger-qa-engineer-reviewer` will mutation-probe this plan.
> Pre-empt it: every test in the matrix must be shown to fail when the behaviour it guards is broken.
> Per D45, every new test is **RED-first + mutation-checked**; per D58, suites that mutation-probe the
> **same source file** run **serially** (the UIH-M26 race lesson) or each probe in its own worktree.

| A# | the mutation that turns its test RED |
|----|--------------------------------------|
| A1 | let `LedgerStore` bypass the `FactLogStore` protocol and touch the concrete store directly ⇒ seam test FAILs; or perturb one canonical key order ⇒ digest ≠ `7f6784e1…` ⇒ A1 FAILs |
| A2 | swap one `Decimal` for `Double` in the server wiring, or shuffle key order in `CanonicalJSON`, or change the fold order in the server path ⇒ a tax byte differs from the device golden ⇒ A2 FAILs; perturb the Div775 walk ⇒ ≠ −$1.76 ⇒ A2 FAILs |
| A3 | make `append` re-add an existing FactID (drop the dedup guard) ⇒ re-pull returns > 0 / factCount grows / digest changes ⇒ A3 FAILs |
| A4 | leak `snapshot.mark` / `currentFX` / `netCash` into a tax input on the server path ⇒ the two-worlds tax bytes diverge ⇒ A4 FAILs (the binding R8 negative control) |
| A5 | derive the storage prefix from a **client-supplied** `tenantId` instead of the verified identity ⇒ the forged-id request reads T2 ⇒ A5 FAILs |
| A6 | make remove-C2 also drop C1's facts/credential (the D83-T7 "wipe nukes everything" bug) ⇒ C1 factCount before ≠ after ⇒ A6 FAILs |
| A7 | default the unassigned account to a **50%** CGT discount ⇒ its disposals discount ≠ 0% ⇒ A7 FAILs (the D83-T5 hazard) |
| A8 | fork `.scheduled` and `.onDemand` into two code paths (not one `runRefresh`) ⇒ their ingest results / snapshot keys differ ⇒ A8 FAILs |
| A9 | let a throttled / locked invocation make an IBKR call ⇒ the fake-transport call-counter > expected ⇒ A9 FAILs |
| A10 | let a failed pull truncate the prior fact set or fabricate a zero ⇒ post-fail factCount ≠ pre-fail / state ≠ `.error` ⇒ A10 FAILs |
| A11 | log the token (or place it in an error type / response body / fact / export) ⇒ the grep sweep finds the sentinel ⇒ A11 FAILs |
| A12 | serve a non-`CanonicalJSON` body, or a type the client `Codable` can't decode to equality ⇒ round-trip ≠ direct projection ⇒ A12 FAILs |
| A13 | hardcode `nextScheduledAt` / mis-compute `staleness` ⇒ a field ≠ the store/instant ⇒ A13 FAILs |
| A14 | skip `dropAll()` on a material config write (stale cache served) ⇒ the 2nd-device read still sees the old result ⇒ A14 FAILs |
| A15 | emit a second `FiredAlert` while the rule is still crossed (no re-arm/de-dupe) ⇒ A15 FAILs |
| A16 | expose an update/delete-fact op, or return `allFacts()` out of canonical order ⇒ ordering / append-only assertion FAILs |

- A test that survives its mutation is vacuous — fix the test, don't widen the tolerance. **There is no
  tolerance to widen on tax bytes (A2/A4) — the boundary is zero.**

## W6 server-determinism (the lift's binding property)

> The W6-specific gate beyond ordinary coverage: **server compute is byte-identical to device compute**,
> and **R8 holds across the lift** (the server moves zero tax cell). This is the wave's reason to exist
> and the exit gate.

- **Server == device (A2, the bar).** The same real `.flex-local` facts + the same committed
  `ProjectionConfig`, replayed through the **server** `FactLogStore` and the **device** file store,
  produce a `CanonicalJSON` `TaxProjection` (EOY + CGT disposals + income + forex + `forexRealisations`)
  that is **byte-identical**, with the same `factDigest`, every CGT / franking / FX / FITO / Div775 /
  option cell matching the W1 hand-derived oracle (IND × SMSF, FY25 × FY26) — gated by the **same**
  oracle that gated W1. FY26 Div775 == **−$1.76 AUD** on the server path (D84 oracle). A 1-byte delta is
  a wave-blocking FAIL.
- **Replay/rebuildability holds across the seam (`domain-core.md` A3, re-run server-side; proven in the lift by A1+A2).** Drop all server projections + replay server-side ⇒
  byte-identical output (the device `RebuildabilityTests` re-plumbed onto the server store). The lift
  introduces no `Date()`/`TimeZone.current`/`Locale.current`/`UUID()`/randomness into any fold or its
  server wiring (R6); FactID stays the seed-free FNV-1a hash; money stays exact `Decimal`; JSON stays
  sorted-keys / fixed-Decimal.
- **R8 re-lock across the lift (A4, binding, non-vacuous).** The overlay lane (cash, daily-P/L,
  snapshot, quotes) still never enters the fact log, the lot ledgers, or any tax input — proven by the
  server-side `DailyPLR8RelockTests` with its mutation negative-control. The **only** intentional tax
  mover remains `Div775Engine` (D74/D84).
- **The determinism canary (production-side NFR1).** A scheduled job replays the committed golden
  `.flex-local` fixture server-side and alarms on **any** `TaxProjection` digest drift — the standing,
  in-production form of the A2 gate. Its assertion is the same exact-string digest equality; an alarm is
  a re-opened A2.
- **PortabilityLockTests stays green.** `LedgerCore` imports Foundation only (no UIKit/SwiftUI/SwiftData)
  — the construction-level guarantee that the core is server-liftable verbatim (R4). A new Apple-only
  import in the core FAILs this and blocks the wave.

## The Integrate-phase gate

> The milestone-level product gate. The CEO/tech-lead does not call W6-M1/W6-M2 (or the wave) `done`
> until this gate is green. Distinct from the per-task reviewer gates (`ledger-backend-engineer-reviewer`,
> `ledger-architect-reviewer`, `ledger-tech-lead-reviewer`) that already passed.

Gate is GREEN only when **all** hold:

- [ ] **Coverage complete** — every `A#` owned by the in-scope milestone(s) has at least one passing
  test in the matrix; no orphan `A#`, no orphan test. (W6-M1 owns A1, A7, A16, the `ConnectionState`
  seam under A9/A10; W6-M2 owns A2, A3, A4, A8, A11 + the server-side `domain-core.md` A3 replay/Portability re-runs. The
  later-milestone `A#` (A5/A6/A9/A10/A12/A13/A14/A15) carry committed fixtures + a written harness now,
  asserted fully when their milestone opens.)
- [ ] **Acceptance green** — every in-scope acceptance test passes against committed fixtures, offline,
  deterministically (re-run twice ⇒ identical result; the byte-identical harness makes this exact).
- [ ] **Integration green** — every in-scope W6 use case passes end-to-end (`UC-HEALTH-4`,
  `UC-SCHED-1`, `UC-HEALTH-1`, `UC-SCHED-3`); every named seam exercised
  (`FactLogStore`, `TokenStore`, `runRefresh`, `SydneyMidnightBoundary`, `CanonicalJSON`).
- [ ] **Server-determinism green (the wave bar)** — **A2** server tax bytes == device golden (exact
  string + digest `7f6784e1…`-class parity), the W1 oracle re-passes on the server path, FY26 Div775 ==
  −$1.76, the `domain-core.md` A3 replay/rebuild holds server-side, A4 R8 re-lock holds, `PortabilityLockTests` green.
- [ ] **Non-vacuity shown** — each test's mutation probe documented (table above) and confirmed
  RED-on-break (D45); same-file probes run serially (D58).
- [ ] **Reviewer sign-off** — `ledger-qa-engineer-reviewer` returned `PASS` on this test plan (tests truly
  assert the `A#`; integration is real; the byte-identical harness is exact, not tolerant). This is the
  agentic-integrity check on the product-QA work.

Then I emit the standard handoff (`verdict` form), and on any failure the gate is **not** waived for the
four founder prerequisites or a deadline:

```json
{"persona":"ledger-qa-engineer","task_id":"W6-M2","summary":"W6 Integrate gate: A# coverage, server-vs-device byte-identical replay, R8 re-lock, idempotent ingest.","files_changed":["Tests/Backend/ServerVsDeviceReplayParityTest.swift","Tests/Backend/ServerIngestIdempotencyTest.swift","Tests/Backend/ServerDailyPLR8RelockTest.swift","Tests/Backend/SyncWorkerSharedPathTest.swift","Tests/Backend/TokenNeverLeaksTest.swift","Tests/Fixtures/golden/server_tax_projection/","Tests/Fixtures/div775_oracle_fy26.json"],"decisions":["A2 bound to the byte-identical harness against the device golden; Div775 bound to the −$1.76 first-principles oracle (D75), never the projection's own sum.","Single-tenant cut: A5/A6 fixtures committed now, isolation invariant exercised by construction (prefix from verified identity)."],"open_questions":["None"],"verdict":"PASS | FAIL","blocking_issues":["<concrete, per-A# on FAIL>"],"status":"DONE | BLOCKED"}
```

- **FAIL is per-`A#` and concrete** — "A2 FAILs: server `TaxProjection` differs from the device golden
  by 3 bytes in the FY26 Div775 line on the real `.flex-local`" — never a soft pass with caveats.
- A milestone with an uncovered or failing `A#` ships `BLOCKED`, not a fake `DONE`. The four W6 founder
  prerequisites (AWS account/region/budget, IBKR token re-issued for unattended use, APNs key +
  `aps-environment`, Path-A + price-feed confirmation) gate the **deploy / live-push** legs (A8's deploy
  leg, A15's live APNs) — but the **offline** acceptance (A1–A16 as written, fake transport) is **not**
  gated on them and must be green regardless (spec OD6; `milestones/W6.md`). Deadlines do not lower the
  bar (`handoff-and-review.md`).
- When a defect traces to a recurring producer behaviour (e.g. an oracle that re-sums the projection
  instead of re-deriving — the DPC cash-bug lesson, D75), append a note to that persona's `MEMORY.md`
  for `ledger-hr` to promote (behaviour → SKILL.md, gotcha → knowledge/, fact → spec).

## Definition of done

- [ ] A#-to-test matrix complete — every in-scope W6 `A#` mapped, every test names its `A#`; the
  later-milestone `A#` carry committed fixtures + a written harness.
- [ ] Fixtures committed, deterministic, offline, scrubbed (token never in any committed byte, D31);
  the device-output goldens are reviewed artifacts.
- [ ] Each `A#` bound to a named harness (byte-identical / oracle / property / boundary).
- [ ] Cross-task integration covers every in-scope W6 use case and exercises the `architecture.md` §11
  seams (`FactLogStore`, `TokenStore`, `ProjectionStore`, `runRefresh`, `SydneyMidnightBoundary`,
  `CanonicalJSON`).
- [ ] Non-vacuity mutation probe documented per test; same-file probes run serially (D58).
- [ ] W6 server-determinism proven: A2 server == device golden (exact bytes + digest), W1 oracle
  re-passes server-side, the `domain-core.md` A3 replay/rebuild holds, A4 R8 re-lock holds, `PortabilityLockTests` green.
- [ ] Integrate-phase gate run; `ledger-qa-engineer-reviewer` PASS; handoff emitted with binary `status`.
