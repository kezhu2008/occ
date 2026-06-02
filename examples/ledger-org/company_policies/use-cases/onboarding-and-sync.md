# Instruction Manual — Onboarding & Sync

Tap-by-tap user instructions for the Onboarding + Reconcile use cases the app supports, plus the
W6 add-a-server-connection flow (PROPOSED — gated on the boss confirming Path A of the backend
lift). Grounded in the live code and the W6 backend-migration proposal — not the stale
`ui-review/UI-MANUAL.md`. Index: [`use-cases/index.md`](index.md). Backing contracts (R#/A#):
[`company_policies/specs/domain-core.md`](../specs/domain-core.md) (on-device) and
[`company_policies/specs/backend-lift.md`](../specs/backend-lift.md) (the W6 server lift, `[BL]`).

> **Post-D85 note:** the shipping app has **no demo/sample-data path**. There is no "SAMPLE DATA"
> chip, no "USE DEMO CREDENTIALS" button, and no "clear sample data" action. A fresh install goes
> straight to **CONNECT IBKR**; a no-token launch lands on an honest-empty real store. "Looking
> around without connecting" is **not** a shipped use case (demo creds are dev/CI-only).

> **Reconcile vs onboard.** Onboarding (UC-ONB-1) *builds* the ledger from a Flex pull. Reconcile
> (UC-SYNC-7) *proves* that the built ledger ties out to the broker before any tax number is
> trusted — it never mutates facts. The two are distinct goals with distinct see-thats; keep them
> separate.

---

## UC-ONB-1
**First-run onboarding via Flex (live connect)**

- **Goal:** Connect a real IBKR account on first launch and build the ledger from a live read-only Flex pull.
- **Actor / preconditions:** A first-run user (not `onboarded`), or anyone who re-onboarded. Needs a read-only Flex token + Query ID from IBKR (Activity Flex query, from inception, XML). The flow owns the whole screen — no tab chrome.
- **Entry point:** App launch (pre-app) → OnboardingFlow → Welcome → Choose Source → Flex Credentials → Ingesting → Ready.
- **Steps (tap-by-tap → see-that):**
  1. On the Welcome screen ("A ledger. Not a dashboard."), tap **CONNECT IBKR**. → the progress header reads "STEP 2 / 5", "01 · CHOOSE SOURCE".
  2. Tap the **IBKR Flex Web Service** card (tagged "LIVE · RECOMMENDED"), then **NEXT**. → "02 · FLEX CREDENTIALS".
  3. Paste into **FLEX TOKEN** and **QUERY ID**. → the live hint flips to green "Looks good — ready to pull." and **INGEST** enables (token ≥ 8 chars, queryId ≥ 5).
  4. Tap **INGEST**. → "03 · INGESTING · FLEX"; a staged list animates ("TLS handshake · ndcdyn.interactivebrokers.com" … "Reconcile → facts" … "Fold → positions · tax · optimise") with a spinner + "n/N · pct%" counter.
  5. On success the footer flips to "✓ READY" and auto-advances (~0.7s) to "✓ PROJECTION READY / Your ledger is built." with a SUMMARY card (MARKET VAL, UNREALISED, "N facts ingested", "X share trades · Y option events").
  6. Tap **OPEN PORTFOLIO →**. → the tab app opens on POSITIONS; the IBKR FLEX card reads "● CONNECTED / LIVE".
- **Outcome:** Real portfolio ingested into the append-only fact log at `…/Ledger/real`, deduped on IBKR stable ID; credential saved to Keychain; SourcePreference = .real (survives relaunch); `onboarded` = true. A second launch that re-pulls the same statement yields an identical fact count (idempotent).
- **Variations & edge states:** Pull fails (auth / query / rate-limit / IBKR-generating / network / server) → the failing stage marked "FAILED", a red error card with a token-free reason + RETRY / BACK (UC-ONB-4). A transfer-in with missing cost basis → a red "⚠ N COST-BASIS GAP(S) DETECTED" card on the Ready screen (resolved later via the GAPS inbox). No token / locked Keychain on a later real-intent launch → honest-empty real store, never demo holdings (D85).
- **Backed by spec:** R1, R2, R12, R24, R39, A1, A4, A5 (index anchors for UC-ONB-1: R24, R39, A1; plus the append-only/idempotent-ingest invariants the build relies on).
- **Source:** OnboardingFlow.swift:211 + OnboardingSteps.swift:272 + OnboardingViewModel.swift:263 + AppContainer.swift:889 (connectReal, origin .onboarding).

## UC-SYNC-7
**View sync state + reconcile to the cent (confirm the ledger ties out to the broker before trusting any report)**

- **Goal:** Prove the freshly-built ledger reconciles to the IBKR statement — net cash per currency, position quantities, and the FY fact-date span all tie out — before any tax or valuation number is acted on.
- **Actor / preconditions:** An onboarded, connected user (`flexConnected == true`) with at least one completed Flex pull. Reconcile is read-only: it asserts over the existing fact log and projections, it never appends or mutates a fact (R2 append-only stays intact).
- **Entry point:** SYNC → STATE → LEDGER STATE panel (the as-of tie-out), with the per-currency cash check on POS → CASH and the gap check on SYNC → GAPS.
- **Steps (tap-by-tap → see-that):**
  1. SYNC → **STATE**. → a LEDGER STATE panel "N / N FY YEARS STITCHED" with a StitchBar, "OLDEST <date>" / "NEWEST <date>" from real fact dates, and a green "HISTORY COMPLETE" rule when the multi-month statements stitch with no missing FY.
  2. Read the **LEDGER · APPEND-ONLY** summary. → Trades, Cash flows, Corporate actions, Option events, and Cost-basis fills each counted, plus "Total facts · immutable · DIGEST <8-char>…". The digest is stable across re-reads of the same fact set (the A6 replay property — same facts ⇒ byte-identical projection).
  3. Switch scope to the connected entity, then POS → **CASH**. → the derived cash balance per (entity, account, currency) — e.g. "AUD +93,627.88" and a signed "USD −47,376.22" (a negative/margin FX balance renders, never clamped to 0) — which the user eyeballs against the broker's cash report.
  4. SYNC → **GAPS**. → "No unresolved gaps — your cost-base is honest" when every parcel has a basis; otherwise the HIGH/MED/LOW severity groups list each unresolved gap (a transfer-in with no basis is the canonical one) with its TAX IMPACT, so the user knows the tie-out is incomplete until resolved (branch to UC-SYNC-10 resolve).
- **Outcome:** The user has confirmed the ledger ties out — FY span complete, fact digest stable, per-currency cash matches the broker, zero unresolved gaps — and can trust POS / TAX. No state is written: reconcile is an assertion, not a mutation.
- **Variations & edge states:** A ~1-week-old account shows 1 FY stitched (not a defect). Empty real ledger → W1 placeholder span, FY count 1, "No ledger state". An unresolved cost-basis gap → tie-out is INCOMPLETE; the GAPS badge is non-zero and any CGT touching that parcel is provisional until the gap is filled. A multi-month stitch that skips an FY → the StitchBar shows the missing year and "HISTORY COMPLETE" does not appear.
- **Backed by spec:** R2, R8, R26, A3, A5, A24 (index anchors for UC-SYNC-7: R26, A24, A3 — reconcile-by-Decimal-exact-equality A24, byte-identical-replay A3; plus append-only A5/R2).
- **Source:** SyncState.swift:135 + SyncViewModel.swift:201 / :219 / :538 + CashSection.swift:21 + GapsInbox.swift:40.

## UC-CONN-1
**Add a server-managed IBKR connection (PROPOSED — W6 Path A)**

> **PROPOSED / not yet built.** This use case opens only when the boss confirms **Path A** of the
> backend lift (single-tenant always-on backend). Steps below are the *intended flow* — exact labels
> are not final. Under Path A the LedgerCore lifts verbatim into AWS Lambda behind a `FactStore`
> protocol; the IBKR token moves from the device Keychain to AWS Secrets Manager (one secret per
> connection, lifting today's single-item Keychain blocker), and an EventBridge cron pulls
> unattended. The phone becomes a thin read client. Acceptance is binding: tax numbers must be
> **byte-identical** to the on-device build on the same real statement.

- **Goal:** Register an IBKR Flex connection that a server pulls on a schedule, so the ledger stays current without the phone being open — and confirm the server-built ledger matches the on-device one to the cent.
- **Actor / preconditions:** An onboarded user who has confirmed Path A. Has a read-only Flex token + Query ID. The backend (SyncWorker Lambda + DynamoDB fact store partitioned by `tenant#connection` + Secrets Manager + the read API behind the seam) is provisioned. Multiple connections are supported (the W6 multi-connection store, unlike the device's single-item Keychain).
- **Entry point (intended):** SYNC → SOURCES → "SERVER CONNECTIONS" → ADD SERVER CONNECTION → enter Flex credentials → confirm.
- **Steps (action → see-that):**
  1. SYNC → SOURCES → **SERVER CONNECTIONS** → tap **ADD SERVER CONNECTION**. → an entry sheet with FLEX TOKEN + QUERY ID and the same "▸ HOW DO I GET THESE?" 3-step IBKR helper as onboarding (UC-ONB-1 step 3), plus a note that the token is stored server-side in Secrets Manager, not on the phone.
  2. Enter token + queryId and confirm. → the credential is written to its own Secrets Manager secret keyed by connection; the connection appears as "● PROVISIONING" then "● CONNECTED · SERVER" with a "next scheduled pull <time>" line. The device Keychain is NOT used for this connection.
  3. Wait for (or trigger) the first server pull. → the SyncWorker Lambda reads the token from Secrets Manager, runs the verbatim LedgerCore reconcile, and appends facts to the DynamoDB store (idempotent put — unique on `(tenant, connection, factId)`, the server analogue of the device dedupe). The thin client then shows the same POS / TAX read models, now sourced from the server.
  4. Reconcile the server ledger against the on-device baseline (UC-SYNC-7). → the per-currency cash, position quantities, and the FY fact-date span tie out, and the tax cells (EOY + CGT disposals) are byte-identical to the on-device build for the same statement — the W6 acceptance gate.
- **Outcome (intended):** A server-managed connection exists; its credential lives in Secrets Manager (one secret per connection); EventBridge cron pulls it unattended on a schedule plus an on-demand "sync now" trigger; facts land in the multi-tenant DynamoDB store behind the `FactStore` protocol; the phone reads server state over the thin API. The on-device single-tenant path remains available and unchanged.
- **Variations & edge states:** Path A not yet confirmed → the SERVER CONNECTIONS section is hidden and only the on-device flows (UC-ONB-1 / UC-SYNC-*) are reachable. Provisioning fails (bad token / Secrets Manager write error) → "● FAILED" with a token-free reason + RETRY; no partial connection is left. Scheduled pull fails (auth / rate-limit / network) → the connection shows "last pull failed · <reason>", the prior server facts stay intact (append-only, never wiped), and the next cron tick retries. Tax cells differ from the on-device baseline on the same statement → a BLOCKING acceptance failure (the lift is "a lift, not a rebuild"); not shippable until byte-identical.
- **Backed by spec (`backend-lift.md` namespace, tagged `[BL]`):** R1, R17, A1, A2, A11 [BL] (index anchors for UC-CONN-1: R1, R17, A11 [BL] — FactStore seam R1, server-held write-only credential R17, token-never-leaks A11; plus the wave bar A2 `server-tax-byte-identical-to-device` and seam-preserves-replay A1). These resolve in [`company_policies/specs/backend-lift.md`](../specs/backend-lift.md), NOT in `domain-core.md` — the same raw numbers mean different things across the two files.
- **Source:** (to be implemented — W6 Path A; FactStore protocol seam + SyncWorker Lambda + Secrets Manager TokenStore + EventBridge schedule + thin read API).
