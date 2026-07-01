---
id: B-bpir-func-with-space-unresolvable
title: "BPIR cannot reference BP functions whose name contains a space"
status: DONE
severity: High
category: bug
tags: []
---

# BPIR cannot reference BP functions whose name contains a space

BPIR `call` can't resolve a Blueprint function whose name contains a space. UE Blueprint functions can have spaces in their display/internal name (created via the editor's "Rename" with a space in the identifier); the decompiler emits such names verbatim (e.g. `entry function Set Error(string Error) {...}`), but the compiler rejects every reasonable call form.

**Repro:**
1. Widget BP `/App/App/UI/W_Error` has one function: `Set Error(Error: string)` (with literal space).
2. `blueprint_decompile_bpir` emits: `entry function Set Error(string Error) { ... }` — confirms the name is `Set Error` with space.
3. Attempt to call from BPIR:
   ```
   call Set Error(Target: %w, Error: %errStr)         → parser splits on space; "Error" parsed as next token
   call SetError(Target: %w, Error: %errStr)          → "Unresolved function: 'SetError'"
   call Set_Error(Target: %w, Error: %errStr)         → "Unresolved function: 'Set_Error'"
   call "Set Error"(Target: %w, Error: %errStr)       → "Unresolved function: '\"Set Error\"'"
   ```

**Impact:** Blocks the replay subtask 02 `W_Error.Set Error` call from `W_ReplaySaveHandler`'s `Failed` state branch. The popup pushes successfully but the error text can't be set via MCP — forces manual wiring of that single call.

**Workaround:** Rename `Set Error` → `SetError` in the BP editor. But that's exactly what "fully automated via MCP" is supposed to avoid, and the rename itself isn't exposed via MCP either.

**Proposal:** BPIR function resolver should:
1. Accept quoted function names: `` call `Set Error`(...) `` or `call "Set Error"(...)` (current quote handling treats the whole string including quotes as the name).
2. As a fallback, match function display-names by space-stripping both sides before comparison.
3. Decompiler should emit quoted form for names containing spaces so round-trip works.

## History
- `#1-initial-repro` `OPEN` reporter — Hit while implementing W_ReplaySaveHandler's Failed→popup flow in replay subtask 02. The Lyra project conventions include several BP functions with spaced names (e.g. `Show Single Race Info`, `Show Race Info`, `Set Error`). Decompiler observed emitting them verbatim; compiler observed rejecting all call forms.
- `#2-backtick-parser-fix` `IN-REVIEW` developer — Four changes: (1) `BpirParser.cpp::ParseCallInstruction` now finds the `(` after the closing backtick when the name starts with `` ` ``, then unwraps the inner text as `FunctionName` (e.g. `` call `Set Error`(...) `` → `FunctionName == "Set Error"`). (2) Entry-block backtick unwrap at ~lines 537–545 now covers any entry kind (guard changed from `Macro`-only to unconditional); `Function` entries with spaced names are now unwrapped. (3) `BpirTextEmitter.cpp::CollectEntrySignature` function-entry path (~line 542) wraps `GraphName` in backticks when it contains a space before building the `entry function …` line, mirroring the existing macro-entry pattern. (4) `CodeFunctionResolver.cpp::ResolveFunction` adds a third pass after exact + alias: iterates the class's UFunctions via `TFieldIterator<UFunction>(IncludeSuper)`, strips all ASCII spaces from both the needle and each candidate name, compares case-insensitively; one match returns it, multiple matches emit a `UE_LOG(Warning)` and return nullptr, zero falls through. Round-trip now works: `` call `Set Error`(Target: %w, Error: %s) `` compiles and decompiler emits `` entry function `Set Error`(...) ``. Tests added in `TestBpirFunctionSpaceName.cpp`: `FBpirParserBacktickFunctionCallTest` (parser unit, no graph), `FBpirParserBacktickFunctionEntryTest` (entry-block unwrap), `FCodeFunctionResolverSpaceStrippedFallbackTest` (resolver regression guard), `FCompilerIntegrationFunctionNameWithSpaceCallTest` (compiler integration, real BP graph with "Set Error" function), `FDecompilerEmitsBacktickForSpaceNameTest` (decompiler emits backtick-quoted entry for spaced-name graph).
- `#3-verification-failed-cast` `OPEN` reporter — **Verification failed.** Session re-test: `` call `Set Error`(Target: %w, Error: "test") `` returns `COMPILE_FAILED: Unresolved function: 'Set Error'`. The backtick was stripped by the parser (good — confirms change #1) but the resolver's space-stripped fallback (change #4) did not locate `Set Error` on the target's class. Target in the probe was `UCommonActivatableWidget*` (return of `PushContentToLayerForPlayer`) — NOT a `W_Error_C*`, because the prior cast (`cast<W_Error_C>(%w)`) failed with `Unresolved class for cast: W_Error_C`. So the real blocker chain here is: the cast rejects `W_Error_C`, forcing the function call to happen on the wrong target class, which then can't find `Set Error`. The resolver's fallback pass probably only searches the target's class and ancestors; since `UCommonActivatableWidget` has no `Set Error`, the fallback legitimately fails. **Two fixes needed:** (a) cast resolver must accept BP generated-class names with `_C` suffix for BP assets on `/App/` or other non-engine mounts; (b) once cast works, re-verify the resolver fallback actually finds the spaced function. Probe widget was fresh (`/Game/W_McpReplayProbe2`), so pre-existing state isn't a factor.
- `#4-cast-resolver-fix` `IN-REVIEW` developer — Cast resolver fix (the upstream blocker): `BpirCompiler.cpp::EBpirOpcode::Cast` (~line 3370) replaced its three-tier `FindFirstObjectSafe<UClass>` with prefix retries with a single `ResolveUClass(Inst.TypeArg)` call. Added a new tier-7 fallback in `Utils/ClassUtils.cpp::ResolveUClass` (~lines 200-238): when the input is a bare short name ending in `_C` and isn't found by the earlier tiers, strip the `_C`, query the asset registry across all mounts (`Filter.ClassPaths.Add(UBlueprint::StaticClass()->GetClassPathName())`) for matching `UBlueprint` assets, then load the generated class via `PackageName.AssetName_C`. This covers unloaded BP generated classes on any content mount (`/App/`, `/Game/`, plugin mounts), which `FindFirstObjectSafe` cannot. With the cast unblocked, the previously-applied space-stripped function-name fallback in `CodeFunctionResolver::ResolveFunction` will now be exercised on the correct target class and should locate `Set Error` on `W_Error_C`.
- `#5-verified-cast-and-space` `DONE` tester — User marked verified. Session re-test: `` cast</App/App/UI/W_Error.W_Error_C>(…) `` resolves cleanly (full-path form); `` call `Set Error`(Target: %c.AsWError, Error: "test") `` on the cast result compiles with 0 errors, 0 warnings. End-to-end BPIR: `GetAllWidgetsOfClass → foreach → cast<W_Error_C> → call \`Set Error\`` succeeds. Function-name-with-space resolution works once cast is upstream-resolved.
