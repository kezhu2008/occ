# Consultation — agentic feedback between roles (not agile ceremony)

Roles must be able to tell each other "this isn't right / isn't clear" and have it resolved — **without importing human agile practice.** No standups, sprints, tickets, story points, or retros. Consultation is a **structured, ID-referenced artifact** that flows along the org edges and is resolved by the artifact's owner. Optimize for an agent consuming it, not a human in a meeting.

## When to consult (don't guess, don't silently absorb)

- A developer hits an **ambiguous or contradictory spec/task** → consult the tech-lead (owner of the task) rather than picking an interpretation and building the wrong thing.
- The **tech-lead finds the architecture forces a bad decomposition** (an invariant blocks a clean split, a seam is missing) → consult the architect with the **impact on the spec named**.
- The **architect finds a use-case under-specified or infeasible** → consult the PM.
- Any role spotting a **defect in an upstream authoritative artifact** → consult its owner. Code is truth; if code contradicts a doc, that's a consultation, not a silent override.

This is distinct from **escalation** (which goes to the founder for direction-changing calls) and from **review** (a gate that returns PASS/FAIL). Consultation is *lateral/upward clarification within the org*, and it is **bidirectional** — feedback flows up the design chain, not just work down it.

## The consultation artifact

```json
{
  "type": "consultation",
  "from": "ledger-tech-lead",
  "to": "ledger-architect",              // the OWNER of the artifact in question
  "about": "architecture.md §7.2 FactStore seam",
  "question_or_impact": "Decomposing W6-M1 needs a FactStore protocol that does not exist; the append path is welded to the on-disk log. This blocks an independently-shippable server-lift task.",
  "spec_impact": ["R5", "A6", "data-model.md §10"],   // the spec/artifact IDs affected
  "proposed_change": "Introduce `protocol FactStore` before the lift; bind on-disk + Dynamo impls behind it.",
  "blocking": true
}
```

Resolution (the owner returns):
```json
{
  "type": "consultation-resolution",
  "resolves": "<the consultation>",
  "decision": "Accepted — adding FactStore protocol to architecture.md §7.2; data-model.md §10 updated.",
  "artifact_changes": ["architecture.md", "data-model.md"],
  "status": "RESOLVED | DEFERRED | ESCALATED"
}
```

## Rules (keep it agentic and cheap)

- **Reference IDs, not prose history.** Name the exact spec/use-case/architecture IDs affected so the owner can act without reconstructing context. No narrative meeting notes.
- **The owner resolves by changing the artifact**, then the change flows through that artifact's normal review gate (a revised spec is re-reviewed). Consultation feeds the review loop; it doesn't bypass it.
- **Blocking vs non-blocking.** A `blocking:true` consultation halts the dependent work until RESOLVED (or the manager escalates). Non-blocking is logged and batched.
- **Cap the round-trips.** If a consultation doesn't converge in a couple of exchanges, the manager escalates it (BLOCKED) rather than looping — same convergence discipline as review.
- **Log it.** Resolved consultations that changed an artifact are noted in `decisions.md` (if a decision) or the artifact's history. Recurring consultations on the same ambiguity are a signal for HR (the spec/skill needs a durable fix).
- **No human ceremony.** Do not schedule, do not "sync", do not assign sprint points. The artifact IS the communication.
