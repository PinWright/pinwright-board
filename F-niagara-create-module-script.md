---
id: F-niagara-create-module-script
title: "No verb creates a Niagara module script from HLSL, graph mutators refuse standalone scripts, and create_node's CustomHlsl gets no typed pins"
status: IN-REVIEW
severity: Medium
category: feature
tags: [niagara, authoring, hlsl, custom-hlsl, module-script, graph, gap-analysis-2026-09-30]
---

# Niagara module scripts cannot be authored

Three linked gaps block an agent from expressing behavior that stock modules lack.

1. **No creation verb.** Nothing creates a `UNiagaraScript` module, function or dynamic-input
   asset. A grep of `Source` for `NiagaraScriptFactoryNew|create_script|create_module` finds
   nothing.
2. **Standalone scripts cannot be mutated.** Every `niagara.graph.*` mutator resolves through
   `NiagaraEdit::ResolveTarget`, which refuses anything that is not a System or Emitter:
   "Asset is not a Niagara System or Niagara Emitter" (`Handlers/Niagara/NiagaraEditTypes.cpp:1056-1060`).
   An existing module asset's graph therefore cannot be edited, even though `niagara.graph.get`,
   `niagara.inspect`, `niagara.validate` and NIR all read standalone scripts
   (`F-rpc-niagara-inspect-standalone-script`, DONE).
3. **CustomHlsl nodes get no typed pins.** `niagara.graph.create_node` with
   `nodeClass: NiagaraNodeCustomHlsl` writes `CustomHlsl` by reflection and calls
   `ReallocatePins` (`Handlers/Niagara/NiagaraGraphHandler.cpp:1040-1064`). Its comment says the
   signature is "rebuilt from the new HLSL source", which is false: function-call pins are
   allocated from `Signature` (`NiagaraNodeFunctionCall.cpp:366,465`; `Signature` at
   `NiagaraNodeFunctionCall.h:77`). The node ends up with no typed input or output pins, and no
   payload field can declare them.

## Competitors (compare row "HLSL module create", 2026-09-30)
- **Monolith: works, the model to follow.** `CreateScriptFromHLSL`,
  `MonolithNiagaraActions.cpp:5670-6152`; module and function wrappers at `:6154`, `:6159`.
  - Params: `{name, save_path, hlsl, description, inputs[], outputs[]}`. Unknown types and
    dotted names are rejected (`:5748-5780`).
  - Creates the asset with `NewObject<UNiagaraScript>` and a hand-built graph.
  - Fills the CustomHlsl `Signature` before `Finalize`, with the ParamMap first on each side
    (`:5909-5954`). This satisfies the pin-count check at `NiagaraNodeCustomHlsl.cpp:479`.
  - Wires Input → MapGet(`Module.<in>`) → HLSL → MapSet → Output (`:5956-6046`) using the
    wizard utilities through `PrivateIncludePaths` into NiagaraEditor/Private
    (`MonolithNiagara.Build.cs:32-56`).
  - Gaps: never compiles, so no errors are returned (`:6119`); no conflict check; the usage
    bitmask stays at its default.
- **ue-mcp: thin and broken.** `create_niagara_module_from_hlsl`, `NiagaraHandlers.cpp:429`,
  impl `:2927-2986`.
  - Uses the factory, then adds a CustomHlsl node whose `inputs`/`outputs` are only counted and
    echoed back (`:2965-2982`). The node has no `Signature`, no pins and no wiring, and the
    template's dummy graph is left in place.
  - It claims "pins are derived from the HLSL body", which is false.
  - Its `get_niagara_custom_hlsl` / `set_niagara_custom_hlsl` read/write pair
    (`NiagaraHandlers_StackEdits.cpp:1415,1473`) is a useful companion shape.
- **VibeUE:** scratch-pad modules only (`UNiagaraScratchPadService.cpp:410-496`), plus
  low-level `AddCustomHlslNode` / `AddPin` / `GetCompileMessages`. `AddPin` rebuilds `Signature`
  from pins (`:261`, `:825`). Scoped to a system; no standalone asset.
- **Epic NiagaraToolsets:** never creates modules. It only supports an inline single-rvalue
  expression on a stack input (`FNiagaraExt_StackInputData_HlslExpression`,
  `NiagaraExternalSystemEditorUtilities.h:583`); see `F-niagara-hlsl-expression-input`.

## UE APIs (5.8)
- **`UNiagaraModuleScriptFactory`** (`NiagaraScriptFactoryNew.h:41`) duplicates
  `/Niagara/DefaultAssets/DefaultModule` (`Config/DefaultNiagara.ini:86`), so creation starts
  from Input → MapGet(`Module.InputArg`) → MapSet(`Particles.DummyFloat`) → Output
  (`NiagaraScriptFactory.cpp:144-172`). The class carries no `_API` macro, so resolve it by
  reflection (`FindObject<UClass>(.../Script/NiagaraEditor.NiagaraModuleScriptFactory)`) and create
  through `IAssetTools::CreateAsset`.
