---
id: B-foliage-add-type-name-with-slash-kills-the-editor
title: "foliage.add_type concatenates its `name` onto a hardcoded /Game/Foliage with no validation, so a name carrying a path — the only thing a caller can try, because the verb exposes no savePath — builds `/Game/Foliage//Game/...` and CreatePackage's Fatal log kills the editor process outright, taking every unsaved package with it"
status: IN-REVIEW
severity: Critical
category: bug
tags: [foliage, add_type, create_procedural, crash, editor-process-death, unvalidated-input, package-path, double-slash, data-loss, shared-editor]
encounters: 1
lastSeen: 2026-08-30T16:00:00+05:00
---

# A string argument ends the process, and the validator it needed is already in the same file

`foliage.add_type` builds its package path by blind `Printf` and hands the result straight to
`CreatePackage`. `Handlers/Environment/FoliageHandler.cpp:1584-1589`:

```cpp
  FString PackagePath = TEXT("/Game/Foliage");
  FString AssetName = Name;
  FString FullPackagePath =
      FString::Printf(TEXT("%s/%s"), *PackagePath, *AssetName);

  UPackage *Package = CreatePackage(*FullPackagePath);
```

`Name` is never checked. A `name` beginning with `/` yields `/Game/Foliage//Game/...`, and
`CreatePackage` treats a double slash as unrecoverable —
`C:/UE_5.8/Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectGlobals.cpp:1094-1097`:

```cpp
		if( InName.ToView().Contains(TEXT("//")))
		{
			UE_LOGF(LogUObjectGlobals, Fatal, "Attempted to create a package with name containing double slashes. PackageName: %ls", PackageName);
		}
```

`Fatal` is not compiled out in any configuration. There is no error return for the handler's
`if (!Package)` at `:1590` to catch: the process is gone before that line is reached.

## Measured — it killed a live editor, and the callstack names the plugin line

Ticket-verification session, 2026-08-30, editor pid 7856 on the 13:32 build, running
`/Game/Maps/PW_VegetationTest`. One call:

```js
call({ method: "foliage.add_type", args: {
  name: "/Game/PinWrightScratch/PWScratch_RemoveVerify1332",
  meshPath: "/Game/PinWrightScratch/SM_SphereProbe",
  density: 100
}})
```

The RPC never answered — the transport reported the stream ending early
(`WinError 10054`), and the next `editor.status` returned `EDITOR_NOT_RUNNING (connection
refused)`. `Saved/Logs/EAContentExamples58.log` at `[2026.08.30-10.58.01:754]` (log clock):

```
Fatal error: [File:.../UObject/UObjectGlobals.cpp] [Line: 1096]
Attempted to create a package with name containing double slashes.
PackageName: /Game/Foliage//Game/PinWrightScratch/PWScratch_RemoveVerify1332
[Callstack] UnrealEditor-CoreUObject.dll!CreatePackage() [UObjectGlobals.cpp:1099]
[Callstack] UnrealEditor-PinWright.dll!AutoHandler_317_() [.../FoliageHandler.cpp:1589]
[Callstack] UnrealEditor-PinWright.dll!FRpcDispatcher::ProcessRequest() [RpcDispatcher.cpp:813]
```

The callstack is the whole diagnosis: engine `CreatePackage`, called directly from the
registered handler at the line above, on the RPC dispatch path. No PinWright frame between the
argument and the fatal.

## Why a caller reaches this, rather than it being an exotic input

**`foliage.add_type` is the only asset-creating verb in the namespace with no path parameter.**
Its registered params (`:1486-1503`) are `name`, `meshPath`, `density`, `minScale`, `maxScale`,
`alignToNormal`, `randomYaw`, `enableDensityScaling` — and the destination `/Game/Foliage` is a
literal at `:1584`. Every neighbouring verb takes a path:

| verb | where the asset goes |
|---|---|
| `foliage.create_procedural` | `savePath`, sanitized, default `/Game/ProceduralFoliage` |
| `foliage.add_instances` | `foliageTypePath`, a full object path |
| `foliage.get_instances` | `foliageTypePath`, a full object path |
| `foliage.remove` | `foliageTypePath`, a full object path |

So a caller who wants the type somewhere other than `/Game/Foliage` — a scratch folder, for
instance, which is what this session wanted — has exactly one lever to try, and pulling it ends
the process. The wiki page for the verb documents neither the hardcoded destination nor the
constraint on `name`.

