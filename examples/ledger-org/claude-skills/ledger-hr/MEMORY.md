# ledger-hr — MEMORY

Short-term, role-scoped feedback HR receives about its OWN behavior (CEO/persona corrections). HR triages everyone else's MEMORY between waves; this file is only HR's own. HR does not self-promote — these accumulate until OCC/the CEO acts on them.

## Per-persona triage notes (carried between waves)

- **ledger-ceo** — over-explained the architecture in a `report-architecture` roll-up (restated `architecture.md §6` / Div775 internals instead of pointing to them). Signal: summarize HIGHER — the CEO is the single founder-facing layer; it should reference the invariant, not re-derive it. Watch for a recurring pattern → candidate behavior promotion to `ledger-ceo/SKILL.md` (RED baseline: a roll-up that restates a spec must be flagged; the rewritten skill points to the artifact). Not yet recurring enough to promote.
- **ledger-backend-engineer** — left sample/seed data in a default object on a retire task (the class-closing/exhaustive sweep on a remove/rename should have caught it). Signal: a retire task must run the class-closing sweep AND scrub default/sample data. Watch for recurrence → candidate behavior promotion to `ledger-backend-engineer/SKILL.md` (RED baseline: a retire task that leaves seed data in a default must turn the probe RED) or a `knowledge/` gotcha if it's a one-off trap rather than a behavioral miss.

(All other personas: empty.)
