# Decision Escalation — front-loaded, never a mid-run stop

The CEO's mandate is to **minimize founder intervention** while staying *trustworthy*. The company
is long-running: the founder's decision weight is paid **up front** in `review-and-plan`, and the
run itself **never hard-stops to ask**. Escalation here means *front-loading* a class of decision
into the signed plan — not parking the run mid-flight.

## The classification (what decides where)

A decision is **direction-changing** when it is any of:
- **Irreversible** or expensive to reverse once shipped/announced.
- **Externally visible** — public API, user-facing naming, brand vocabulary.
- **Strategic** — pricing, business model, scope, public positioning.
- **Spends money** or incurs ongoing cost.
- **Locks future waves** — architecture that constrains decisions the founder hasn't made.

Everything else is **reversible / internal / low-blast-radius** → decide autonomously and log.

## How each class is handled (no hard stop in any of them)

- **Reversible / internal** → decide, bind to its acceptance harness, log in `decisions.md`.
- **Direction-changing, foreseeable** → it must already be settled in the signed plan
  (`review-and-plan` enumerated it and the founder pre-decided it or set a default + tolerance).
  The run reads the plan's answer and proceeds.
- **Direction-changing, unforeseen (surfaced mid-run)** → the CEO decides against its best-judgment
  default, logs it `DIRECTION`-tagged, and **keeps running**. If two options both satisfy the
  plan, it picks the one cheapest to reverse. If that decision (alone or cumulatively) bends a
  charter non-negotiable or a roadmap success criterion, the CEO emits a **drift digest** and
  runs on — see `automation.md` §2b.

The old "escalate = stop and wait" gate is gone. The founder course-corrects at the next
`review-and-plan`, informed by the DIRECTION log and any drift digest — the run does not block on
that answer.

## Worked examples

| Decision | Verdict | Why |
|----------|---------|-----|
| Postgres vs SQLite for internal local storage | **Decide now**, log it | Internal, reversible, pure engineering. Deciding it *is* reducing intervention. |
| Rename public "Account" → "Wallet" | **Front-load in the plan** | Public/documented API + brand + semantic shift. Settle it in `review-and-plan`; if it surfaces mid-run unforeseen, keep "Account" (the cheapest-to-reverse default), log `DIRECTION`, run on. |
| Launch price $0 vs $9/mo | **Front-load in the plan** | Core business model. The founder sets price (or a default + tolerance) at planning time; the run never invents a price unasked, but never stalls either — it ships behind a configurable value. |
| OpenAPI vs gRPC for an internal service | **Decide now**, log it | Reversible, internal. |
| Choose a paid third-party API with monthly cost | **Front-load in the plan** | Spends money. The spend ceiling is a planning decision; mid-run the CEO stays within the signed ceiling and logs, or defers the thread if no ceiling covers it. |

## The autonomy trap, resolved by front-loading

"Reduce founder intervention **as much as possible**" must not become "auto-pick a launch price
while the founder sleeps." The resolution is *when*, not *whether*: direction-changing calls are
made **with the founder, in planning** — pre-decided or given a default + tolerance — so the run
already holds the answer. A mid-run fork is then either (a) covered by that standing authority, or
(b) an unforeseen one the CEO decides toward the cheapest-to-reverse default and flags via the
drift digest. Either way the founder's judgment is honored without the run ever stopping.

## Decide-and-flag, prefer reversible (replaces "stage and wait")

For each direction-changing fork that surfaces mid-run:
1. **Prefer the cheapest-to-reverse adequate option.** Build behind an abstraction/flag/config so
   the founder's later answer is a one-line change, not a rewrite (keep "Account", add an internal
   alias; wire price as a config value, don't hardcode or announce). This is decision *hygiene*,
   not a reason to wait.
2. **Decide and keep moving.** Pick the default, log it `DIRECTION`-tagged in `decisions.md`, and
   continue. Do not park the run for the founder.
3. **Leave the audit trail**: the proposal, the trade-offs, the default chosen, in one line. If it
   bends the charter/roadmap, the drift digest carries it to the founder; otherwise the next
   `review-and-plan` does.

## Frontloading (the CEO's pre-flight)

Front-loading is how the run stays uninterrupted. At the top of `run-the-company`, the CEO scans
the upcoming milestone for: credentials/keys, paid signups, App Store / deploy permissions, and
every foreseeable irreversible/strategic fork. Access needs go into ONE batched list up front;
direction-forks are pre-decided (or given a default + tolerance) during `review-and-plan`. A need
or fork discovered mid-run is a front-loading miss — but it does **not** stop the run: an access
gap parks only its own thread (`automation.md` §3); an unforeseen fork is decided-and-flagged
(above). Either way, note the miss for HR so the next pre-flight catches it.

## Logging

Every autonomous decision → append to `company_policies/decisions.md`:
```
## D<n> — <date> — <one-line decision>
Reversible: yes | Blast radius: internal | Direction-changing: no
Rationale: <why>. Alternatives considered: <…>.
```
Direction-changing calls are tagged `DIRECTION` (`Direction-changing: yes`) and carry the default
chosen + the tolerance it stayed within; if one bent the charter/roadmap, note `→ drift digest
<date>`. The founder's later answer at `review-and-plan` is appended (`founder confirmed |
reversed <date>`).
