---
id: B-fab-hard-dll-imports-vs-optional-uplugin-refs
title: "Fab 0.6.0 fails to load on consumer projects: UnrealEditor-PinWright.dll hard-imports 8 engine-plugin DLLs while .uplugin declares their plugins Optional (error 126, editor aborts startup)"
status: IN-REVIEW
severity: Critical
category: bug
tags: [packaging, fab, uplugin, plugin-references, optional, dll-imports, error-126, receipt-gate, launcher-engine, customer-report]
encounters: 1
lastSeen: 2026-07-24T07:22:00+03:00
---

# Fab customer editor won't start: hard DLL imports vs Optional .uplugin refs

Customer report (Yury, 2026-07-24): PinWright 0.6.0 installed from Fab into
launcher UE 5.7.4 (`Engine/Plugins/Marketplace/PinWrigh4c3074441499V5`),
Blueprint-only project "MOTOR". Editor startup aborts with a modal and exits.
Log (MOTOR.log) shows `UnrealEditor-PinWright.dll` failing `LoadLibrary` with
GetLastError=126 and exactly these missing imports:

```
UnrealEditor-CommonUI.dll
UnrealEditor-GeometryScriptingCore.dll
UnrealEditor-GeometryScriptingEditor.dll
UnrealEditor-SmartObjectsModule.dll
UnrealEditor-MassSpawner.dll
UnrealEditor-PCG.dll
UnrealEditor-PoseSearch.dll
UnrealEditor-Chooser.dll
```

`PinWrightRecorder` (Core/CoreUObject/Engine/Json only) loads fine; the main
module carries all risky imports.

## Root cause (three layers)

1. **Hard imports are unavoidable in the shipped binaries.**
   `PinWright.Build.cs` statically links CommonUI (L76), Chooser (L29 public),
   GeometryScriptingCore/Editor (L93), and via `TryAddConditionalModule`
   SmartObjects/Mass*/PCG/PoseSearch (L139-159). `TryAddConditionalModule`
   keys on engine *source presence*, not host-project plugin enablement, so on
   any stock engine (and on Epic's Fab build farm, which compiles against a
   vanilla engine via UAT BuildPlugin) all of these link in. The
   `__has_include` guards in AIHandler/PCG/PoseSearch sources are
   header-presence guards only; they always evaluate true. The single runtime
   guard (`EnsurePoseSearchAvailable`, PoseSearchHandler.cpp:104) runs after
   DLL load, too late to help.

2. **Optional refs are dead on launcher engines (receipt gate, UE 5.4+).**
   At 0.6.0 the owning plugins (Chooser, GeometryScripting, SmartObjects,
   MassGameplay, CommonUI, PCG, ...) were `"Optional": true` in the .uplugin.
   In shared-build-environment editor builds, `FPluginManager` reads
   `BuildPlugins` from the target receipt; a BP-only project falls back to
   `Engine/Binaries/Win64/UnrealEditor.target`, whose 212 BuildPlugins contain
   none of these. Optional refs to plugins not in that list are silently
   dropped (5.7 PluginManager.cpp:2528-2534, 2566-2573, Verbose-only log).
   Result: plugins never enabled, their Binaries dirs never registered,
   error 126. C++ consumers who rebuild get a project receipt that does list
   them, which is why the bug looks intermittent across customers.

