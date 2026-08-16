---
id: B-compile-material-not-tick-gated
title: "material.authoring.compile_material flushes rendering commands and recreates live render state from the handler body, but is not in the tick-unsafe table"
status: OPEN
severity: Medium
category: bug
tags: [material, material-authoring, landscape, tick-safety, safepoint, render-flush, deadlock-risk]
---

# `compile_material` is hazard family B and is not gated

`Dispatch/SafePoint.cpp:24-36` defines hazard family B as "RE-ENTRANT SLATE TICK / VIEWPORT DRAW /
RENDER FLUSH from a handler body", and names the mechanism explicitly: *"FlushRenderingCommands from
inside the tick's parallel-task wait is a documented stall/deadlock source."*
`material.authoring.compile_material` now sits squarely in that family and is **absent** from
`GTickUnsafeMethodNames` (`SafePoint.cpp:46-138`; only `render.detect_z_fighting` at `:72`,
`python.execute`, and the two console verbs are listed).

## What the verb now does per call

- `MaterialAuthoringHandler.cpp` opens `FMaterialUpdateContext UpdateContext;` with default options
  (`RecreateRenderStates | SyncWithRenderingThread`). `bSyncWithRenderingThread` makes **both** the
  constructor and the destructor call `FlushRenderingCommands` (`MaterialShared.cpp:5026`, `:5078`).
- `MaterialLandscapeConsumers.h` then calls
  `ALandscapeProxy::UpdateAllComponentMaterialInstances(true)` **per consuming landscape**, which
  builds one `FComponentRecreateRenderStateContext` per landscape component plus a second
  `FMaterialUpdateContext` that also carries `SyncWithRenderingThread`, and
  `GetCombinationMaterial` adds a further `FlushRenderingCommands` per new combination MIC
  (`LandscapeEdit.cpp:620`).

So the flush count per call scales with the number of consuming landscapes and their components,
where before the change it was bounded by the pre-existing
`GShaderCompilingManager->FinishAllCompilation()` in `MaterialCompileErrorCollector.h`.

## Why this is filed rather than fixed

Stated so the severity is not over-read:

- The gap is **pre-existing, and widened** — not introduced. `compile_material` already called
  `FinishAllCompilation()` before this change, which is comparable, and it was never gated.
- `landscape.set_material` reaches the same engine code through `PostEditChangeProperty` and is
  **also** absent from the table, so the table is already inconsistent about this code path.
- No hang, stall or deadlock has been observed. The verb was driven repeatedly through the
  dispatcher during integration pass 11 without incident — but that exercised ordinary arrivals,
  not a mid-tick arrival, which is the actual hazard.

## Fix shape

A one-line entry in `GTickUnsafeMethodNames` for `material.authoring.compile_material`, per
`docs/rpc-design.md` §10 ("table entry preferred over a hand-written gate"). Decide at the same time
whether `landscape.set_material` belongs there too — the two now reach identical engine code, and
leaving one gated and the other not is the kind of inconsistency the table exists to remove. Any
entry needs a test, because gating changes when the response is delivered.

## History
- `#1-found-by-audit` `OPEN` reporter — Found by auditing the derived-state-honesty changeset
  (`099b83b3`..`0fe35187`) against `Docs/rpc-design.md`'s own "before you ship a verb" checklist
  during integration pass 11. The checklist line "Tick-unsafe work is gated through
  `Dispatch/SafePoint.h`" is the one that fails. Verified directly:
  `grep -n "compile_material" Dispatch/SafePoint.cpp` returns no table entry.
