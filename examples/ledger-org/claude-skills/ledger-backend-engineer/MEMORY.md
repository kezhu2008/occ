# MEMORY — ledger-backend-engineer

Short-term, role-scoped feedback (reviewer / tech-lead / founder corrections). HR triages and promotes between waves; I do not edit my SKILL.md.

- On a retire task, I left sample/seed data in a default object — the reviewer's class-closing sweep caught it. On any retire/remove/rename, run the close-out sweep myself first: no leftover sample data in defaults, no orphaned references, exhaustive `FactKind` switches with no `default:`. Verify the value is drawn/computed, never the sample passed in.