3. **SeenPlugins poisoning breaks mixed optional/non-optional closures.**
   0.6.0 had PoseSearch non-optional while Chooser stayed optional. In the
   plugin-reference BFS, a name first seen via a receipt-gated optional ref is
   marked seen and never re-queued when a later NON-optional ref (e.g.
   PoseSearch's own hard dep on Chooser) references it
   (PluginManager.cpp:2537-2578, `bIsNewlySeen` check at 2575). So even when
   PoseSearch enables, Chooser doesn't, and `UnrealEditor-PoseSearch.dll`
   (which PE-imports UnrealEditor-Chooser.dll) fails to load. This matches the
   customer's second screenshot ("Plugin 'PoseSearch' failed to load because
   module 'PoseSearch' could not be loaded").

## Unexplained residue — RESOLVED (customer descriptor obtained 2026-07-25)

The customer supplied the installed `PinWright.uplugin` from the
Marketplace folder (Docs.zip). It matches repo commit 01b1ffe5 byte-for-
byte in shape: 0.6.0, PoseSearch non-optional, Chooser/GeometryScripting/
SmartObjects/MassGameplay/CommonUI/PCG Optional:true. So Epic's Fab
pipeline ships the descriptor verbatim, and run B (PoseSearch modal) is
fully explained by SeenPlugins poisoning with this exact file.

The run-A contradiction dissolves on timestamps: the installed .uplugin's
mtime is 06:42 on 2026-07-24, AFTER the MOTOR.log run (06:34:46-06:35:10).
The customer evidently (re)installed/updated the plugin between the two
runs — run A executed against an EARLIER installed build whose descriptor
had all refs optional (the 4fd94060 "mark all optional" shape), which
silently receipt-gates every ref and produces exactly run A's silence.
Run B then ran against the 0.6.0 descriptor above. No engine-code
contradiction remains; both failures are the two known mechanisms.

## Fix status and requirements

- Dev HEAD already contains the core fix: commit 19f2c63e "Make DLL-linked
  plugin references non-optional in .uplugin". Verified against 5.7 engine
  source that this works on launcher + BP-only: non-optional refs bypass the
  receipt gate, `MarkEnabledPlugins` registers every enabled plugin's
  Binaries dir before module loading (PluginManager.cpp:1847).
- Hard rules for the descriptor (from engine source):
  - All-or-nothing per plugin name: a name must never be Optional if it also
    appears anywhere in the non-optional transitive closure (poisoning, layer
    3). Audit the three remaining Optional refs (Interchange,
    InterchangeOpenUSD, ChaosCloth): confirm they are not DLL-linked and not
    in the closure.
  - No duplicate entries for one name in the Plugins array (later entry
    silently dropped).
  - Keep refs minimal: `{"Name", "Enabled": true, "TargetAllowList":
    ["Editor"]}`. No HasExplicitPlatforms / PlatformAllowList /
    TargetConfigurationAllowList on refs.
- Packaging guard: package-fab.ps1 ships the Plugins block verbatim and has
  no assert linking Build.cs deps to non-optional refs. Add a package-shape
  check: every module in PublicDependencyModuleNames /
  PrivateDependencyModuleNames / TryAddConditionalModule that belongs to an
  engine plugin must have a non-optional ref in the .uplugin (or live in a
  soft-loaded sub-module).
- Long-term (better for consumers): split the integrations into
  `LoadingPhase: "None"` sub-modules, hard links quarantined there, main
  module gates on `IPluginManager::Get().FindPlugin(X)->IsEnabled()` then
  `FModuleManager::LoadModulePtr`. Engine precedent: 5.8 IKRig -> IKRigUAF
  (refs `Enabled:false, Optional:true`). Avoids force-enabling ~30 plugins in
  every consumer project and silently overriding consumers' deliberate
  disables (PCG/Mass are commonly disabled for cook-time/policy reasons).
  Feasibility CONFIRMED against the codebase (no hard blocker):
  - Registration already fits: `REGISTER_RPC_HANDLER` pushes into a
    PINWRIGHT_API-exported append-only collector
    (HandlerRegistration.h/.cpp); sub-module DLL static-init lands in the
    same array; `LoadModulePtr` calls go in `UPinWrightSubsystem::Initialize`
    just before `DrainAutoRegistrations` (PinWrightSubsystem.cpp ~L113),
    before wiki generation and transport start. Wiki/namespace index derives
    from the live registry (WikiHandler.cpp L171-220), so disabled
    namespaces disappear automatically.
  - Recommended partition: 5 sub-modules (PinWrightGeometry ~85 RPCs,
    PinWrightPCG 14 + IR sidecar via existing IrSidecarRegistry seam,
    PinWrightChooser 6, PinWrightPoseSearch 3, PinWrightCommonUI 4) plus
    reflection-only rewrite of the SmartObjects/Mass block in AIHandler.cpp
    (contiguous L1065-1614, no shared statics; LevelSnapshots
    PCGRestoration pattern) to avoid two extra modules for 7 rarely-used
    RPCs.
  - Refs flip to `Enabled:false, Optional:true` (IKRig shape; uniform
    across 5.3-5.8 since 5.3 lacks the receipt gate and Enabled:true would
    force-enable there).
  - Known risks: silent test-coverage loss (host .uprojects must enable all
    7 plugins + CI assert on loaded-integration log line); stripped-source
    engines (keep Directory.Exists probe compiling sub-module empty); wiki
    topic-page overlays advertising absent namespaces (ship PLUGIN_DISABLED
    affordance in dispatcher UNKNOWN_ACTION path + WikiHandler banner in
    the same change); duplicate-name drain overwrite (no stubs left
    behind); Chooser Internal include path differs on 5.3
    (Experimental\Chooser).
  - Effort: ~2-3 days incl. version matrix + launcher BP-project boot
    verification; +1 day for the SmartObjects/Mass reflection rewrite.
    Tests move with their sub-modules (~18 typed test files).
- Rejected: Windows delay-load (`PublicDelayLoadDLLs`) is for third-party
  DLLs; MSVC can't delay-load data imports from UE modules, no engine
  precedent, crashes instead of degrading.

## Customer workaround (until fixed build ships)

Enable these plugins in the consumer project: PoseSearch, Chooser, PCG,
MassGameplay, SmartObjects, GeometryScripting, CommonUI. Project-level refs
run a clean BFS, so their own deps come in transitively; if a further modal
names another plugin, enable that one too.

## History

- `#1-filed-from-customer-report` OPEN (reporter): filed from Yury's Fab
  0.6.0 report (MOTOR.log + screenshots). Root cause researched across
  plugin repo, MOTOR.log, and UE 5.6/5.7/5.8 engine source; core descriptor
  fix already at dev HEAD (19f2c63e) but unreleased and end-to-end
  unverified; residual work: closure audit of remaining Optional refs,
  packaging assert, confirm what Fab actually shipped, long-term sub-module
  split.
- `#2-fix-unreleased-on-both-channels` OPEN (reporter): verified the fix is
  in NO released package. GitHub release repo pinwright-ue: latest release
  v0.5.0 (2026-07-15), its .uplugin still has the broken shape (Chooser/
  GeometryScripting/SmartObjects/MassGameplay/CommonUI/PCG Optional:true,
  PoseSearch non-optional, i.e. the SeenPlugins-poisoning mix). Fab carries
  0.6.0, cut from a pre-fix dev commit (fix 19f2c63e postdates the 0.6.0
  descriptor commits 01b1ffe5/59e26f68). Deploying requires cutting 0.7.0
  to both Fab and the release repo. Also appended sub-module-split
  feasibility findings to the body (confirmed feasible, no hard blocker,
  recommended 5-module + reflection partition). Additional finding: the
  fresh package at X:\src\unreal\EAContentExamples57\Plugins\PinWright\dist\
  PinWright-Fab-UE5.7.zip (built 2026-07-24 00:30 from 715f3c8d, includes
  19f2c63e) carries the FIXED non-optional descriptor but is still labeled
  Version 5 / VersionName 0.6.0 - a version-number collision with the
  broken live Fab 0.6.0; bump before submitting.
- `#3-submodule-split-implemented` IN-REVIEW (developer): sub-module split
  implemented in the dev repo working tree (uncommitted, NOT yet compiled -
  compile/test loop is the verification step). Five new LoadingPhase=None
  editor modules quarantine the hard links: PinWrightGeometry (14 handler
  files + 18 tests), PinWrightPCG (12 handlers + PCGIR + 11 tests),
  PinWrightChooser (1 + 1 test, Internal-include probe covers 5.3
  Experimental layout), PinWrightPoseSearch (1 + 2 tests), PinWrightCommonUI
  (3 + 3 tests incl. UCLASS fixture). SmartObjects/Mass RPCs in
  AIHandler.cpp rewritten reflection-only (zero includes/linkage; script
  paths verified against 5.3-5.8 engine source; USmartObjectComponent
  DefinitionAsset(5.3)/DefinitionRef(5.4+) drift handled). .uplugin: five
  module entries added; GeometryScripting/PCG/Chooser/PoseSearch/CommonUI
  refs flipped to Enabled:false+Optional:true (IKRig UAF shape);
  SmartObjects/MassGameplay refs removed. Main Build.cs dropped all eight
  problem deps. New IntegrationGates.h/.cpp: gate table + LoadModulePtr in
  UPinWrightSubsystem::Initialize before DrainAutoRegistrations; dispatcher
  unknown-action path emits ERR_PLUGIN_DISABLED for skipped prefixes
  (geometry./pcg./chooser./pose_search./ui.activatable_/
  ui.list_stack_widgets/ui.get_active_widget); WikiHandler prepends an
  unavailability banner on skipped-namespace topic pages.
  release-manifest.json gained the five Source trees + test excludes;
  PDS.uproject enables all seven plugins for dev/test coverage. Review pass
  fixed two missing PINWRIGHT_API exports (AddActorVerification,
  McpActorUtils::FindActorByName) and the CommonUI prefix gap. Remaining
  before DONE: compile+test loop on the version matrix; one scratch-host
  build with one of the five plugins disabled (verifies UBT accepts
  hard-linking modules of an Enabled:false+Optional ref, the IKRig
  precedent shape); note two main-module tests
  (TestStaticMeshSetMaterialAssignsSlot, TestBpirRoundTrip CommonUI class
  path) fail rather than skip on plugin-disabled hosts - benign on dev/CI.
- `#4-compile-and-suite-green-on-56` IN-REVIEW (developer): mcp-test-loop
  run on the UE 5.6 dev host (PDS.uproject). Cycle 1 compile produced two
  errors, both module-move artifacts, fixed inline: C4273 stale
  PINWRIGHT_API on FPCGIRDecompiler (now module-internal to PinWrightPCG)
  and C2668 ambiguous ToObjectPath in PoseSearchHandler.cpp (unity-merge
  with TestUtils.h's global inline helper in the small module; file-local
  helper renamed PoseSearchAssetObjectPath). Cycle 2: compile clean, full
  suite green - 3469/3469 tests passed, 0 failed, no crash, TEST COMPLETE
  EXIT CODE 0. Startup log confirms
  "PinWright integrations: loaded=[geometry,pcg,chooser,pose_search,ui]
  skipped=[]" and 139 tests from the moved namespaces ran, so sub-module
  registration and test coverage are intact. Still pending before DONE:
  5.3-5.8 version matrix, and the scratch-host build + boot with one of
  the five plugins disabled (the actual Fab-consumer scenario).
- `#5-version-matrix-fixpoint-green` IN-REVIEW (developer):
  mcp-version-matrix reached fixpoint - all six versions (5.3-5.8) GREEN
  at final tip a8de27b4, none blocked, every version reconfirmed at that
  tip in the sweep. Three per-version fix commits: 13eb1deb (5.3),
  bfd96388 (5.4), a8de27b4 (5.8); 5.5/5.6/5.7 passed clean. Rocket
  packaging gate passed on all six; Fab-ready zips staged per version
  under X:\src\unreal\EAContentExamples5N\Plugins\PinWright\dist\
  (verified: descriptor Version 6 / VersionName 0.7.0 with the five
  LoadingPhase=None sub-modules - the 0.6.0 version collision is
  resolved). Sole remaining verification gap: boot a host with one of the
  five plugins disabled and confirm the editor starts with the
  integration skipped (the exact customer scenario from this ticket).
- `#6-shipped-descriptor-confirmed` IN-REVIEW (reporter): customer
  supplied the installed Marketplace PinWright.uplugin (Docs.zip). It
  matches repo 0.6.0 (01b1ffe5) exactly - Fab ships descriptors verbatim.
  Its mtime (06:42) postdates the MOTOR.log run (06:34-06:35), so run A
  used an earlier all-optional installed build and run B this 0.6.0;
  both failures now fully explained, investigation closed. See the
  resolved residue section in the body.
- `#7-enabled-false-refs-are-poisonous` IN-REVIEW (developer): the demo
  trial-build CI exposed a second engine trap. The 5.8 demo smoke failed
  twice (Enabled:true and Enabled:false ref shapes) with "Plugin
  'Interchange' failed to load because module 'InterchangeImport' could
  not be loaded" on a BLANK host with PinWright engine-installed; a blank
  5.8 boot WITHOUT the plugin is clean. Root cause (engine source +
  BFS simulation over real 5.3-5.8 plugin graphs): the reference BFS
  seen-marks every listed name before any flag check, so our
  Enabled:false GeometryScripting ref blocked SkeletalMeshModelingTools'
  (default-enabled on 5.8, reachable via our MovieRenderPipeline ref ->
  ChaosClothAsset -> Dataflow) enabled GeometryScripting ref;
  GeometryScripting never enabled; DLL cascade (SMMT -> DataflowEditor ->
  HairStrandsEditor -> InterchangeImport) ended at the fatal modal.
  Removing refs trips UBT's undeclared-dependency warning (checks
  bOptional only) and fails the Rocket gate. Fix 4d382237: ALL optional
  refs now Enabled:true+Optional:true (sim-clean on all four versions,
  both walk orders); docs and memory corrected (the previously-recorded
  "IKRig Enabled:false shape" guidance was wrong for names reachable in
  default-boot closures). Demo CI 5.8 re-run pending; 5.3-5.7 demo legs
  already green.
- `#8-all-versions-validated-0.7.0` IN-REVIEW (developer): 0.7.0 now
  verified on every installed engine, both release and demo editions.
  With the Enabled:true+Optional:true descriptor a blank 5.8 host with
  the plugin engine-installed boots clean (the exact
  disabled-plugin/consumer scenario), and demo MCP smoke passes 5.3-5.8
  (gate 199->198). Two demo legs (5.4/5.5) went CI-red on a teardown
  flake only - editor child tree (EpicWebHelper) held the plugin DLL so
  the engine-install cleanup threw after 30s; fixed in demo-smoke.ps1
  bc3f2450 (kill launched PID's descendant tree scoped by
  ParentProcessId, retry removal to 90s) and the stale installs the flake
  left in C:\UE_5.4/5.5 Marketplace were removed by hand. CI artifact
  storage corrected to the mcp-version-matrix convention
  (EAContentExamples<NN>\Plugins\PinWright\dist) instead of an invented
  path. Ready for the human tester to close: cut 0.7.0 to Fab + the
  release repo.
