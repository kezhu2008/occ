# Backlog — W6 backend lift: proposed server-era use cases

**Register: PROPOSED (server-era — nothing here is built).** This is the W6 future-state use-case
backlog for the AWS backend lift (see `../specs/backend-lift.md`, `../system-design.md`,
`../backend-migration-review.md`). Designed around the new user reality — the server syncs unattended,
the phone just **observes**. Nothing here enters the canonical catalog (`use-cases/index.md`) count
until it ships **and** is QA-verified; the canonical IDs already exist as `(planned — W6)` rows there.

**PM owner:** `ledger-product-manager` · **Date:** 2026-05-31 · **Recommended build chain:**
`ledger-tech-lead → ledger-backend-engineer → ledger-backend-engineer-reviewer → ledger-qa-engineer`.
**Reviewed by:** `ledger-pm-reviewer`.

> **Steps here are intended user flows, not tap-by-tap** — the W6 UI doesn't exist yet, so exact
> labels/screens are the designer's to set. Each entry's **Acceptance criteria** is the testable
> contract and **Backed-by spec** cites the `backend-lift.md` `A#` (the `[BL]` namespace) that QA
> asserts against. `Source` is "(to be implemented)" throughout. These rows correspond to the
> canonical `(planned — W6)` IDs in `use-cases/index.md` §6–§7 — this file is the demand, that file
> is the index.

---

## 0. The three server-era shifts (why these use cases exist)

| Shift | Today (on-device, W1–W5) | After the W6 lift |
|-------|--------------------------|--------------------|
| **Sync inverts** | The user drives it: "Sync now", the app must be open, the ingest is staged on the device (UC-SYNC-5). | The **server** pulls Flex on a schedule + on demand with no phone open; the user **observes freshness** and *optionally* nudges a refresh. |
| **Connections appear** | "Connect" = paste a token onto this one device, one credential (UC-SYNC-1/2). | A server-held, encrypted connection that syncs **unattended**; N connections per tenant, each isolated. |
| **Health is a read surface** | The device ledger-state/digest panel (UC-SYNC-7). | A server-sourced sync-health view: per-connection status + an honest "last synced" freshness stamp. |

---

## 1. Proposed use cases

## UC-CONN-1
**Add a server-managed IBKR connection (server syncs unattended)** *(PROPOSED — status: proposed; 2026-05-31)*

**Goal:** The user links an IBKR account so the **server** syncs it automatically from then on — no
phone open, no "Sync now" button machine.
**Actor / preconditions:** A signed-in user (or, single-tenant Path A, the device owner) with a Flex
token + Query ID.
**Entry point:** Thin client → Connections → "Add IBKR connection" (carries the existing
"how to get your Flex token" helper).
**Steps (intended):**
1. Open the add-connection screen.
2. Enter the Flex token + Query ID.
3. Submit → the credential is sent to the server and stored in the secret store (encrypted, **never**
   kept on the device, never shown back).
4. The server runs the first sync **asynchronously**; the screen shows an honest "We're importing your
   portfolio — we'll update when it's ready" state (not a device-side staged-ingest spinner).
5. When the first server sync completes, the portfolio populates.
**Outcome:** A server-side connection exists and syncs on a schedule; the portfolio appears once the
first server sync finishes; the device holds no credential.
**Variations & edge states:** Invalid/expired token → the server reports a clear, **token-free** reason
and the connection shows "needs attention". First sync still running → an honest "importing…" state,
never a fabricated zero portfolio. The token is never displayed back and never appears in any log /
error / API response / export.
**Acceptance criteria:**
- A valid token results in the server syncing the account **unattended** thereafter — verified by
  querying the read surface with the **app closed** and seeing facts/positions the device never
  touched.
- The token is stored **server-side only**, is never persisted on the device, never shown back, and is
  **grep-clean** across every produced artifact.
- A bad token surfaces a clear reason + a re-authorize path without exposing the token.
**Backed-by spec:** `backend-lift.md` **A11** (`token-never-leaks`) [BL], **A8**
(`unattended-scheduled-sync`) [BL], **A3** (`idempotent-server-ingest-no-cursor`) [BL].
**Source:** (to be implemented) — replaces on-device UC-ONB-1 + UC-SYNC-1.

## UC-SCHED-1
**Scheduled sync (server pulls Flex with no phone open)** *(PROPOSED; 2026-05-31)* — **NEW**

**Goal:** The portfolio stays current on its own — the server pulls Flex on a schedule, so the user
never has to open the app to sync.
**Actor / preconditions:** A tenant with ≥ 1 healthy connection.
**Entry point:** Server cron (EventBridge) → the `SyncWorker` sync handler → `FactLogStore` (no user
action; the user later just **observes** the result).
**Steps (intended):**
1. On the schedule (e.g. hourly on market days + an end-of-day Sydney-midnight snapshot), the server
   invokes the sync handler with `reason=.scheduled`.
