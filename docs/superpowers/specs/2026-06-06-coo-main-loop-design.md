# COO — the main-loop operating discipline

**Date:** 2026-06-06
**Status:** Design approved; ready for implementation plan.
**Scope:** OCC framework change. One spec, full fold-in.

## Problem

In a normal conversation turn — **no slash command, no skill invocation** — nothing
governs the main loop. It does specialist work inline: reads big files, writes code, edits
specs. This **bypasses the OCC org** (the whole point of the layered delegation) and
**pollutes the main context window** with detail that should have lived and died inside a
subagent.

The CEO does not fix this: the CEO is a *skill* you invoke. Skills load into subagents, and
the main loop cannot be a subagent of itself. So the default front — the main loop — is
ungoverned.

## Solution (proven in the live ledger instance, D135/D136)

Promote the **main loop to a COO**: a standing operating discipline that **facilitates the
conversation but offloads actual work to specialists**.

- The founder talks to the **COO (the main loop)** by default, no command.
- The COO **triages** every turn, **delegates down** to specialist subagents (relaying only
  conclusions), and **escalates up** to the CEO for direction-changing calls.
- The COO is **the discipline running on the main loop, not a subagent** — it is the only
  level that can fan out (subagents cannot spawn subagents). Isolation comes from
  *delegating*, not from being a subagent: a specialist's detail stays in its subagent and
  dies there; only its conclusion returns. The main window stays thin.

The COO therefore lives in the repo's **`CLAUDE.md`**, not in `.claude/skills/`. OCC must now
produce that file.

## Decisions (locked in brainstorming)

1. **COO owns execution; CEO = pure planner/governor.** Mirror ledger D136. `run-the-company`
   is removed from the CEO; the autonomy doctrine moves CEO → COO.
2. **COO = a formal org seat**: a `job-descriptions/coo.md` + an `org-chart.md` line, but its
   *instantiation* is the repo `CLAUDE.md` (emitted by recruiting), **not** a
   `.claude/skills/<company>-coo/`. It has **no skill** and **no reviewer**.
3. **Default-delegate triage.** The COO answers inline ONLY for: a fact already in its window,
   a one-line conversational reply, or pure routing/status. Anything touching repo files,
   code, specs, data, or a domain question → delegate to the owning specialist, even if it
   "looks like one line." "Looks simple" is an explicit red flag, not a license.
4. **Full fold-in, one spec.**
5. **(a)** Recruiting writes the COO doctrine inside a **managed delimited block** in
   `CLAUDE.md` so founder-authored content coexists; re-recruiting rewrites only the block.
6. **(b)** The COO is a **standing/default seat in every OCC org** — strategic-review always
   includes it.

## Design

### 1. Org-model change

Replace today's "the founder talks to one persona — the CEO" with:

- **Founder ↔ COO (main loop), by default, no command.**
- **COO triage** (default-delegate):
  - *Trivial* (in-window fact / one-line reply / routing or status) → answer inline.
  - *Domain question or work* (touches files, code, specs, data, a domain) → spawn the owning
    specialist as an **isolated subagent** (the agent loads its `<company>-<role>` skill).
    **Relay only its conclusion.** Never pull the specialist's detail — file dumps, skill
    bodies, code, API responses — into the main window.
  - *Direction / scope / "is the wave done" / new seat / pricing / public naming / spend* →
    **escalate UP to the CEO** (with the founder).
- **CEO = the governance layer escalated up to** (`review-and-plan`, `report-architecture`,
  `report-progress`). **Specialists = delegated down to** (subagents).
- The COO is the only level that can fan out; it is the discipline on the main loop, not a
  subagent.

`references/corporate-structure.md` diagram gains a COO box:
`FOUNDER ↔ COO (default)`, COO escalates ↑ to CEO, delegates ↓ to managers/executors. The
CEO is still reviewed by the board-reviewer + founder sign-off for plans.

### 2. COO seat artifacts

