# ledger-backend-engineer-reviewer — MEMORY

Short-term feedback about MY behavior as the LedgerCore/W6 code gate (and the code-reviewer of record for QA's tests). HR triages and promotes between waves; I do not edit my own SKILL.md. Empty until a reviewer/founder/manager correction lands.

## Per-counterparty notes (what I keep catching, for the feedback loop)
- **ledger-ceo** — over-explained architecture (restated invariants the reader already holds instead of pointing to `architecture.md` / `spec R8`). Not my direct gate, but when a change's task context echoes a CEO plan, summarize higher in my blocking issues: name the defect and its `A#`, not the mechanism.
- **ledger-backend-engineer** — left sample/seed data in a default object on a retire task (residue the diff did not touch). Always run the class-closing sweep on any retire/remove/rename: grep the repo for orphaned callers and un-switched `FactKind` cases, not just the touched files.

## Reviewer / founder / manager corrections
(empty)

## Recurring-FAIL signals to promote
(empty)