## The validator is already in the file — twice — and `name` is the one argument it is not applied to

This is not a missing capability. The same handler validates its *other* path argument
immediately above the crash, `:1573-1580`:

```cpp
  if (!StaticMesh) {
    if (!FPackageName::IsValidLongPackageName(MeshPath)) {
      Ctx.SendError(TEXT("INVALID_ARGUMENT"),
          FString::Printf(TEXT("Invalid package path: %s"), *MeshPath));
```

And `foliage.create_procedural`, in the same file, sanitizes its `savePath` before the identical
concatenation, `:2073-2081`:

```cpp
    FString SafeSavePath = SanitizeProjectRelativePath(RequestedSavePath);
    if (SafeSavePath.IsEmpty()) {
      Ctx.SendError(TEXT("SECURITY_VIOLATION"),
          FString::Printf(TEXT("Invalid or unsafe savePath: %s"), *RequestedSavePath));
```

`SanitizeProjectRelativePath` is a shared helper with call sites across `AssetManageHandler.cpp`,
`EQSHandler.cpp`, `AssetWorkflowHandler.cpp` and `SpawnMaterialUtils.h`. Both the check and the
error code the fix needs already exist; `name` is simply not run through either.

## Second reachable site, same shape — source-derived, deliberately NOT executed

`foliage.create_procedural` sanitizes `savePath` and then does the same blind concatenation with
`name`, `:2083-2087`:

```cpp
  FString AssetName = Name + TEXT("_Spawner");
  FString FullPackagePath =
      FString::Printf(TEXT("%s/%s"), *PackagePath, *AssetName);

  UPackage *Package = CreatePackage(*FullPackagePath);
```

and reuses that same `AssetName` for each generated foliage type at `:2194-2197`. A
`name: "/Game/X/Y"` there produces `/Game/ProceduralFoliage//Game/X/Y_Spawner` and the same
Fatal. **This second site is read from source and was NOT reproduced** — reproducing it means
deliberately killing an editor a second time, which is not worth the confirmation. Whoever fixes
`:1584` must fix `:2083` and `:2194` in the same change; treat the second as high-confidence
inference, not as a measurement.

The auto-type site at `:137` (`CreatePackage(*AutoPackagePath)`, the `Auto_<Mesh>` type
`foliage.add_instances` creates from a mesh path) builds its name from the mesh *basename*, so a
caller's slashes cannot reach it. Not affected.

## What it costs, which is the reason this is Critical and not High

An editor process death is not a failed call. It takes **every unsaved package in that editor**,
and PinWright editors are shared: this one was hosting another agent's work at the time. In this
single incident the log shows the level `/Game/Maps/PW_VegetationTest` dirty and autosaving
minutes earlier, and an in-memory-only foliage type `/Game/Foliage/FT_Wave_RotProbe` that
existed nowhere on disk (`Content/Foliage/` has no such file after the crash; only the
`Saved/Autosaves/` copy the editor happened to write at `[10.54.59:604]`). Anything created
through the RPC surface and not yet saved is gone, and nothing warns the caller that the
argument they are about to pass is a process-level hazard.

The blast radius is also *not* scoped to the caller: a second agent sharing the editor loses its
session to an RPC it did not make and cannot see.

## Fix

Validate `name` before the concatenation, at all three sites, and refuse rather than crash:

```cpp
  if (Name.Contains(TEXT("/")) || Name.Contains(TEXT("\\"))) {
    Ctx.SendError(TEXT("INVALID_ARGUMENT"),
        FString::Printf(TEXT("'name' must be a bare asset name, not a path: %s. "
                             "foliage.add_type always creates under /Game/Foliage."), *Name));
    return true;
  }
```

Two decisions worth making explicitly rather than by default:

1. **Refuse, or accept a path?** Refusing is the minimum and stops the crash. But the reason the
   input was tried at all is that the destination is hardcoded, so a refusal leaves the
   ergonomic trap standing and only makes it survivable. Adding a `savePath` to
   `foliage.add_type` — the parameter every sibling verb already has, resolved through the same
   `SanitizeProjectRelativePath` `create_procedural` uses at `:2073` — removes the motivation
   for the bad input instead of punishing it. Recommended, and cheap, since the helper is
   already linked into this translation unit.