- **`UNiagaraNodeCustomHlsl`** is `MinimalAPI` (`NiagaraNodeCustomHlsl.h:13`).
  `GetCustomHlsl`/`SetCustomHlsl` (:19-20) and `RebuildSignatureFromPins` (:84) are NOT exported.
  Keep the reflected `CustomHlsl` write, then call the virtual `RefreshFromExternalChanges()`.
  Set `ScriptUsage` (:28) to Module; the constructor default is Function (`.cpp:22`).
- **Map pin helpers:** `Wizard::Utilities::AddReadParameterPin` / `AddWriteParameterPin` are
  `NIAGARAEDITOR_API`-exported in the public `Widgets/Wizard/SNiagaraModuleWizard.h:123-124`.
  **The header exists on 5.5-5.8 and is absent on 5.3/5.4** (checked every local engine). The
  MapGet/MapSet classes themselves remain private and unexported
  (`Private/NiagaraNodeParameterMapGet.h:13`): find them by class name and pass them through a
  forward declaration. Wire with `UEdGraphSchema_Niagara::TryCreateConnection`.
- **Compile:** `UNiagaraScript::RequestCompile(VersionGuid, bForce)` (`NiagaraScript.h:1230`) is
  synchronous; it waits for the job (`NiagaraScript.cpp:3083-3088`). Errors can therefore be read
  back in the same call from `GetVMExecutableData()` (:1284): `LastCompileStatus` :511, `ErrorMsg`
  :529, `LastCompileEvents` :533 (the events carry node guids). This is the editor's own
  standalone path (`FNiagaraScriptViewModel::CompileStandaloneScript`,
  `NiagaraScriptViewModel.cpp:423`).
- **Usage bitmask:** `GetLatestScriptData()->ModuleUsageBitmask` (:798); its default covers
  particle stages only (`NiagaraScript.cpp:417`). `niagara.add_module` already enforces the
  bitmask (`B-niagara-add-module-ignores-usage-bitmask`).

## Proposed scope

**(a) Mutators accept standalone scripts.** `ResolveTarget` accepts a standalone
`UNiagaraScript` for `kind: graph|node|pin`, so `create_node`, `connect_pins`, `remove_node`
and `set_pin_default` work on module, function and dynamic-input assets. Add an editor-open
guard analogous to the emitter one, and verify an open script editor does not clobber the write.

**(b) Typed CustomHlsl pins.** `create_node` for `NiagaraNodeCustomHlsl` accepts
`payload.inputs` / `payload.outputs` as `[{name, type}]` and fills `Signature` (ParamMap first,
`bRequiresExecPin=false`) before `Finalize`. The response lists the typed pins read back off the
node. Fix the misleading comment at `NiagaraGraphHandler.cpp:1045-1051`.

**(c) New mutate verb `niagara.create_module_script`.**
- Params: `{assetPath, hlsl, inputs:[{name, type}], outputs:[{name, type, namespace}],
  usages:[...], description?, compile=true, save?}`. `usages` is required because no default is
  safe (`rpc-design.md` §3).
- Refuse when the asset already exists.
- Factory-create, strip the template's dummy pins and links, build and wire the CustomHlsl node,
  and set the bitmask. Then `RequestCompile(GetExposedVersion().VersionGuid, true)`.
- Response: `{assetPath, nodes, pins, usages, compile:{status, errors:[{message, nodeGuid,
  pinGuid?}]}, saved}`. Every field is measured. A failed compile keeps the asset but reports
  `compile.status: "failed"` and does not claim success.
- 5.3/5.4: return `UNSUPPORTED_ENGINE_VERSION` unless a substitute for the wizard pin helpers is
  found. Record the row in `docs/engine-version-support.md`.
- Round-trip readback needs no new code: `niagara.decompile_nir` already emits `customHlsl`
  blocks (`niagara.nir` wiki, "customHlsl @(x, y)").
- A `kind: function|dynamicInput` variant can follow with the matching factory and usage.

