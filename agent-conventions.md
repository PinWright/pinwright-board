---
type: guide
summary: "Distilled PinWright plugin conventions + invariants for fix-workflow agents. Not a ticket."
---

# PinWright agent conventions (per-ticket briefing)

**The source is the arbiter.** This doc lives on the board, decoupled from plugin versions. Treat every
line as a strong prior and verify against the current source under `Plugins/PinWright/Source/` before
acting on it. If this doc and the code disagree, the code wins — then propose an edit here.

Deep-dive docs (in the plugin clone, `Plugins/PinWright/`):
- `CLAUDE.md` — agent conventions, wire protocol, wiki authoring, aspect-version bumping
- `Docs/arch.md` — layers, request flow, FHandlerContext, jobs, dual-surface, testing architecture
- `Docs/lessons.md` — one-line UE API traps (the section 8 source; check it for anything not listed here)
- `Docs/test-organization.md` — test taxonomy, placement, naming
- `Docs/error-code-catalog.md` — generated inventory of all `SendError` codes

Updating THIS doc: append dated one-liners under "Recently learned" and commit via `board-commit.ps1`
(never plain `git` on the board). The normal channel for substantive edits is workflow-log-audit's
user-approved proposed-edits.

## Module map (`Source/PinWright/Private/`)

- `Transport/McpTransport.h/.cpp` — HTTP `POST /mcp` only; JSON-RPC 2.0; 5 protocol methods; async completions; oversize spill to `Saved/PinWright/HttpResponses`.
- `JsonRpc.h` — envelope helpers + `JsonRpc::Serialize` (use it; never hand-roll a local `TJsonWriterFactory` block — 3 dupes existed in `McpTransport.cpp` alone).
- `Dispatch/RpcDispatcher.h/.cpp` — TMap dispatch; game-thread marshal; GC/save deferral; reentrancy guard; auto-validates required params; strict `UNKNOWN_PARAMS` rejection.
- `Catalog/` — `ToolCatalog`, `WikiHandler` (discovery surface), `WikiOverlay`, `WikiDiskGenerator` (writes `wiki-generated/` at launch from `docs/wiki-src/`).
- `Public/PinWrightSubsystem.h` + `PinWrightSubsystem.cpp` — orchestrator only, no domain state; 0.1s ticker.
- `Handlers/<Domain>/*.cpp` — ~90 files, ~1,200 methods (Actor/, Blueprint/, Material/, UI/, Niagara/, Recorder/, Spatial/, ...). **New handlers go here**, one `.cpp`, no header, no manual wiring.
- `Handlers/HandlerRegistration.h`, `Handlers/ParamSpec.h`, `Handlers/HandlerContext.h` — the handler framework.
- `State/PluginState.h/.cpp` — `FPluginState` singleton: `FBlueprintTracker`, `FSaveThrottler`, `FJobRegistry`, Sequencer/Niagara registries; all access via `FCriticalSection`.
- `Utils/` — `PathUtils`, `AssetUtils` (incl. `SpawnActorInActiveWorld<T>` — there is NO `ActorUtils::SpawnActorWithLabel`, a recurring ticket fiction), `ClassUtils` (`ResolveUClass` 3-tier), `JsonUtils`, `PropertyUtils`, `JsonBuilders.h` (canonical JSON shapes), `IrSidecarRegistry.h`.
- `Tests/<Domain>/` — compiled into the main module DLL; there is NO separate tests module. **New tests go here.**
- Shared helpers used by ≥2 sibling handlers: named-namespace header co-located in the handler subdirectory (e.g. `Handlers/Material/MaterialFinders.h`); promote to `Utils/` only on cross-cluster reuse.

## Handler + dispatcher patterns