2. **Guard `CreatePackage` centrally.** 320 `CreatePackage(*...)` call sites exist under
   `Source/PinWright/Private/`, across ~30 handler files. This ticket proves only that one of
   them is reachable with an unvalidated caller string, and asserts a second by inspection; it
   does **not** claim the other 318 are unguarded — that is unaudited. A shared
   `PinWright::CreatePackageChecked(Path, OutError)` that rejects `//` and returns a null plus
   an error code would make the whole class non-fatal without auditing every site, and is the
   architectural fix. Filed as the recommendation, not as a measured need.

Regression test: a behavioural test cannot assert on a `Fatal` (it takes the test process too),
so the test has to assert the **refusal** — `foliage.add_type {name: "/Game/X/Y"}` returns
`INVALID_ARGUMENT` and creates nothing — which is exactly the post-fix contract and fails on the
current build by crashing rather than by returning red. State that in the test's comment so a
future reader does not "fix" it into a crash-expectation.

## Cross-links

- `E-foliage-add-type-auto-save-undocumented` (WONTFIX, Low) — the other thing this verb does
  that its docs do not say. Same discoverability gap, different consequence; that one is a
  redundant call, this one is process death. Its WONTFIX is about the save, not about `name`.
- `B-add-instances-auto-foliage-type-name-mismatch` — concerns the `Auto_<Mesh>` naming at
  `:137`, the one `CreatePackage` site in this file a caller's slashes cannot reach. Adjacent,
  not the same.
- `B-create-anim-blueprint-duplicate-name-crash` — the nearest sibling in class: a caller string
  reaching an engine fatal through a creation verb. Worth reading together if the central
  `CreatePackageChecked` route above is taken.

## Severity

**Critical.** Impact class: process death, and with it loss of unsaved asset and level data —
both bands the rubric puts at Critical, reached by one well-formed argument to a registered verb
on its documented happy path. **Reach modifier declined upward and downward.** No upward bump:
`foliage.add_type` is not an every-session verb. No downward "rare edge path" bump either: a
path-shaped `name` is the input the namespace's own conventions train a caller to supply, and
the verb offers no other way to choose a destination.

**What is measured and what is not.** Measured: the `add_type` crash, from the callstack in this
project's log, with the offending package name printed in full. Inferred from source and **not**
executed: the `create_procedural` sites at `:2083`/`:2194`, and the claim that a bare `name`
without slashes is unaffected (every existing call in this project passes one and none has
crashed, which is weak evidence, not a test).

## History
- `#1-name-concatenation-fatals-createpackage` `OPEN` reporter — Found while re-verifying
  unrelated foliage tickets in-editor; the call that found it ended the session it was
  verifying in. `foliage.add_type {name: "/Game/PinWrightScratch/PWScratch_RemoveVerify1332",
  meshPath: "/Game/PinWrightScratch/SM_SphereProbe"}` killed editor pid 7856 (13:32 build,
  `/Game/Maps/PW_VegetationTest`) with `Fatal: Attempted to create a package with name
  containing double slashes. PackageName: /Game/Foliage//Game/PinWrightScratch/PWScratch_RemoveVerify1332`,
  callstack `CreatePackage() [UObjectGlobals.cpp:1099]` ← `AutoHandler_317_()
  [FoliageHandler.cpp:1589]` ← `FRpcDispatcher::ProcessRequest() [RpcDispatcher.cpp:813]`.
  Every line re-derived at HEAD `1a9e5778`: unvalidated concatenation `FoliageHandler.cpp:1584-1587`,
  `CreatePackage` `:1589`, registered params `:1486-1503` (no path parameter of any kind), engine
  `Fatal` `UObjectGlobals.cpp:1094-1097`. **The file already owns both halves of the fix** —
  `FPackageName::IsValidLongPackageName` is applied to `meshPath` eight lines above the crash
  (`:1573-1580`) and `SanitizeProjectRelativePath` is applied to `create_procedural`'s `savePath`
  (`:2073-2081`) — so this is an unapplied validator, not a missing one. Second site
  `create_procedural`'s `name` (`:2083-2087`, reused `:2194-2197`) is **source-derived and
  deliberately not executed**; the `Auto_<Mesh>` site at `:137` builds from a mesh basename and is
  not reachable. Cost recorded as measured: the editor was shared with another agent, the level
  was dirty, and `/Game/Foliage/FT_Wave_RotProbe` existed only in memory and is absent from
  `Content/Foliage/` afterwards. Rated Critical on process death plus unsaved-data loss, both
  reach modifiers declined. Dedup: no board ticket covers `add_type`'s `name`, `CreatePackage`
  double-slash, or this crash; the three neighbours named under Cross-links were each read and
  are different defects. No fix attempted — nothing under `Plugins/PinWright/Source/` was edited.
