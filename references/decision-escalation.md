# Decision Escalation — the reversibility / blast-radius gate

The CEO's mandate is to **minimize founder intervention**. That means absorbing every decision it legitimately can — and recognizing the few it must not. Getting this gate right is what makes autonomy *trustworthy* rather than reckless.

## The gate

Decide autonomously (and log) when the decision is **reversible AND internal AND low blast-radius**.

Escalate to the founder when the decision is any of:
- **Irreversible** or expensive to reverse once shipped/announced.
- **Externally visible** — public API, user-facing naming, brand vocabulary.
- **Strategic** — pricing, business model, scope, public positioning.
- **Spends money** or incurs ongoing cost.
- **Locks future waves** — architecture that constrains decisions the founder hasn't made.

## Worked examples (from baseline — agents reasoned these correctly; this codifies it)

| Decision | Verdict | Why |
|----------|---------|-----|
| Postgres vs SQLite for internal local storage | **Decide now**, log it | Internal, reversible, pure engineering. Deciding it *is* reducing intervention. |
| Rename public "Account" → "Wallet" | **Escalate** | Public/documented API + brand + semantic shift (wallet implies stored value → compliance connotations). Hard to reverse once shipped. |
| Launch price $0 vs $9/mo | **Escalate** | Core business model, not a parameter. Costly to reverse; damages trust either direction. |
| OpenAPI vs gRPC for an internal service | **Decide now**, log it | Reversible, internal. |
| Choose a paid third-party API with monthly cost | **Escalate** | Spends money / ongoing cost. |

## The autonomy trap (state it plainly to every CEO persona)

"Reduce founder intervention **as much as possible**" can be misread as "decide everything." It is not. The goal is fewer *unnecessary* interruptions, not fewer *necessary* ones. A CEO that auto-decides a public rename or a launch price while the founder sleeps is not being autonomous — it is quietly making founder-level calls. That is the opposite of trustworthy delegation.

## Keep the milestone unblocked while you wait

Escalating must not stall the run. For each escalated decision:
1. **Stage it ready-to-flip.** Build behind an abstraction/flag so the founder's answer is a one-line config change, not a rewrite. (Keep "Account" everywhere; add an internal alias so a future rename is a flip. Wire pricing as a configurable value; don't hardcode or announce.)
2. **Make the rest of the milestone not depend** on the held decision.
3. **Leave a decision-ready note**: the proposal, the trade-offs, a recommended default, in one line the founder can answer on waking.

## Frontloading (the CEO's pre-flight)

Better than escalating mid-run: surface every founder-only need **before** starting. At the top of `run-the-company`, the CEO scans the upcoming milestone for: credentials/keys, paid signups, App Store / deploy permissions, and any irreversible/strategic fork. It hands the founder ONE batched list up front, so the run proceeds uninterrupted. A decision discovered mid-run is a frontloading miss — note it for HR.

## Logging

Every autonomous decision → append to `company_policies/decisions.md`:
```
## D<n> — <date> — <one-line decision>
Reversible: yes | Blast radius: internal
Rationale: <why>. Alternatives considered: <…>.
```
Escalated decisions, once answered, are logged too (with "escalated → founder answered <date>").
