# <Company> System Design — <next-phase / wave name>

> Owned by the **Tech Lead**. The **"can we build & run it?"** document for the next deployment phase. Reviewed by `tech-lead-reviewer` for delivery soundness, and consulted with the **Architect** for design-fit (the architecture↔system-design seam is a consultation edge, not a merge).
>
> **Distinct from `architecture.md`.** Architecture answers *"is it right?"* — invariants the system enforces, seams, the current-state shape. This answers *"can we operate it?"* — the topology, identity, API surface, client changes, and migration we have to stand up and run. Where this design leans on an invariant, **point to `architecture.md`/`specs/`** rather than restating it. Where this design needs a new invariant, that is a **consultation back to the Architect**, logged below.
>
> Version-stamped. Scoped to one wave/phase; supersedes the prior wave's design note. Open decisions that are irreversible / external / strategic / cost-incurring are **flagged for the founder to settle in `review-and-plan`** (`decision-escalation.md`) — built behind a flag on a signed default so the run proceeds without a mid-run stop, never silently invented.

**Phase:** <e.g. M4 — server-lift & multi-device sync>
**Backing specs:** <R#/A# subsystems this phase delivers — links into `specs/`>
**Status:** draft | in-review | approved | superseded

---

## 1. Purpose & goals
<Two or three sentences: what this phase makes possible that the current shape cannot, and why now. Tie to the milestone success criteria.>

**Goals (this phase delivers):**
- <observable capability — e.g. "a second device sees the same ledger within N seconds, offline-first">
- <…>

