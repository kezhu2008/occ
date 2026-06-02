# OCC — One Consultancy Company

A **meta-skill** that staffs an AI "company" to build and maintain a specific code repository. OCC is the consultancy that designs and hires the org; it does not build your product — the org it recruits does.

> **A company = one code repository pursuing one business intent.**

## The idea

Reuse modern corporate structure — the proven answer to *"how does one owner get a large amount of trustworthy work done without being in every decision?"* OCC instantiates that as a layered team of persona **skills** scoped to one repo:

```
Founder ─► CEO ─► managers (product / ux / tech-lead) ─► executors (+ reviewers)
                                                  └─ HR (trains personas over time)
```

OCC sells two services, like a real consultancy:
1. **Strategic review** (`skills/occ-strategic-review/`) — interview the founder, design/revise the org, write versioned job descriptions.
2. **Recruiting** (`skills/occ-recruiting/`) — instantiate the org as invokable persona skills, each reusing superpowers and paired with an independent reviewer.

## What's in here

| Path | What |
|------|------|
| `SKILL.md` | OCC meta-skill entry — the model, the two capabilities, the canonical layout. |
| `skills/occ-strategic-review/` | Capability 1 — org design. |
| `skills/occ-recruiting/` | Capability 2 — writes persona skills. |
| `references/` | The model: corporate structure, handoff & the review gate, memory-vs-persistence, decision escalation, dynamic workflows, artifact layout. |
| `templates/` | Fill-in scaffolds: charter, org-chart, job description, persona, reviewer, handbook, specs, memory/performance. |
| `examples/ledger-org/` | A full worked org for `~/git/ledger`, generated without touching the real repo. |

## Two things OCC v2 formalizes over v1 (`running-the-company` + global `persona-*`)

1. **Personas are scoped to the repo** (`<repo>/.claude/skills/<company>-<role>/`), not global — and indexed by a queryable **org-chart registry** instead of informal chain strings.
2. **The review gate and the memory→skill loop are structural**: every executor has a named independent reviewer; only HR promotes persona MEMORY into SKILL.md, via TDD (RED baseline first).

## Using OCC

1. Install OCC's skills where your agent discovers them (`SKILL.md` + `skills/*` → `~/.claude/skills/`).
2. In your product repo: invoke `occ` → run `occ-strategic-review` (design) → `occ-recruiting` (hire).
3. Then talk to your CEO: `<company>-ceo` → `run-the-company`.
4. Between waves: `<company>-hr` trains the org; re-run `occ-strategic-review` only for structural change.

## Design provenance

Built TDD-style per superpowers:writing-skills: RED baselines first probed the three discipline-bearing behaviors (independent-review gate, memory-vs-persistence filing, decision escalation). The review gate was the one behavior fresh agents skipped under deadline pressure — so the handbook and reviewer personas carry explicit anti-rationalization for it. The other two were codified from the throughline agents already reasoned correctly.