Out of scope: scratch-pad modules (VibeUE's approach). They are local to one system, and an
asset module plus `niagara.add_module` covers the same need.

## Unverified
- Whether `Output.<name>` or `Particles.<name>` is the right write namespace for module outputs.
  Monolith writes `Output.`; the factory template writes `Particles.`.
- Whether `RelayoutGraph` is exported.
- A 5.3/5.4 substitute for `AddRead/WriteParameterPin`.

## Acceptance
- Create a module with `inputs [Scale: float]`, `outputs [Velocity: vec3]` and
  `usages [ParticleUpdate]`. `compile.status` reports a measured success, and NIR decompile shows
  the wired `customHlsl` block.
- A deliberately bad HLSL line returns `compile.errors` carrying the CustomHlsl node guid.
- `niagara.add_module` accepts the new module into ParticleUpdate and rejects it from
  SystemSpawn.
- `niagara.graph.create_node` on the new module asset succeeds (it is refused today with
  `UNSUPPORTED_ASSET`).
- A CustomHlsl node created with `payload.inputs` reads back with those typed pins.
- An existing asset path is refused. 5.3/5.4 either work or refuse with
  `UNSUPPORTED_ENGINE_VERSION`.

Effort M-L: (a) S, (b) S-M, (c) M. Risk: unexported setters (the reflection pattern already in
use covers them), the 5.3/5.4 guard, and the output-namespace question. Value: this is the only
way for an agent to author custom per-particle behavior.

## History
- `#1-filed-from-gap-analysis` `OPEN` reporter — Filed from the 2026-09-30 competitor gap analysis (compare row "HLSL module create": PinWright partial, ue-mcp/VibeUE/Monolith yes). Competitor code read at the audited commits under X:\tmp\uemcp-cmp: only Monolith builds a working graph; ue-mcp's node is pinless and unwired; none returns compile errors for a standalone module. Engine citations are UE 5.8 source reads. The wizard-helper range was checked against local 5.3-5.8 installs. Nothing was built or run.
- `#2-implemented` `IN-REVIEW` developer — (a) `NiagaraEdit::ResolveTarget` accepts a standalone `UNiagaraScript` for graph/node/pin kinds and resolves its one graph; the same applies to `ResolveNiagaraGraph`, used by `niagara.graph.connect_pins` and `remove_node`. Both refuse `EDITOR_OPEN` while the script toolkit holds the asset, through the new `RefuseScriptAssetEditWhileToolkitOpen` in `NiagaraEditorOpenGuard.h`, because the toolkit edits a duplicate and overwrites the asset on Apply. `compile:true` on a script runs the synchronous `RequestCompile`. Files: `NiagaraEditTypes.cpp` and `NiagaraGraphHandler.cpp`. (b) `create_node` CustomHlsl: `payload.inputs` / `outputs` `[{name,type}]` fill `Signature` before `Finalize` (ParamMap `Map` first on each side, `bRequiresExecPin=false`). Names must be identifiers and types go through `ResolveNiagaraParameterType`. Pins now carry `niagaraType`, and the misleading comment is fixed. Helpers are `ParseCustomHlslPinSpecs` / `SetCustomHlslSignature` in `NiagaraGraphCreateNodePayload.h`. (c) New `niagara.create_module_script` in `Handlers/Niagara/NiagaraCreateModuleScriptHandler.cpp` (+ `NiagaraScriptCompileReport.h/.cpp`). It creates the asset from the reflected `NiagaraModuleScriptFactory`, strips the template body and its stale script variables, and builds Input -> MapGet(`Module.<in>`) -> CustomHlsl -> MapSet(`<ns>.<out>`) -> Output with checked links. It then sets the bitmask from the required `usages`, compiles synchronously and reports `compile.status` / `errors`. Outputs require `namespace` (Particles/Emitter/System/Output/Local/Transient/StackContext); this settles the Output-vs-Particles question by making the caller choose. 5.3/5.4 get `UNSUPPORTED_ENGINE_VERSION`, recorded in `docs/engine-version-support.md`. **Known gap against the acceptance list:** a syntax error in the HLSL body is reported by the VM backend compiler with no node guid (`NiagaraCompiler.cpp:2258` adds bare-text events). The response therefore includes `nodeGuid` only when the engine attributes an error, plus a top-level `customHlslNodeGuid` for correlation; it does not fake the attribution. Tests: `PinWright.niagara.create_module_script.{CreatesWiredCompiledModule,BadHlslReportsFailedCompile,RefusesExistingAssetAndInvalidArguments,AddModuleHonorsUsages}`, `PinWright.niagara.graph.create_node.StandaloneScriptCustomHlslTypedPins` (`Tests/Niagara/TestNiagaraCreateModuleScript.cpp`). Wiki: `niagara.md`, `niagara.graph.md`. Compile-checked with UBT -SingleFile; not yet run.
- `#3-fixround-scoped-fixture` `IN-REVIEW` developer — Full-suite fix round. `AddModuleHonorsUsages` was the one unreviewed rooted-producer call (`BuildEmptySystemWithEmitter`) flagged by `PinWright.infra.contract.AddToRoot.ScopedFixtureOwnership`. It now uses `NiagaraEditTestUtils::DuplicateFixtureSystemWithEmitters`, held by `TStrongObjectPtr` with `CleanupTestAsset` on scope exit, and the fixture's first emitter. Nothing in the file is rooted. The other four create_module_script tests and `graph.create_node.StandaloneScriptCustomHlslTypedPins` passed in that run. Compile-checked with -SingleFile; not re-run.
- `#4-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Passed in w23-final: `PinWright.niagara.create_module_script.CreatesWiredCompiledModule`, `.BadHlslReportsFailedCompile`, `.RefusesExistingAssetAndInvalidArguments`, `.AddModuleHonorsUsages` and `PinWright.niagara.graph.create_node.StandaloneScriptCustomHlslTypedPins`, covering Acceptance bullets 1, 3, 4, 5 and the existing-path refusal. Not met: bullet 2 asks that a bad HLSL line return `compile.errors` carrying the CustomHlsl node guid; the VM backend reports HLSL syntax errors without a node guid, so the response gives a top-level `customHlslNodeGuid` instead (deliberate deviation). The 5.3/5.4 `UNSUPPORTED_ENGINE_VERSION` path is unverifiable here (5.8 only). Needs: owner acceptance of the guid deviation and a 5.3/5.4 build check.
