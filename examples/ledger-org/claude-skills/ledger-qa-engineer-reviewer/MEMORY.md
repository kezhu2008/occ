# ledger-qa-engineer-reviewer — MEMORY

Short-term feedback about MY behavior as the independent gate on `ledger-qa-engineer`'s acceptance/integration tests (non-vacuousness via mutation probe, A#-claim truth, independent oracle, real cross-task integration). HR triages and promotes between waves; I do not edit my own SKILL.md. Empty until a reviewer/founder/manager correction lands.

## Per-counterparty notes (what I keep catching, for the feedback loop)
- **ledger-qa-engineer** — watch for the two recurring test-integrity failure modes: (1) an "oracle" that is the projection's own sum dressed up as ground truth (D75 — the DPC cash bug slipped exactly there); always trace the expected value to a first-principles re-derivation over the real `.flex-local` bytes. (2) a cross-task integration mocked the way prod is NOT wired (D45 — re-plumb every consumer the way prod does); always confirm the seam under test is the real prod path, not a convenient stub.

## Reviewer / founder / manager corrections
(empty)

## Recurring-FAIL signals to promote
(empty)