2. The handler runs the **same** `runRefresh` body the on-demand path uses, pulls the Flex statement
   (SendRequest → poll → GetStatement), reconstructs the Sydney-midnight quote snapshot, appends any
   genuinely-new facts to the server store, and re-folds projections.
3. The next time the user opens the thin client, the data is already fresh — with a server-sourced
   "as of" stamp.
**Outcome:** Facts/positions/tax update with the app fully closed; the user observes a current
portfolio they never had to sync.
**Variations & edge states:** Re-pull of the same statement is **idempotent** — 0 newly-persisted
facts, digest unchanged, no cursor (D-no-cursor). A second pull on the same Sydney day leaves the
daily snapshot **frozen**. The IBKR per-token rate limit (error 1018) is respected by a per-connection
throttle. A failed pull leaves prior data intact + a token-free error and flags the connection.
**Acceptance criteria:**
- The sync handler wired with server collaborators (canned credential, fake transport returning the
  real recorded bytes, fixed clock) invoked `reason=.scheduled` then `.onDemand` runs the **same**
  `runRefresh` body to the **same** ingest result; the chosen instant flows through
  `SydneyMidnightBoundary` to a **deterministic** snapshot key.
- Re-ingesting the same statement bytes returns **0 newly-persisted** and an unchanged digest (no
  since-date/cursor); a different statement with overlapping rows adds only genuinely-new FactIDs.
- The result is verified with the app **closed** (the unattended bar).
**Backed-by spec:** `backend-lift.md` **A8** (`unattended-scheduled-sync`) [BL], **A3**
(`idempotent-server-ingest-no-cursor`) [BL].
**Source:** (to be implemented) — replaces device-driven UC-SYNC-5/6 (demoted to an optional nudge).

## UC-HEALTH-2
**Sync-health view (per-connection status + freshness)** *(PROPOSED; 2026-05-31)* — **NEW**

**Goal:** See, per connection, whether the server is keeping data fresh and exactly when it last
synced — so the user trusts that syncing is handled for them.
**Actor / preconditions:** A signed-in user with ≥ 1 connection.
**Entry point:** Thin client → Sync health (the read surface that replaces the device ledger-state
panel).
**Steps (intended):**
1. Open Sync health → see each connection with a status (Active / Syncing / Needs attention) and a
   server-sourced "last synced <time>" / "as of <timestamp>" stamp.
2. Read the freshness stamp on every data screen; a slipped schedule / failing connection goes amber
   with an honest "last synced <time>".
3. Tap a connection for detail (last successful sync, last error if any — token-free).
**Outcome:** The user understands each connection's health at a glance and trusts the freshness, with
no manual sync needed.
**Variations & edge states:** Stale data is **labeled honestly** (amber + age), never presented as
current and never a fabricated zero. The status is read-only and tenant-isolated — a request
authenticated as one tenant sees **only** that tenant's connections.
**Acceptance criteria:**
- Each connection shows an **accurate** status + last-sync time **sourced from the server**; a failing
  connection is clearly flagged and stale data is amber-labeled with its age.
- Freshness is derived server-side `(asOf − lastPullAt)`; a connection past its stale threshold reports
  `staleness = stale` (not hidden as current).
- The view is tenant-isolated: a T1-authenticated request returns only T1's connection state and
  leaks **0** bytes of any T2 resource.
**Backed-by spec:** `backend-lift.md` **A13** (freshness + SyncHealth fields) [BL], **A5**
(`multi-tenant-isolation`) [BL].
**Source:** (to be implemented) — replaces the device-centric UC-SYNC-7 ledger-state view.

---

## 2. Open product decisions (boss)

1. **Scope — single-tenant (Path A) vs multi-tenant SaaS (Path B).** The headline fork. Path A lets
   us degrade auth to a device-pinned sign-in and drop the full account journey; Path B needs the
   account + privacy/delete surface. W6 is Path A for now (`use-cases/index.md` coverage note); the
   `[MT]` account use-cases stay in the broader `backend-lift.md` backed-use-case set until Path B is
   specced.
2. **First-sync experience (UC-CONN-1):** async "we'll update when ready" vs holding the user on a
   progress screen until the first server sync finishes.
3. **Manual-refresh prominence:** keep a visible "refresh now" affordance on Sync health, or lean
   fully on the schedule (UC-SCHED-1) and hide it behind pull-to-refresh? (Affects how much the sync
   inversion is felt.)

---

## 3. Recommendation

The reading product is safe — it carries over unchanged (`backend-lift.md` §"Backed use-cases"). The
W6 build is the **connection + scheduled-sync + sync-health** journey above. Recommend the boss settle
the **scope fork (Path A vs B)** first, then hand this set to `ledger-tech-lead` to fold into the W6
wave (`milestones/W6.md`, `tasks/W6-M1.md` → `tasks/W6-M2.md`) and decompose. None of this enters
`use-cases/index.md`'s count until each piece ships and `ledger-qa-engineer` verifies it against the
named `A#`.