**Non-goals (explicitly out of scope — name them so reviewers don't expect them):**
- <e.g. "no real-time collaborative editing; last-writer-wins is acceptable this phase">
- <e.g. "no public/third-party API; the API is internal client↔server only">

---

## 2. Target architecture (component diagram)
> The runnable topology after this phase ships: every deployable unit, every trust boundary, every hop. ASCII. Mark what is **new this phase** vs existing.

```
┌──────────────┐      <protocol>       ┌──────────────────┐
│  <client>    │ ───────────────────►  │   <API / edge>   │   ← NEW this phase
│  (existing)  │ ◄───────────────────  │                  │
└──────────────┘      <auth header>    └────────┬─────────┘
                                                 │
                                       ┌─────────▼─────────┐
                                       │  <service / core> │
                                       └─────────┬─────────┘
                                       ┌─────────▼─────────┐
                                       │   <store>         │   ← NEW this phase
                                       └───────────────────┘
```

| component | new/existing | responsibility | runs where |
|-----------|--------------|----------------|-----------|
| <client> | existing | <…> | <device> |
| <API edge> | NEW | <…> | <host / platform> |
| <service> | NEW | <…> | <…> |
| <store> | NEW | <…> | <managed / self-hosted> |

**Trust boundaries:** <where the network/process/tenant boundary sits; what crosses it and in what form.>

---

## 3. Data & storage
> What is persisted *for operations* — not the domain shapes (those live in `data-model.md`; link, don't restate). Cover where bytes live, how they're shaped on the wire/at rest, and the durability story.

- **System of record:** <which store holds the truth; pointer to `data-model.md` for the shapes>
- **Storage engine & why:** <e.g. SQLite→Postgres; an internal/reversible call — log in `decisions.md`>
- **On-wire / at-rest format:** <serialization, versioning, ordering rules — point to the invariant in `architecture.md` if one governs it>
- **Durability / backup / retention:** <snapshots, PITR, retention window>
- **Tenancy / isolation:** <single-tenant vs row-scoped vs DB-per-tenant; how a query can't cross tenants>

---

## 4. Identity, auth & secrets
> Who is allowed to do what, and how the system proves it. Most of this is **founder-flagged** (external surface, cost, or strategic) — stage, don't decide.

- **Identity model:** <accounts/devices/users; how a principal is established>
- **AuthN:** <token type, issuer, lifetime, refresh; how the client obtains and presents credentials>
- **AuthZ:** <what scopes/roles gate which operations; default-deny stance>
- **Secrets handling:** <where keys/tokens live — env, secrets manager, Keychain on device; rotation; what is NEVER in the repo or logs>
- **Transport security:** <TLS expectation, pinning if any>

> **Founder flags here** (see §9): an auth provider choice, a paid identity service, or any user-facing sign-in vocabulary are escalation-class — propose a default, stage behind config.

---

## 5. Sync subsystem
> The heart of most next-phase lifts. How divergent local state converges, offline-first, without silent loss. The conflict policy is an **invariant-bearing** decision — consult the Architect.

- **Sync model:** <push/pull, polling vs streaming, change-feed vs full-pull>
- **Conflict resolution:** <LWW / CRDT / merge rules — state it precisely; this maps to an `A#` and must be testable>
- **Ordering & identity:** <how changes are ordered (vector clock / monotonic version / timestamp) and deduped (idempotency keys)>
- **Offline behavior:** <what works offline, what queues, how the queue drains and bounds>
- **Failure & resume:** <partial-sync, retry/backoff, poison-message handling; what a crash mid-sync leaves behind (must be recoverable — point to a determinism/replay invariant if one applies)>

**Acceptance hooks:** <the `A#` that prove sync is correct — e.g. "two clients, concurrent edit ⇒ deterministic converged state (test)">

---

## 6. API design
> The contract between client and server for this phase. Even if internal-only, write it as a contract the executor implements and QA tests against.

| method | path | purpose | auth | request → response (shape) | backing R#/A# |
|--------|------|---------|------|----------------------------|---------------|
| <GET>  | </…> | <…>     | <scope> | <… → …>                 | R#, A#        |
| <POST> | </…> | <…>     | <scope> | <… → …>                 | R#, A#        |

- **Versioning:** <path/header version; how a v1 client survives a v2 server>
- **Errors:** <error envelope; status codes; what's retryable vs fatal>
- **Idempotency:** <which writes carry an idempotency key; replay semantics>
- **Compatibility contract:** <backward/forward compat window the client relies on>

> Any **public/third-party** API surface or user-facing naming is **founder-flagged** (§9). Internal client↔server contracts are the Tech Lead's to decide and log.

---

## 7. Client changes
> What the existing client must change to use the new topology. This is the bridge the migration walks across.

- **New capabilities:** <sync engine, token store, retry/queue layer>
- **Changed flows:** <which use-cases (UC-…) gain network/offline states — coordinate with UX manuals>
- **Local migration:** <on-device schema/store changes; how an updated app reads old local data>
- **Graceful degradation:** <behavior with no network, expired token, server down — the app must stay usable offline-first>
- **Backward compat:** <can an un-updated client coexist during rollout? for how long?>

---

## 8. Migration plan
> The step-by-step from today's running system to the target, **reversible at every step**. This is where "can we run it" is won or lost: each step is independently shippable, observable, and has a rollback.

| step | action | reversible? | rollback | verify (observable) |
|------|--------|-------------|----------|---------------------|
| 1 | <stand up new store, no traffic> | yes | <tear down> | <healthcheck green> |
| 2 | <dual-write / backfill> | yes | <stop dual-write> | <row counts match> |
| 3 | <flip read path behind flag> | yes | <flip flag back> | <parity check passes> |
| 4 | <cut over / retire old path> | <…> | <…> | <…> |

- **Data backfill:** <one-time vs continuous; how integrity is proven (the parity/reconciliation check, ideally an `A#`)>
- **Cutover strategy:** <flag-gated, canary, all-at-once; who/what triggers it>
- **Rollback trigger:** <the concrete signal that says "roll back now">
- **Zero/low-downtime story:** <or the accepted maintenance window, and who approves it>

---

## 9. Risks & open decisions
> Two lists. **Risks** = things that could go wrong, with a mitigation. **Open decisions** = forks — each tagged decide-now (logged) or **FOUNDER** (front-loaded into `review-and-plan` per `decision-escalation.md`: irreversible / external / strategic / costs money / locks future waves). FOUNDER items are pre-decided or given a default + tolerance and **built behind a flag** so the run is never blocked waiting; an unforeseen one is decided against its default and flagged `DIRECTION`, never a mid-run stop.

**Risks**
| risk | likelihood | impact | mitigation / detection |
|------|-----------|--------|------------------------|
| <e.g. backfill diverges under load> | <med> | <high> | <parity check gates cutover; rollback on delta> |

**Open decisions**
| id | decision | recommendation (staged default) | owner | escalate? |
|----|----------|---------------------------------|-------|-----------|
| OD1 | <internal: store engine> | <Postgres> | tech-lead | decide-now → log in `decisions.md` |
| OD2 | <auth provider — paid / external> | <provider X, behind config flag> | founder | **FOUNDER** — cost + external surface |
| OD3 | <user-facing sync naming / API exposure> | <keep internal this phase> | founder | **FOUNDER** — externally visible |

> Every **FOUNDER** row must be staged behind an abstraction/flag (one-line flip on answer) and must not block the rest of the phase. Batch them into the CEO's founder pre-flight; a fork discovered mid-run is a frontloading miss — note for HR.

---

## 10. Decision & consultation pointers
- **Logged decisions:** <links to the `decisions.md` D# entries this design made>
- **Architect consultations:** <the seams/invariants raised back to the Architect — e.g. "sync conflict rule needs an `A#`; spec R# impacted">
- **Spec impact:** <any R#/A# this phase adds or changes — co-owned with PM + Architect>
