# Consultation — template

A consultation is a **structured, ID-referenced artifact** that flows along an org edge and is resolved by the artifact's owner. It is how one role tells another "this isn't right / isn't clear" without importing human agile ceremony — no standups, sprints, tickets, points, or retros. Fill in the two JSON blocks below. The artifact IS the communication; optimize it for the agent that will consume it.

> Use this when you hit an ambiguous/contradictory spec, an architecture that forces a bad decomposition, an under-specified or infeasible use-case, or a defect in an upstream authoritative artifact. This is NOT escalation (founder-bound, direction-changing) and NOT review (a PASS/FAIL gate). Consultation is lateral/upward clarification within the org, and it is bidirectional — feedback flows up the design chain.

---

## 1. The consultation (raised by the consumer of the artifact)

```json
{
  "type": "consultation",
  "from": "<your role / skill name, e.g. ledger-tech-lead>",
  "to": "<the OWNER of the artifact in question, e.g. ledger-architect>",
  "about": "<exact artifact + section, e.g. architecture.md §7.2 FactStore seam>",
  "question_or_impact": "<state the concrete blocker, not history: what you tried to do, what blocks it, why building anyway would be wrong>",
  "spec_impact": ["<ID>", "<ID>"],
  "proposed_change": "<your suggested fix to the owner's artifact, concrete enough to act on>",
  "blocking": true
}
```

Field guidance:
- **`from` / `to`** — `to` is always the OWNER of `about` (consult org-chart.md to find who owns the artifact). Consultation flows along an org edge to the one role that can change the thing.
- **`about`** — name the exact artifact and section. One consultation, one artifact.
- **`question_or_impact`** — the impact, stated tightly. Don't narrate a meeting; say what is blocked and why a silent interpretation would build the wrong thing.
- **`spec_impact`** — **reference IDs, not prose** (R#, A#, use-case IDs, `data-model.md §10`). These let the owner act without reconstructing context. If nothing downstream is affected, use `[]`.
- **`proposed_change`** — propose, don't demand. The owner may accept, alter, defer, or escalate.
- **`blocking`** — `true` halts the dependent work until RESOLVED; `false` is logged and batched. Be honest: not everything is blocking.

---

## 2. The resolution (returned by the owner of the artifact)

```json
{
  "type": "consultation-resolution",
  "resolves": "<id or restatement of the consultation above>",
  "decision": "<what was decided, in terms of the artifact: Accepted/Altered/Declined + why>",
  "artifact_changes": ["<artifact>", "<artifact>"],
  "status": "RESOLVED"
}
```

Field guidance:
- **The owner resolves by changing the artifact** — list every artifact touched in `artifact_changes`. The revised artifact then flows through its **normal review gate** (a revised spec is re-reviewed). Consultation feeds the review loop; it does not bypass it.
- **`status`** is one of:
  - `RESOLVED` — artifact changed (or change consciously declined with reason); dependent work may proceed.
  - `DEFERRED` — non-blocking; logged and batched for later.
  - `ESCALATED` — couldn't converge or it's a direction-changing call; handed to the founder via escalation.

---

## Rules (keep it agentic and cheap)

- **Reference IDs, not prose history.** Name exact spec/use-case/architecture IDs so the owner acts without rebuilding context.
- **Cap the round-trips.** If it doesn't converge in a couple of exchanges, the manager escalates it (`ESCALATED`) rather than looping — same convergence discipline as review.
- **Log it.** A resolution that changed an artifact is noted in `decisions.md` (if a decision) or the artifact's history. **Recurring** consultations on the same ambiguity are a signal for HR — the spec or persona skill needs a durable fix.
- **No human ceremony.** Do not schedule, sync, or assign points. Fill the JSON, send it along the edge, resolve it by editing the artifact.