- **`company_policies/job-descriptions/coo.md`** (new). Names all four flows, like every JD:
  - *Reports to:* founder (it is the front).
  - *Reviewed by:* **none** — see rationale below.
  - *Consults:* CEO (up, for direction), specialists (down, for domain detail).
  - *May decide (judged by outcome):* triage/routing; autonomous execution of an already
    board-PASSed + founder-signed plan; reversible/internal/low-blast-radius calls, logged
    with their harness to `decisions.md`.
  - *Must escalate (to the CEO + founder):* any direction-changing fork — a charter-named
    surface at risk, a scope cut, a new seat, pricing, public naming, a spend ceiling.
  - *Mission:* facilitate the founder conversation while keeping the main window thin by
    delegating; own the autonomous execution of signed plans; never do specialist work inline.
  - *Must never:* do specialist work inline to "save a hop"; pull specialist detail into the
    main window; mark anything DONE that skipped its independent reviewer/QA gate; hard-stop a
    long run to ask the founder (decide-and-flag / park-the-thread / drift-digest instead);
    bypass a permission denial.
- **`org-chart.md`** — add the COO row: `role = COO`, `skill = — (main loop / CLAUDE.md)`,
  `reviewer = — (none)`, `reports-to = founder`. The two `—` cells are annotated as
  deliberate exceptions, not recruiting misses.
- **No-reviewer / no-skill rationale** (documented in `corporate-structure.md`): the COO
  **produces no artifact** — it relays conclusions and logs decisions. Its work is gated by
  the **downstream reviewer + QA gates it dispatches** and harnessed by `decisions.md`
  outcomes. It has no skill because the main loop cannot load itself as a subagent; the
  `CLAUDE.md` is its instantiation.

### 3. CLAUDE.md template + recruiting (the new capability)

- **New `templates/claude-md.md`** — the generative source for `<repo>/CLAUDE.md`. Sections
  (generalized from the proven ledger COO doctrine):
  - operating-mode header (`you are the COO`, the default front);
  - **on every founder turn** — the default-delegate triage (Decision 3);
  - the **routing table** (domain → specialist skill) — a fill-in, generated per company;
  - the rules that hold — gates hold (no specialist self-certifies DONE; live/data claims
    proven on real data); secrets never in the transcript; stay thin (compact state in-window,
    durable detail to files); don't bypass a permission denial;
  - escalating up vs deciding down;
  - **running the company (execution — COO-owned)** — the long-running autonomy doctrine;
  - the deferred two-tier reasoner-isolation note.