- Register with `REGISTER_RPC_HANDLER("ns.verb", "ns", "Summary", RPC_PARAMS(...))` — static auto-registration, drained at subsystem init (`Handlers/HandlerRegistration.h`).
- `RPC_PARAM_REQ/OPT/DEF(Name, Type, Desc[, Default])` — `RPC_PARAM_DEF`'s default MUST be a string literal (`"false"`, not `false` — the macro wraps it in `TEXT(...)`).
- The dispatcher rejects payload keys not declared in the ParamSpec (`UNKNOWN_PARAMS`, `Dispatch/RpcDispatcher.cpp`) — every accepted alias must be its own `RPC_PARAM_OPT`; an undeclared alias read via `GetStringFirstOf` is dead code in production.
- `FHandlerContext` validators are 2-arg out-param overloads that auto-send the error: `FString S; if (!Ctx.RequireString(TEXT("x"), S)) return true;`. There is NO `HadError()` and NO 1-arg `RequireString` — a frequently hallucinated signature that does not compile. Header `Handlers/HandlerContext.h` is the source of truth.
- Call `SendSuccess` / `SendError` exactly once per request. 3-arg `SendError(Code, Message, Result)` attaches a structured payload that reaches the client as `structuredContent`.
- All handlers run on the game thread; requests defer during GC/serialization; don't spawn threads that touch UObjects.
- Async completion (AsyncTask / next-tick lambda): capture `Ctx.MakeAsyncToken()` and respond via `Token->SendSuccess/SendError`. Calling the subsystem's `SendAutomationResponse` directly from a lambda bypasses test capture and `REGISTER_RPC_FORMATTER` — anti-pattern. Canonical consumer: `Handlers/Environment/LandscapeHandler.cpp`.
- Wrap EVERY mutation in `FScopedTransaction` — placed after validation, before the first mutation. Without it, blueprint recompiles create `REINST_` classes the `UTransBuffer` holds stale refs to → PIE ensures in `CheckAndHandleStaleWorldObjectReferences`.
- Never fake-success a feature the running engine can't do — `Ctx.SendUnsupportedEngineVersion(RequiredVersion, Feature)`.
- Optional-engine-plugin handlers: `REGISTER_RPC_HANDLER` is ALWAYS unconditional (namespace must appear in the wiki); only the body branches on a `__has_include`-derived macro; `Build.cs` soft-links via `TryAddConditionalModule`. Canonical: `Handlers/Water/WaterHandler.cpp`.
- The transport timeout is response-only — the handler keeps running and its work commits. Mutating handlers must be idempotent under client retry (upsert, not append).
- Long-running work: manual opt-in `Ctx.StartJob(...)`; clients poll `system.job_status`. No auto-promotion exists.
- A verb that reads OR mutates based on a boolean (`dryRun`) is wrong — split into two verbs (precedent: `blueprint.graph.find_orphaned_nodes` / `delete_orphaned_nodes`).
- Mutating material handlers load via `LoadMaterialForMutationOrReportError` (`Handlers/Material/MaterialFinders.h`) — a bare `LoadObject` mutates a copy the open `FMaterialEditor` silently discards; fail loud with `EDITOR_OPEN`.
- Dual-surface rule: one shared builder per structured output; asset-dump sidecar and live RPC both call it. New IRs register with `REGISTER_DECOMPILE_IR` (`Utils/IrSidecarRegistry.h`), never new dispatch branches in `AssetDumpHandler.cpp`.
- Any change to serialized sidecar bytes bumps that aspect's version in the `Versions` map (`Handlers/Asset/AssetDumpCache.cpp::GetAspectVersion`) in the SAME commit — else stale caches keep serving pre-fix bytes and the fix "doesn't work" from the client.

## Error conventions

- Codes are domain-specific UPPERCASE (`ASSET_NOT_FOUND`, `SPAWN_FAILED`); full generated inventory: `Docs/error-code-catalog.md` (553 unique codes; registry migration tracked in ticket `E-error-code-vocabulary-registry`).
- Every `SendError` code a NEW handler emits must also be registered as an `ERR_<CODE>` constant in `Handlers/ErrorCodes.h` (alphabetical order, `=` column-aligned) — the suite test `PinWright.core.error_codes.AllEmittedCodesAreRegistered` scans emit sites against that header and fails the WHOLE suite otherwise. Register the codes in the SAME edit that introduces them (a missing registration costs a full compile+test cycle to discover).
- Every error must carry diagnostic context: what was searched, what class/path was checked, and an actionable hint. Generic "Could not resolve X" makes callers blame the tool instead of their input.
- Never return bare `compiled: true` from anything IR/graph-shaped — surface `UBlueprint::Status` and `FCompilerResultsLog` entries; IR compile success ≠ Kismet compile success.
- `UNSUPPORTED_ENGINE_VERSION` is reserved for the `SendUnsupportedEngineVersion` helper.
- Handler errors travel inside the `tools/call` result (`isError: true`), NOT as JSON-RPC envelope errors; envelope codes (-32700..-32603) are transport-level only.
- Internal/unexpected errors (empty code, `INTERNAL`, `ERR_AUTOMATION_ERROR`) auto-carry a `call("support")` report hint — don't use those codes for caller mistakes.

## Tests + fixtures

