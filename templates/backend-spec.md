# <Subsystem> Backend Spec

> **Co-owned by Product Manager + Architect.** PM frames the required behaviour from the use-cases; the Architect makes it a precise, invariant-bearing contract. `pm-reviewer` **and** `architect-reviewer` both gate this file. Lives in `company_policies/specs/<subsystem>.md`.
>
> This is the **behavioural contract a developer implements against and a test asserts against** — no guessing. Organize **by subsystem, not by use-case** (one `R#`/`A#` backs many use-cases).
>
> **Two parts, both load-bearing:**
> - **`R#` requirements** — what the system MUST do/hold. Behaviour and invariants, **architecture-agnostic** (state the *what*, not the *how*).
> - **`A#` acceptance** — **executable** facts. A test asserts each *without a human judging*.
>
> **Where things live (do not mix):** business vision → `charter.md`; user flow → `use-cases/`; **technical invariants** (Decimal-only, Foundation-only, determinism, ordering) → **here**.

## Subsystem
<One line: what this subsystem is responsible for, and its boundary — what it owns vs. what it calls out to.>

## Backed use-cases
<The UC ids this spec satisfies, so the trace is bidirectional. Every UC here must reference these R#/A# back; every A# below must appear in at least one UC or be a named invariant. A UC with no backing spec — or a spec item no UC/invariant uses — is a gap the reviewers FAIL.>
- `UC-<…>`, `UC-<…>`

---

## Requirements (R#)

> Each `R#` is a behaviour or invariant the system MUST hold, stated **architecture-agnostic** (no class names, no file formats unless the format *is* a non-negotiable invariant). One requirement, one line. Number monotonically; never renumber — append.

R1. <A behaviour the system must exhibit. e.g. "Posting a transaction is append-only; an existing entry is never mutated in place — corrections are new compensating entries.">
R2. <An invariant that must always hold. e.g. "All monetary values are exact decimal; no binary floating-point is used in any money path, intermediate or stored.">
R3. <An ordering / determinism rule. e.g. "Replaying the event log from empty yields the same derived state regardless of machine, locale, or wall-clock time.">
R4. <A boundary / failure rule. e.g. "An import that fails validation commits nothing — the ledger is unchanged (all-or-nothing).">
R5. <…>

> Guidance: if you cannot write at least one `A#` that *proves* an `R#`, the requirement is too vague — sharpen it until it is testable.

---

## Acceptance (A#) — offline-testable; this is what tests assert

> Each `A#` is a **provable fact**: a test can assert it with no human judgement. Give it the fixture/precondition, the exact operation, and the **exact expected result** (value, equality kind, error). Prefer exact equality over "looks right". State the tolerance explicitly (usually **zero**). Each `A#` names the `R#`(s) it proves and the `UC`(s) it serves.

A1. **<short name>** — *Given* <fixture/precondition>, *when* <operation>, *then* <exact, machine-checkable result>.
&nbsp;&nbsp;&nbsp;&nbsp;e.g. "Given a ledger reconciled to a statement, the reconciled cash balance equals the statement closing balance by Decimal **exact** equality; a 1¢ delta FAILs." — *(proves R2; serves UC-RECONCILE)*

A2. **Replay determinism** — Drop all derived projections and replay the log from empty ⇒ resulting state is **byte-identical** to the pre-drop snapshot. — *(proves R3; serves UC-…)*

A3. **All-or-nothing import** — Given an import file with one invalid row, the import is rejected and `entryCount` after == `entryCount` before. No partial entries exist. — *(proves R4; serves UC-…)*

A4. **No float in the money path** — A property/round-trip test over N random amounts: serialize → deserialize → re-serialize is identical, and no value equals its float-rounded form when they should differ at the cent. — *(proves R2)*

A5. **Append-only** — After posting then "correcting", the original entry is still present and unchanged; the correction is a distinct new entry referencing it. — *(proves R1; serves UC-…)*

A6. <…each remaining invariant/behaviour gets at least one A#…>

> **Tracing checklist (reviewers enforce):**
> - [ ] Every `R#` is proven by ≥1 `A#`.
> - [ ] Every `A#` names its `R#`(s) and ≥1 `UC` (or is an explicit named invariant).
> - [ ] Every `A#` is assertable with **no human judging** — exact value/equality/error, stated tolerance.
> - [ ] No business vision or user-flow prose leaked in (those live in `charter.md` / `use-cases/`).

---

## Out of scope / non-goals
<What this spec deliberately does NOT cover, to keep the contract tight and stop scope creep into tests.>

## Open questions
<Anything PM and Architect have not yet locked. An open question blocks the `A#` it touches — flag it; do not let a developer guess.>

## Source
<file:line once implemented; "(to be implemented)" while proposed. The developer's tests and QA's acceptance tests both point back to the `A#` ids above.>