- **`skills/occ-recruiting/SKILL.md` gains a step** — after writing the persona skills,
  **emit `<repo>/CLAUDE.md`**:
  - fill the routing table from `org-chart.md` (each manager/executor → its trigger domains,
    pulled from each JD's domain), inject the company name and the `<company>-*` skill names;
  - handle the COO seat specially: JD ✅, skill ❌, reviewer ❌, `CLAUDE.md` ✅;
  - **managed-block write (Decision a):** wrap the COO doctrine in
    `<!-- OCC:COO begin -->` … `<!-- OCC:COO end -->`. If `<repo>/CLAUDE.md` exists, replace
    only that block and leave founder-authored content intact; if absent, create the file with
    the block. Re-recruiting a changed org rewrites only the block (and its routing table).
- **`skills/occ-strategic-review/SKILL.md`**: the COO is a **standing seat** in every org
  design (Decision b) — it always appears in `org-chart.md`, with its no-skill/no-reviewer
  exceptions noted.

### 4. Reference / SKILL doc spread

- **`SKILL.md`**:
  - rewrite "the founder talks to **one** persona — the CEO" → the founder talks to the **COO
    (main loop)** by default; the CEO is escalated up to;
  - rewrite "Run my company / ship the next milestone → invoke the CEO" → **planning** is the
    CEO (`review-and-plan`); **running** is the COO main loop, triggered by the founder saying
    "run the company" in normal conversation;
  - add top-level `CLAUDE.md` to the canonical repo layout (the COO instantiation);
  - add the COO to the recruited-org list (with its exceptions);
  - update the workflow diagram: recruiting now also emits `CLAUDE.md`.
- **`references/corporate-structure.md`**: add the COO layer, the main-loop subtlety, and the
  no-reviewer/no-skill exception.
- **`references/automation.md`**: the long-running execution doctrine (decide-and-flag,
  park-the-thread, drift-digest) **attaches to the COO**; the CEO keeps front-loading every
  foreseeable fork in planning.
- **`references/decision-escalation.md`**: a mid-run direction-fork **escalates ↑ from the COO
  to the CEO** (with the founder); reconcile with the in-flight long-running-CEO diff.

### 5. CEO re-split

- CEO command surface drops to **3**: `review-and-plan`, `report-architecture`,
  `report-progress`. **`run-the-company` is removed from the CEO.**
- "Run the company" becomes the **COO's** no-command trigger → autonomous execution of the
  **board-PASSed + founder-signed** plan.
- Autonomy doctrine **moves CEO → COO**. The CEO keeps: front-load every foreseeable fork in
  `review-and-plan`; produce the signed plan; **assemble** the access pre-flight list. The COO
  **verifies access and parks gaps** at run start. This rewinds part of the uncommitted
  long-running-CEO diff onto the COO.
- Guardrail unchanged: the COO executes **only** an already board-PASSed + founder-signed plan
  — it never authors or self-approves the plan. The plan/scope board-gate stays the CEO's.

### 6. Worked example (`examples/ledger-org`)

- Add **`examples/ledger-org/CLAUDE.md`** — the proven `~/git/ledger/CLAUDE.md` COO doctrine,
  wrapped in the managed block.
- Add **`examples/ledger-org/company_policies/job-descriptions/coo.md`**.
- Update `org-chart.md` (COO row), `ceo.md` (3 commands, run-the-company removed,
  autonomy-doctrine references moved), and the `claude-skills/ledger-ceo/SKILL.md` already in
  the working diff — keep them consistent with sections 4–5.

## Affected files (implementation checklist)

New:
- `templates/claude-md.md`
- `company_policies/job-descriptions/coo.md` (template-side, via `templates/job-description.md` instance) — and the ledger-org copy
- `examples/ledger-org/CLAUDE.md`
- `examples/ledger-org/company_policies/job-descriptions/coo.md`

Edited:
- `SKILL.md`
- `references/corporate-structure.md`
- `references/automation.md`
- `references/decision-escalation.md`
- `templates/org-chart.md`
- `skills/occ-recruiting/SKILL.md`
- `skills/occ-strategic-review/SKILL.md`
- `examples/ledger-org/company_policies/org-chart.md`
- `examples/ledger-org/company_policies/job-descriptions/ceo.md`
- `examples/ledger-org/claude-skills/ledger-ceo/SKILL.md`

## Out of scope

- The deferred **two-tier reasoner isolation** (main loop as a near-empty conduit spawning a
  COO-reasoner subagent for the routing decision). Kept as a documented "deferred" note in the
  template; not built here.
- Re-recruiting tooling beyond the managed-block rewrite.

## Success criteria

- A freshly recruited company has a top-level `CLAUDE.md` whose COO doctrine routes a
  no-command founder turn to triage → delegate/escalate, with a routing table matching its
  `org-chart.md`.
- The CEO no longer carries `run-the-company`; the COO carries execution; the autonomy doctrine
  appears once, on the COO.
- The org docs (SKILL.md, corporate-structure.md, automation.md, decision-escalation.md) read
  coherently with the COO as the default front and the CEO as the escalated-to governor — no
  stale "founder talks to the CEO" lines remain.
- `examples/ledger-org` carries a `CLAUDE.md` + `coo.md` consistent with the framework and with
  the proven live `~/git/ledger/CLAUDE.md`.