- `IMPLEMENT_SIMPLE_AUTOMATION_TEST` only; category string `PinWright.<Group>.<Case>`; flags `EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter` (NOT `ApplicationContextMask` — doesn't compile standalone).
- Files under `Source/PinWright/Private/Tests/<Domain>/`, mirroring the taxonomy in `Docs/test-organization.md` (Assets, World, Gameplay, EditorOps, Media, Blueprint, Networking, Utility, Bpir, Infra, Core, WidgetXml).
- The macro's category string is authoritative, NOT the filename — several files use prefixes that don't match their names (`TestBlueprintGraphOrphan.cpp` → `OrphanDetection.*`). Grep `IMPLEMENT_*_AUTOMATION_TEST` empirically before any rename/count.
- Bucket naming: internal utils are `Core.*` (not `Utils.*` — collides with the RPC `Utility` domain); editor-domain is `EditorOps.*` (avoids `Editor.Editor` stutter).
- Fixtures: use THIS host project's content (EAContentExamples57, e.g. `/Game/Characters/Mannequins/Meshes/SK_Mannequin`) or build fixtures in-code. NEVER reference Lyra content or Lyra-only modules.
- A missing required fixture is a test FAILURE (assert and fail) — never a `return true` skip; skips silently green the suite while covering nothing.
- Log-reading: a passing `TestTrue`/`TestEqual` logs NO event line. Empty `BeginEvents..EndEvents` + `Result={Success}` IS the passing signature — don't misread silence as "test didn't run".
- Regression tests must call the SAME production symbol the handler uses — a test re-implementing the fix as a local lambda stays green after reverting the fix (false positive). Promote fix logic to a namespace-scope helper with a header declaration.
- Regression tests must have demonstrably failed pre-fix (differential proof), and byte-equality round-trip tests need at least one explicit content assertion — symmetric loss (both halves drop the same line) defeats pure equality.
- `FAutomationTestBase::TestEqual` has no `UObject*` overload — compare pointers via `TestTrue(What, A == B)` or cast to `void*`.
- `FVector2D` components are `double` — `TestEqual(What, Pos.X, 50.0f)` is an overload ambiguity; pass `50.0`.
- Handlers that dereference `Ctx.GetSubsystem()`: do NOT use `MakeTestContextWithCapture` (null subsystem → crash). Fetch `GEditor->GetEditorSubsystem<UPinWrightSubsystem>()` + `MakeContextWithCapture`; pattern in `Tests/*/TestSystemHandlers.cpp`, `TestUIHandlers.cpp`.
- Capture tests can't see wire shaping — anything about what the CLIENT sees (`content`, `structuredContent`, `isError`, spill markers) must go through the live loopback seam in `Tests/Infra/TestMcpTransport.cpp` (real transport on port 19981 + real HTTP POST).
- Shared test helpers between sibling `.cpp`s: named-namespace `<Cluster>TestHelpers.h` (e.g. `Tests/Recorder/RecorderTestHelpers.h`). Anonymous-namespace duplicates are a LATENT unity ODR error — adaptive unity hides it while files are git-modified and it detonates on the first clean post-commit build.
- A feature macro `#define`d only inside a handler `.cpp` reads as 0 in test TUs — guarded test bodies become dead code with no warning. Define version/feature macros in a shared header.
- Niagara fixtures: `NiagaraEditTestUtils::NewTransientSystem` + `PinWrightNiagara::AcquireSystemViewModel` — bare `NewObject<UNiagaraSystem>(GetTransientPackage())` bypasses the SVM and editor invariants silently.
- CLI run: `UnrealEditor-Cmd.exe <proj>.uproject -ExecCmds="Automation RunTests PinWright;Quit" -unattended -nopause -nocefaccelpaint -log`. `-nocefaccelpaint` is required (CEF assert kills the suite mid-run); do NOT substitute `-NullRHI`. Verify via the log: `rg "Result=\{Fail\}"`, never process exit code alone.

## Memory / lifetime / undo

- Renaming UObjects to a fixed base name: always `MakeUniqueObjectName()` — `UObject::Rename()` with a hardcoded name fatally asserts on a same-Outer collision.
- `Transaction.Cancel()` does NOT reverse `NewObject` creation — error rollback needs manual cleanup (e.g. `RemoveWidgetSubtree`).
- `MarkBlueprintAsStructurallyModified` goes INSIDE the `FScopedTransaction`, not after it — skeleton regen doesn't flush undo.
- Programmatic undo: `GEditor->UndoTransaction()`, never `Exec("Undo")`.
- During authoring, resolve Blueprint members skeleton-first: query `SkeletonGeneratedClass`, fall back to `GeneratedClass` — the skeleton is a superset during authoring; `GeneratedClass` can be stale until a full compile.
- `RegenerateSkeletonOnly` compiles do NOT scrub `GeneratedClass->Children`/`FuncMap` — deleting a node graph + skel-only recompile leaves an orphan UFunction that crashes `TFieldIterator<UFunction>` on cold reload. Scrub the specific UFunction in the same transaction; NEVER call `FBlueprintEditorUtils::RemoveStaleFunctions` (wipes user-authored functions too).
- Multicast completion delegates can fire twice (`OnLightingBuildFailed` AND `OnLightingBuildKept` on cancel) — guard resolve closures with a shared `bFired` flag (`LevelBuildBinds.h::BindLightingBuildCompletion`).
- `FTSTicker` poll-completion: require an observed `inProgress=true` tick before treating `!inProgress` as done — the first tick fires before the operation starts (false positive).
- Diffing a property against a parent CDO: guard `ParentContainer->GetClass()->IsChildOf(Property->GetOwnerClass())` first — leaf-declared properties have no parent slot and `Identical` crashes on non-POD types (`Utils/PropertyUtils.cpp::ExportPropertyToJsonValueWithInheritance`).
- Override detection needs `PPF_DeepComparison` — `PPF_None` misses value-equality in nested containers.

## Lookalike-API traps (highest recurrence — verify each in `Docs/lessons.md`)

- `FindPin(TCHAR*)` uses `FName(..., FNAME_Find)` → silent `NAME_None` miss on dynamically created pins. Use `FindPin(FName(...))` or iterate `Node->Pins` by string.
- `EFieldIteratorFlags` (for `TFieldIterator<T>`) vs `EFieldIterationFlags` (for 3-arg `FindFProperty`) — visually similar, not interchangeable.
- `FindFProperty` does NOT traverse super for delegate/multicast properties by default — pass `EFieldIterationFlags::IncludeSuper` or inherited delegates resolve null.
- `FindObject<T>(nullptr, shortName)` searches ONLY the root package — misses game/plugin types. Use the 3-tier chain: full-path load → `FindFirstObjectSafe` → `TObjectIterator` fallback (`ResolveUClass`/`ResolveUEnum` in `Utils/ClassUtils`).
- `HasAnyFunctionFlags(A | B)` is OR; `HasAllFunctionFlags(A | B)` is AND — use All when both flags are required.
- `UE_VERSION_NEWER_THAN(5, X, 0)` is strict and EXCLUDES 5.X.0 — "5.X or newer" is `UE_VERSION_NEWER_THAN_OR_EQUAL(5, X, 0)`.
- `FClassProperty` inherits `FObjectProperty` — `CastField<FObjectProperty>` matches both; check `FClassProperty` FIRST in CastField chains.
- TWO pin-type converters exist: `ConvertCppTypeToPinType` (BPIR compiler) and `MakePinType` (RPC handlers) — new types must be added to BOTH or pins silently become Wildcard in the other path.
- `PC_Wildcard` pins are silently dropped by `ReconstructNode` — verify the converter produced a concrete category before `CreateUserDefinedPin`.
- ALWAYS check `TryCreateConnection` return values — unchecked failures report broken wiring as successful compilation (~10 historical call sites).
- Find exec pins with `UEdGraphSchema_K2::FindExecutionPin(Node, Direction)`, not `FindPin(PN_Execute)` — macro nodes (ForEachLoop) name their entry pin `"Exec"`.
- `Cast<UK2Node_CustomEvent>` misses `UK2Node_ComponentBoundEvent` (derives `UK2Node_Event`); `UK2Node_InputKey` derives plain `UK2Node` — event sweeps must handle each explicitly.
- `UMaterialInstanceConstant` is NOT a `UMaterial` (`UMaterialInstance : UMaterialInterface`) — `Cast<UMaterial>` dispatch arms skip MICs entirely.
- `Cast<UMaterialExpressionParameter>` misses `UMaterialExpressionTextureSampleParameter` (parallel hierarchy) — detect parameters via `HasAParameterName()` + `GetParameterName()`.
- `FPackageName::DoesPackageExist` wants a bare package name — convert object paths via `ObjectPathToPackageName` first or it fires an `ensure()`.
- `StaticFindObject` on a bare long-package path (`/Game/Foo/Bar`) matches the outer `UPackage`, not the asset — non-null result + failed `Cast<T>` is the symptom; retry the `Package.AssetName` form (`MetaSoundPathUtils.cpp::LoadMetaSoundDocumentAsset`).
- `UMaterialExpressionMaterialFunctionCall::GetInputName(i)` returns the DECORATED `"Name (S)"` form — raw name is `GetInputNameWithType(i, false)`; outputs are undecorated.
- `FMaterialInstanceParameterUpdateContext` dtor fires `PostEditChange` + permutation rebuild — calling `PostEditChange()` again after the scope double-compiles shaders.
- Every `FPostProcessSettings` value write needs the paired `bOverride_<Field> = true` — without it the write silently no-ops at composition time.
- `FJsonObject::Values` is hash-ordered — any JSON written to disk for diff/round-trip must sort keys (`SerializeSortedJsonObject` in `Handlers/Asset/AssetDumpHandler.cpp`).
- `IAssetRegistry::ScanPathsSynchronous` is expensive even with `bForceRescan=false` — skip it unless `IsLoadingAssets()` is true.
- Path containment: `Dir.StartsWith(Root)` passes when `Dir == Root` — require the STRICT subpath `Root + "/"` before any recursive delete (`Utils/AssetDumpWriter.cpp`).
- Engine `UCLASS` types without a `*_API` export macro can't be referenced cross-module via `StaticClass()`/`NewObject<T>`/`Cast<T>` (LNK2019) — resolve by reflection: `FindObject<UClass>(nullptr, TEXT("/Script/Module.Class"))` + exported-base construction + `IsA` + `static_cast`.
- `TArray::Pop(bool)` / `RemoveAt(int, bool)` are deprecated — use the `EAllowShrinking` enum overloads.
- `FEditorDelegates::OnEditorClose` does not exist — shutdown hooks use `FCoreDelegates::OnPreExit`.

## Build invariants

- Unity + shared PCH are ON (`PinWright.Build.cs`) — shared helpers live in NAMED-namespace headers, never anonymous namespaces duplicated across sibling `.cpp`s (ODR collision when unity merges TUs; adaptive unity masks it until a clean build).
- Version branching: `Misc/EngineVersionComparison.h` macros + `__has_include()` for moved headers. NEVER bare `ENGINE_MAJOR_VERSION`/`ENGINE_MINOR_VERSION` in `#if`. The module builds on UE 5.3–5.7.
- Includes of subdirectory headers use the full path from the `Private/` root (`#include "Handlers/Audio/MetaSound/Foo.h"`) — bare `#include "Foo.h"` only resolves from the adjacent `.cpp`.
- Optional engine modules: `TryAddConditionalModule` in `Build.cs`, never a hard `PublicDependencyModuleNames.Add`.
- `StructUtils` is a separate linked module only on UE ≤5.4 (in CoreUObject from 5.5) — gated in `Build.cs`.
- Build target for this host: `EAContentExamples57Editor` via `Build.bat ... -Project=<repo-root>\EAContentExamples57.uproject -Log=<per-host path> -WaitMutex` (never generic `UnrealEditor`; see host `CLAUDE.md` for the collision caveats).

## Board interaction (minimal — mechanics live in the workflow prompts + board `README.md`)

- Do not re-derive board mechanics here; the board README (`X:\src\unreal\.pinwright-board\README.md`) and the workflow prompts are authoritative.
- Never flip OPEN → DONE — only the tester workflow closes tickets; fixers stop at IN-REVIEW.
- Board commits go ONLY through `board-commit.ps1` (machine-global mutex + pathspec-scoped add/commit) — plain `git add`/`commit` on the board races the other three hosts.
- Bare lease writes (`claimedBy`/`claimedAt`, stale-lease clears) are never committed — only meaningful create/status/history changes.
- Harness (workflow-skill) defects go to `harness/H-*.md` per `harness/README.md` — not to the main board, and not fixed mid-run.

## Process norms

- Fix the process, not the output: patching a generated/derived artifact leaves the generator broken.
- If a workaround needs a paragraph of justification, the code is wrong — fix the code.
- Regression tests must have demonstrably failed pre-fix (run them against the pre-fix code, or revert-and-run) — an always-green test proves nothing.
- Most test failures exposed by adding error checking are REAL production bugs surfacing, not test regressions — fix production code, don't relax the test.
- Prefer source-level fixes + narrow invariant checks over post-hoc heuristic validation — enumerating engine-synthesized patterns from outside the engine is a losing fight.
- Tickets are reporter-best-guess, not API references: re-grep cited line numbers, verify prescribed UE class names against engine source before writing an include, and re-grep production code for the acceptance symbol before trusting any IN-REVIEW claim.

## Recently learned (append-only)

<!-- Dated one-liners appended via board-commit.ps1; entries here are candidates for promotion into the sections above. -->