- `#2-compose-through-engine-package-rules` `IN-REVIEW` developer — "Added file-local
  `PinWrightComposeFoliagePackagePath(Folder, AssetName, OutPath, OutError)` to
  `FoliageHandler.cpp` and routed **all four** of that file's `CreatePackage` sites through it, so
  neither of CreatePackage's two Fatals (`//` at `UObjectGlobals.cpp:1094-1096`, empty name at
  `:1118`) is reachable from this file. It reuses the engine's own rules rather than a character
  list — `FName::IsValidXName` + `INVALID_OBJECTNAME_CHARACTERS` on the bare name (catches `/`,
  `.`, `..`, `:`), then `FPackageName::IsValidLongPackageName(..., bIncludeReadOnlyRoots=true,
  &FText Reason)` on the composed path (catches `//`, empty/too-short, missing leading slash,
  trailing slash, `INVALID_LONGPACKAGE_CHARACTERS` incl. `\`, unmounted root) — and surfaces both
  engine `OutReason` texts verbatim in the refusal. Sites, at HEAD line numbers after the edit:
  `add_type` (`:1742`, the measured crash), `create_procedural`'s spawner (`:2343`, the ticket's
  inferred second site), the per-entry `_FT_<n>` types (`:2466`, third site — unreachable once
  the spawner name is checked, kept as a local restatement that skips the entry with a reason),
  and the `Auto_<Mesh>` helper (`:171`, not caller-reachable, guarded for uniformity). All three
  ticket line references re-derived unchanged before editing (`:1584-1589`, `:2083-2087`,
  `:2194-2197`); the engine Fatal is at `C:/UE_5.8/.../UObjectGlobals.cpp:1096` in this host's
  installed engine. Error codes are raw `TEXT()` literals matching this non-adopting file's
  existing style; both codes emitted (`INVALID_ARGUMENT`, `SECURITY_VIOLATION`) are already
  registered in `Handlers/ErrorCodes.h`, so no new code was added.
  **Took recommendation 1 and added `savePath` to `foliage.add_type`** (default `/Game/Foliage`),
  spelled and validated exactly as `create_procedural` spells it —
  `SanitizeProjectRelativePath` -> `SECURITY_VIOLATION`, effective folder echoed as `save_path` —
  because a refusal alone leaves the caller with the same hardcoded destination that motivated
  the bad input. The `name`/`savePath` checks are deliberately placed ABOVE the `meshPath`
  resolution; that ordering is load-bearing for the regression test and is commented as such in
  the handler. Recommendation 2 (a central `CreatePackageChecked`) was NOT taken — out of scope
  for one ticket and it would need the 318-site audit the ticket says is undone.
  Regression tests: new `Tests/Environment/TestFoliageAddTypeNamePathSafety.cpp`, three cases —
  `add_type.NameCarryingAPathIsRefused` (rooted path, interior slash, backslash, `..`, trailing
  slash, plus a bare-name control), `.SavePathChoosesTheDestination`, `.TraversalSavePathIsRefused`.
  **The suite can never reach the Fatal**: every refusal case pairs its bad name with a
  well-formed but absent `meshPath`, so a build with the fix reverted is refused `ASSET_NOT_FOUND`
  at the mesh load — above the concatenation — and the `INVALID_ARGUMENT` assertion goes red
  instead of the host dying. `check_test_ids.py` clean (4790 ids, no dot-prefix collisions).
  Wiki: added a `### foliage.add_type` H3 to `Docs/wiki-src/foliage.md` documenting the bare-name
  rule and `savePath`, and one line under `### foliage.create_procedural` for its own `name`.
  NOT COMPILED AND NOT RUN — the orchestrator builds and runs after all agents return.
  **Fifth instance of the same defect found OUTSIDE this ticket's scope and left unedited:**
  `landscape.create_grass_type` (`Handlers/Environment/LandscapeHandler.cpp:1670-1685`) reads an
  unvalidated `name`, `Printf`s it onto a hardcoded `/Game/Landscape` and calls `CreatePackage`
  — identical shape, identical Fatal, and it takes no `savePath` either. Not touched because it
  is a different namespace and another agent is concurrently editing that file; it wants its own
  ticket."
