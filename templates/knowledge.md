# Knowledge templates — index + per-domain entry

Knowledge is the **shared** MAP + GOTCHAS + LESSONS that make the org repo-familiar. It lives in `company_policies/knowledge/`, is read by everyone, and is **gardened only by HR**. See `references/memory-vs-persistence.md` for the tier model and the HR promotion loop.

**The one rule that shapes every entry: REFERENCE, NEVER RESTATE.** An entry holds the *map* (where a thing lives), the *gotcha* (the trap that bites you), and the *lesson* (what to do instead) — then **points to the authoritative artifact** (`spec R#/A#`, `architecture.md §x`, `File.swift:line`). If the content is a fact, it belongs in a spec, not here. Knowledge that duplicates a spec is deleted on the next GC.

Load discipline: a persona loads only the `<domain>.md` files it touches, found via the index — knowledge never loads wholesale into context.

---

## `knowledge/index.md` (one line per domain)
```md
# Knowledge Index
One line per domain. A persona scans this, then loads only the domain(s) it touches.

| domain | file | covers (5–8 words) | entries | last GC |
|--------|------|--------------------|---------|---------|
| tax-engine | tax-engine.md | FX, Div775, CGT routing traps | 4 | 2026-05-29 |
| persistence | persistence.md | SwiftData migrations, retire/remove sweeps | 3 | 2026-05-29 |
| <domain> | <domain>.md | <what lives here> | <n> | <date> |
```
Guidance: domain = a coherent slice of the system (an engine, a layer, a subsystem), not a single file. Keep `covers` terse — it is a routing hint, not a summary. `entries` and `last GC` let HR spot stale or bloated domains at a glance.

---

## `knowledge/<domain>.md` (per-domain entries)
```md
# Knowledge — <domain>
Budget: ≤ ~150 lines / ≤ ~12 entries. HR-gardened: on add, dedup + merge + supersede.
Authoritative sources for this domain: <architecture.md §x>, <specs R#/A#>.

## <short title of the map/gotcha>
- Map: <where the real logic lives → the artifact reference>
- Gotcha: <the trap — the thing that silently breaks if you don't know it>
- Lesson: <what to do instead>
- Ref: <spec R#/A# | architecture.md §x | File.swift:line>   ← the authority; the lines above must NOT restate it
- Added: <date> by HR (from <persona> MEMORY <task-id>)

## <next entry…>
- Map: …
- Gotcha: …
- Lesson: …
- Ref: …
- Added: …
```
Guidance:
- Each entry = **map + gotcha + lesson + ref**, in 4–6 lines. If it runs longer, you are restating a spec — cut it and lean on `Ref`.
- `Ref` is mandatory. An entry with no authoritative reference is a floating fact; HR either finds its artifact and links it, or the fact moves into a spec and the entry is deleted.
- Depth vs breadth share this store: executor entries are deep/narrow (one domain's traps); lead/architect entries are broad (cross-domain map of where things live). Same file, different readers.
- **Anti-bloat is enforced, not aspirational.** Every wave HR runs a GC: dedup overlapping entries, merge near-duplicates, supersede entries whose referenced artifact moved/changed, and prune anything now restated by a spec. An entry pointing at a dead `File.swift:line` is updated or deleted. Growth without pruning is an HR failure mode.

---

## Good vs bad entry

✅ **Good** — map + gotcha + lesson, points at the authority, restates nothing:
```md
## FX gains route through Div775, not CGT
- Map: foreign-currency movements are taxed by Div775Engine, never the CGT engine.
- Gotcha: a USD movement that skips Div775 silently under-reports tax — no error, wrong number.
- Lesson: when touching any currency-crossing path, confirm it enters Div775Engine first.
- Ref: architecture.md §6 / Div775Engine.swift:40 / spec R12
- Added: 2026-05-29 by HR (from tax-backend-engineer MEMORY M4-T2)
```

❌ **Bad** — restates the spec, no reference, will be deleted on GC:
```md
## How FX tax works
The Div775 algorithm computes the gain as (settlement_rate − acquisition_rate) × notional,
then applies the marginal rate brackets: 0% to $18,200, 19% to $45,000, 32.5% to …
[a full re-derivation of the algorithm that already lives in the spec]
```
Why it's bad: it duplicates `spec R12` (so it rots the moment the spec changes), carries no `Ref`, and blows the line budget. The fix is the Good version above — three lines of navigation + one pointer.
