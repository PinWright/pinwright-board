---
id: B-nir-stack-omits-override-only-module-inputs
title: "NIR stack module rows never emit `input X = ...` for regular module inputs: overrides (linked parameter, dynamic input, override-pin literal) on Module.* inputs are silently dropped"
status: IN-REVIEW
severity: High
category: bug
tags: [gap-analysis-2026-09-28, niagara, nir, decompile, override-chain, silent-omission]
encounters: 1
lastSeen: 2026-09-29T18:18:36Z
---

# NIR drops every override on a regular module input

`AppendOverrideInputs` (`Source/PinWright/Private/NIR/NIRDecompiler.cpp`) builds its input list
from the module's `UNiagaraNodeFunctionCall::Pins` only, then looks each pin name up in the override
node's pins. A stack module's function-call node carries only the parameter-map pin, static-switch
pins and exposed `UNiagaraNodeInput` pins (`UNiagaraNodeFunctionCall::AllocateDefaultPins`,
`C:\UE_5.8\Engine\Plugins\FX\Niagara\Source\NiagaraEditor\Private\NiagaraNodeFunctionCall.cpp:398-403`
creates input pins only for `InputNode->IsExposed()` input nodes). Regular module inputs such as
`SpawnRate` are `Module.*` parameters read through a Map Get inside the module graph, so they
have no pin there. Their overrides exist only as `<Function>.<Input>` pins on the stack override
`UNiagaraNodeParameterMapSet`. So `CollectOverridePinsByInputName` finds the override, but nothing
ever asks for it.

Effect: `niagara.decompile_nir` and every `nir.txt` sidecar omit linked-parameter, dynamic-input
and override-pin literal values on ordinary module inputs, with no warning. The documented v1b
contract (`docs/wiki-src/niagara.nir.md`, "Module input override-chain expansion") is unmet. The
`#3` verification on `F-niagara-decompile-nir-overrides` counted `input` lines, but those come from
function-call pins (enum/DI inputs), not from override chains.

Evidence: `PinWright.niagara.decompile_nir.OverrideLiteralFloat` / `OverrideLinkedParam` /
`OverrideDynamicInput` ran for the first time after `B-niagara-tests-spawnrate-wrong-path` fixed
their fixture path. All three fail with no `input SpawnRate = ` line at all
(`Saved/PinWright/test-runs/7b2d4d4c5a494988a2c90b28389aad29/automation.log`, test slices at
`TestNIRDecompiler.cpp:1045/1082/1122` failures). Also `Saved/PinWright/asset-dumps/App/App/FXE_Trail/nir.txt:196`:
`module SpawnRate @0 enabled` carries only its `static` line. The graph view of the same module
(`:364`) lists only the static-switch pin, which confirms the call node has no `SpawnRate` pin.

## History
- `#1-override-only-inputs-dropped` `OPEN` reporter — `AppendOverrideInputs` enumerates only function-call pins, so override-node pins for `Module.*` inputs (no call-node pin) are never emitted. Surfaced by the three NIR override tests failing on their first real run. The same pin-only enumeration exists in `EmitDynamicInputBody` (`NIR/NIRTextEmitter.cpp`), so inputs of a nested dynamic input are dropped too.
- `#2-walk-override-node-inputs` `IN-REVIEW` developer — `AppendOverrideInputs` now builds its input-name set as the union of the call node's non-static input pins and the override node's un-aliased input names, excluding static-switch names, sorted lexically as before. Each name emits the override-chain expression when an override pin exists, else the call pin's literal (unchanged path). `docs/wiki-src/niagara.nir.md` now states where regular-input `input` lines come from. `-SingleFile` compile of `NIRDecompiler.cpp` on UE 5.8: `Result: Succeeded`. **Not fixed here:** `EmitDynamicInputBody` still walks only the dynamic-input call node's pins, so overrides on a nested dynamic input's own inputs are still dropped (`OverrideRecursionDepth` warns `Could only author 1-deep chain` for the matching fixture-side reason). Not run: verification is the next run of `PinWright.niagara.decompile_nir.Override*`. The linked-parameter case also needs `B-nir-linked-param-map-get-unclassified`.
- `#3-nested-dynamic-inputs-and-shared-lookup` `IN-REVIEW` developer — Closed the gap `#2` left open. The override lookup is now one shared function, `CollectOverridePinsByInputName(UNiagaraNodeFunctionCall&)`. It is declared in `NIR/NIRTextEmitter.h` and defined in `NIRTextEmitter.cpp`, where it resolves the override node via `NiagaraResetModuleInput::FindStackFunctionOverrideNode` and keeps pins in the call's `<FunctionName>.` alias namespace. The anonymous-namespace copy in `NIRDecompiler.cpp`, and the two includes only it used, are removed. `EmitDynamicInputBody` now uses the same lookup: a call-node pin prefers its override pin, and override-only inputs follow in lexical order, each recursing at `Depth + 1`. Nested dynamic-input chains therefore render, and the depth-32 guard is reachable. `PinWright.niagara.decompile_nir.OverrideRecursionDepth` now requires a >32-deep chain plus the marker and the warning (see `B-niagara-tests-spawnrate-wrong-path` `#5`). Header touched: `NIR/NIRTextEmitter.h` (one forward declaration, one function declaration). `-SingleFile` on UE 5.8: `NIRTextEmitter.cpp`, `NIRDecompiler.cpp`, `NIRGraphEmitter_Dataflow.cpp` (a header includer) Succeeded. Not run.
- `#4-nir-aspect-version-bump` `IN-REVIEW` developer — `nir.txt` bytes change with `#2`/`#3` and with `B-nir-linked-param-map-get-unclassified`, so its aspect version in `Handlers/Asset/AssetDumpCache.cpp` goes from 3 to 4 (the reason is noted beside the entry). Cached dumps regenerate instead of serving the old text. `-SingleFile` on UE 5.8: Succeeded.
