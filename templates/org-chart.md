# <Company> Org Chart — the persona registry

> Single source of truth for the org. Recruiting reads/writes it; managers consult it to build persona chains. Keep it matching on-disk reality. **Every producer role has a reviewer** — review is a seat, at every layer.

| role | skill name | reviewer | reports to | owns | stack adapters |
|------|-----------|----------|-----------|------|----------------|
| ceo | `<company>-ceo` | `<company>-board-reviewer` → founder | founder | charter, roadmap, milestones | — |
| board-reviewer | `<company>-board-reviewer` | — | ceo | (reviews CEO plans) | — |
| product-manager | `<company>-product-manager` | `<company>-pm-reviewer` | ceo | use-cases/ ; **co-owns specs/** (demand side) | — |
| ux-manager | `<company>-ux-manager` | `<company>-ux-reviewer` | ceo | ux/ | — |
| architect | `<company>-architect` | `<company>-architect-reviewer` | ceo | architecture.md, data-model.md ; **co-owns specs/** (R#/A# + invariants) | — |
| tech-lead | `<company>-tech-lead` | `<company>-tech-lead-reviewer` | ceo | tasks/, **system-design.md** (deploy/next-phase, API, migration) | — |
| backend-engineer | `<company>-backend-engineer` | `<company>-backend-engineer-reviewer` | tech-lead | core code | <e.g. swift> |
| ios-engineer | `<company>-ios-engineer` | `<company>-ios-engineer-reviewer` | tech-lead | app code | <e.g. ios, swiftui> |
| qa-engineer | `<company>-qa-engineer` | `<company>-qa-engineer-reviewer` | tech-lead | acceptance + integration tests vs spec A#, cross-task code review | — |
| hr | `<company>-hr` | — | ceo (between waves) | performance/, persona SKILL edits | — |

Rules:
- Include only the roles this company needs (no empty seats) — but **Architect and Tech-Lead are distinct** (design vs delivery); don't collapse them on a non-trivial product.
- **specs/ is co-owned**: product-manager holds the demand side, architect holds the behavioural contract (R#/A# + invariants). **system-design.md is the tech-lead's** (topology/API/migration that gets operated), separate from the architect's `architecture.md` (invariants the system enforces).
- Every producing role MUST have a reviewer. CEO's reviewer is the internal `board-reviewer` (then founder sign-off). Reviewers and HR are the only roles without their own reviewer.
- **qa-engineer is product QA** (the second level of QA): does the software work for the user, at the milestone Integrate phase — distinct from the reviewer personas, who are agentic-integrity QA (did the agent behave). Keep both.
- Add stack adapters per executor so managers know which superpowers stack-skills to pull in.
