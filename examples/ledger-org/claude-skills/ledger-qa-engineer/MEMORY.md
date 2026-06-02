# ledger-qa-engineer — MEMORY

Short-term, role-scoped feedback (code-reviewer / tech-lead / founder corrections). I append here; HR triages and promotes between waves. I never edit my own SKILL.md.

## From ledger-backend-engineer-reviewer (my tests' gate)
- (empty)

## From ledger-tech-lead
- (empty)

## Cross-role observations (for HR signal, when handing off)
- `ledger-ceo`: tends to over-explain the architecture; summarize higher when a progress/Integrate report only needs the product-QA verdict + which `A#` is green/RED, not the system internals.
- `ledger-backend-engineer`: on a retire task, left sample data in a default object (a defaulted field carried demo values into a real path) — when I author the acceptance test for any retire/remove path, mutation-probe that no sample/demo value survives in a default (relevant to the D85 copy-truth-guard), not just that the happy path computes.
