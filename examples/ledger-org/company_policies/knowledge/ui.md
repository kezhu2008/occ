# Knowledge — ui
Budget: ≤ ~150 lines / ≤ ~12 entries. HR-gardened: on add, dedup + merge + supersede.
Authoritative sources: `specs/domain-core.md` R23·R24 / A16·A18; `decisions.md` D64/D82/D85; `ux/positions.md`; `Sources/Ledger/` (app target).

## RenderedMoney is the drawn==asserted chokepoint — assert the bytes that DRAW, not the VM input
- Map: money reaches the screen through `RenderedMoney`; a render test must assert the string actually drawn (`TaxSummaryView.swift:272`), not the value handed to the view model.
- Gotcha: asserting the VM input is vacuous — a formatting/clamp bug between the VM and the pixels passes silently because the test never looks at the drawn bytes.
- Lesson: pin the rendered string at the `RenderedMoney` chokepoint; let the VM-input assertion be a control, never the headline.
- Ref: `specs/domain-core.md` A6 (canonical money form, drawn must match); `ux/positions.md`
- Added: 2026-05-31 by HR

## Demo-retired copy-truth — the shipping app NEVER draws demo holdings (WaveCodeCopyLockTests, D85)
- Map: `flex_dummy.xml` is a TEST-ONLY fixture; `WaveCodeCopyLockTests` asserts no shipping surface draws demo/sample copy on every surface.
- Gotcha: a feature/retire edit can leave a demo branch latent in a render path (FM-2, 2026-05-29) — the feature test renders real data and never notices the dead demo wiring.
- Lesson: on any retire/rename/feature edit, grep the whole core + app for demo/sample/placeholder residue (not just the file you touched) and rely on the `WaveCodeCopyLockTests` copy-truth lock to fail on a demo string.
- Ref: `decisions.md` D85 (shipping shows no sample data); `Tests/.../WaveCodeCopyLockTests.swift`
- Added: 2026-05-31 by HR

## Signed cash is never clamped — OWED renders as dim text, not a badge
- Map: per-(entity, account, currency) cash is a signed derived fold; a negative/margin balance renders negative (dim "OWED" text), and the position market value renders the projection's real number, not a hardcoded 0.
- Gotcha: clamping a balance to 0, or hardcoding a `0` market value (`TaxViewModel.swift:500`, D54), hides a real negative FX position or a real loss-harvest candidate value.
- Lesson: render the signed projection figure verbatim; treat sign as a display style, never a clamp.
- Ref: `specs/domain-core.md` R24 (cash signed, never clamped) / A16; `decisions.md` D64/D54
- Added: 2026-05-31 by HR
