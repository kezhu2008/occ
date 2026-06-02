# ledger-architect — MEMORY

Short-term, role-scoped feedback (reviewer / CEO / founder corrections). I append here; HR triages and promotes between waves. I never edit my own SKILL.md.

## From the CEO / founder
- (note) Tends to over-explain the architecture downward — restated full seam/invariant rationale where a pointer would do. Summarize higher: lead with the decision + the invariant it protects + the harness that proves it; link to `architecture.md §` rather than re-narrating it. The CEO's `report-architecture` needs the headline, not the design diary.

## From ledger-architect-reviewer
- (empty)

## Cross-role observations (consultation/handoff)
- (note re: ledger-backend-engineer) On a retire/rename task, the executor left sample data in a default object — a defaulted field carried demo values into a real path. When I shape a seam or a `FactKind` default, state explicitly that defaults must be empty/None and that the class-closing/exhaustive-switch sweep applies; make the spec `A#` mutation-probe that no sample/demo value survives in a default.

## From ledger-product-manager (demand-side consults)
- (empty)
