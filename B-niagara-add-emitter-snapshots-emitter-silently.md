---
id: B-niagara-add-emitter-snapshots-emitter-silently
title: "niagara.add_emitter copies the emitter into the system instead of referencing it, and nothing says so — later edits to the emitter asset never reach the system, which still compiles and strict-validates clean"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, add_emitter, emitter-handle, snapshot, stale, silent-noop, docs, wiki-wrong]
encounters: 4
lastSeen: 2026-09-05T20:55:00+03:00
---

# A system built with `add_emitter` freezes the emitter at add time, and every published signal says it is current

`niagara.add_emitter` takes an `emitterPath` and the wiki describes it as "Add one existing
UNiagaraEmitter asset to a UNiagaraSystem". Callers reasonably read that as a reference. It is
not: the system takes a **copy** of the emitter as it stood at the moment of the call. Edit the
emitter asset afterwards — even compile and save it, even recompile and re-save the system — and
the system keeps running the old version.

There is no signal anywhere. `add_emitter`'s response carries `emitterCount`,
`emittersInvokedBySystemGraph`, `emitterNodesRebuilt` and `dataInterfaceCheck` but nothing that
distinguishes a copy from a reference. `niagara.compile {force:true}` on the system reports
`status: "completed"`. `asset.save` reports `saved:true` with a plausible size. And
`niagara.validate {level:"strict"}` returns `valid: true` with an empty `errors` array on a
system whose emitters are stale.

## Repro (live, UE 5.8, EAContentExamples58, editor on port 27145, 2026-09-02)

1. `asset.duplicate` `/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst` ->
   `/Game/FPS/VFX/Emitters/E_Blood_Spray`, author it, compile, `asset.save` (94450 b on disk).
2. `niagara.add_emitter {systemPath:"/Game/FPS/VFX/NS_Blood", emitterPath:".../E_Blood_Spray",
   name:"Spray"}` -> `emitterCount:1`, `emittersInvokedBySystemGraph:1`, `emitterNodesRebuilt:2`.
   Compile the system, save it.
3. **Now edit the emitter asset**: `niagara.add_module {assetPath:".../E_Blood_Spray",
   modulePath:"/Niagara/Modules/Emitter/SpawnBurst_Instantaneous",
   scriptUsage:"EmitterUpdateScript"}` -> `nodeId 4D7E9BD44012CFE8F8D79C94680B1355`, plus
   `Use Spawn Probability`=true and three inputs. `niagara.compile {force:true}` ->
   `status:"completed"`. `asset.save {force:true}` -> `saved:true, sizeBytes:97479`, confirmed on
   disk.
4. Recompile and re-save the **system**: `niagara.compile {assetPath:"/Game/FPS/VFX/NS_Blood",
   force:true, wait:true}` -> `status:"completed"`; `asset.save` -> `saved:true,
   sizeBytes:885385`.
5. Read the system's stack back:
   `niagara.inspect {assetPath:"/Game/FPS/VFX/NS_Blood", includeStack:true, includeGraphs:false,
   includeCompile:false}`. The `Spray` handle reports **10 modules with exactly one
   `SpawnBurst_Instantaneous`**. The emitter asset has two. The second burst — added, compiled and
   saved in step 3 — is not in the system.

Reproduced independently on a second system/emitter pair authored by a different agent in the same
session: `NS_Impact_Flesh` / `E_ImpactFlesh_Spray`, same missing second burst, same clean
`validate`.

**Recovery is `niagara.remove_emitter` + `niagara.add_emitter`.** A recompile of the system does
not refresh the copy; only re-adding the handle does. Verified: after remove+add on both systems,
the `Spray` handle reports the second `SpawnBurst_Instantaneous`.

## Why it matters

This is a silent-staleness defect on the ordinary authoring order. Building a package of systems
means authoring emitters and wiring them up, and any iteration on an emitter after it has been
wired is lost with no error. It is worse in a multi-agent or multi-person setup, where the emitter
is finished by someone else after the system was assembled — which is exactly how it was hit here:
four of nineteen emitters were still unfinished when their systems were first wired, and two
shipped stale until a structural read caught them.

It also silently defeats the "save early as a floor" practice: an `asset.duplicate` saved
immediately as a floor, then wired into a system, then finished, leaves the system holding the
**unconfigured SimpleSpriteBurst template** under the finished emitter's handle name. That system
spawns 50-unit white sprites and passes `validate level:"strict"` with zero issues.

## The wiki says the opposite, in two places

- `niagara.add_emitter.md` — "Add one existing UNiagaraEmitter asset to a UNiagaraSystem", with a
  long Notes section on `EDITOR_OPEN`, `dataInterfaceCheck` and system-graph wiring and **no
  mention of copy-vs-reference semantics at all**.
- The project's own Niagara authoring recipe states "Never share one emitter asset between two
  systems — editing it would change both", which is the exact inverse of the measured behaviour.
  Anyone following that guidance duplicates emitters per system for a hazard that does not exist,
  while walking into the one that does.

## Fix (proposed by reporter)

Three things, in order of value:

1. **Say it in the response.** `add_emitter` should report the relationship it created — e.g.
   `emitterSource: "copy"` (or `"inherited"` if PinWright ever grows the inheritance path) — so a
   caller learns it from the call it already made.
2. **Detect staleness in `niagara.validate`.** The system's copy carries the source emitter's path;
   comparing its module/graph fingerprint against the current asset would let validate raise a
   `EMITTER_COPY_STALE` warning naming the handle and the source. This is the same shape as the
   `EMITTER_NOT_IN_SYSTEM_GRAPH` check added by `B-niagara-authored-emitter-forces-inert`, and it
   catches a defect with the same "compiles clean, validates clean, renders wrong" signature.
3. **Offer a refresh verb** — `niagara.refresh_emitter {systemPath, emitter}` that does the
   remove+add internally — or at minimum document remove+add as the refresh route on
   `niagara.add_emitter.md`.

Not source-confirmed: no read of `NiagaraHandler.cpp` was made. The diagnosis is entirely from the
RPC responses and the `niagara.inspect` stack read-back, both quoted above.

## Cross-ref

- `B-niagara-authored-emitter-forces-inert` — the other "system compiles and validates clean while
  the emitter does nothing" defect; its `EMITTER_NOT_IN_SYSTEM_GRAPH` check is the model for fix 2.
- `B-niagara-add-emitter-reshapes-handles-before-kill` — same verb, different failure.

## Fix (implemented, uncommitted — not yet compiled or test-run)

**The ticket is TRUE, and its mechanism is one step narrower than reported.** `AddEmitterHandle`
*always* duplicates the emitter into the system — there is no engine mode in which a system
references an emitter asset — but it normally keeps the copy linked to the asset as its **parent**
(`UNiagaraEmitter::CreateWithParentAndOwner` sets `VersionedParent` + `VersionedParentAtLastMerge`).
It strips that link only when the source asset declares itself non-inheritable
(`NiagaraSystem.cpp:3019-3032`, `if (InEmitter.bIsInheritable == false) ... RemoveParent()`), which
is how **every stock Niagara template and behaviour-example emitter loads** and what
`asset.duplicate` preserves — `SimpleSpriteBurst.uasset` serialises `bIsInheritable` (i.e. false)
while `SimpleExplosion.uasset` does not. So the repro's `asset.duplicate` of `SimpleSpriteBurst`
produced an asset the editor would never produce: `NiagaraEmitterFactoryNew.cpp:183` sets
`bIsInheritable = true` on any emitter its wizard creates from a template. PinWright's handler was
byte-identical to the editor's own path — the defect was that **nothing reported which of the two
relationships it had made**, and there was no readback and no non-destructive repair.

Second half of the mechanism, which applies to inherited handles too and is what the `#3` encounter
actually hit: even *with* a parent, the merge runs on load (`UpdateEmitterAfterLoad`) or on demand.
Editing the asset in a live session does not touch the already-loaded system.

**Changed files** (all under `Plugins/PinWright/`):

- `Source/PinWright/Private/Handlers/Niagara/NiagaraHandler.cpp`
  - `niagara.add_emitter` gains `inherit` (boolean, **default true**). The relationship is
    **measured** off the created copy's own parent pointer after the write, never re-derived from
    the engine's inheritability rule (spelled `bIsInheritable` on 5.4+, `TemplateSpecification` on
    5.3). Response gains `emitterSource: "inherited" | "snapshot"` and `parentEmitterPath`; both
    fact blocks (`WiringFacts` / `MutationFacts`) carry `emitterSource` too, so refusals report it.
  - `inherit: false` performs the editor's *Remove Parent Emitter* (`Modify()` + `RemoveParent()`,
    matching `FNiagaraEmitterViewModel::RemoveParentEmitter`).
  - `inherit: true` (default) with a non-inheritable source **refuses**: the handle is rolled back
    out with `RemoveEmitterHandlesById` (the graph is not rebuilt on that path, so the handle list
    is the only thing to undo) and `EMITTER_NOT_INHERITABLE` is sent, naming both remedies.
  - New verb `niagara.refresh_emitter {systemPath, emitter?, compile?, save?}` —
    `MergeChangesFromParent()` per inherited handle. Deliberately a merge, not a re-add: it keeps
    the system's own per-handle overrides, which is the non-destructive route `#3` said did not
    exist. Reports `refreshed[]` (`wasStale` read before, `synchronizedWithParent` read after,
    `graphModified`, `merged`, `errors[]`), `skipped[]` (`reason: "no_parent"`),
    `mergesApplied` / `mergesFailed`. Nothing to merge → `EMITTER_NOT_INHERITED`, not a zero-item
    success. Carries the same `EDITOR_OPEN` refusal, instance quiesce, and pre-save
    `dataInterfaceCheck` gate as the two handle verbs; a run with any failed merge is not saved.
  - Handle resolution (Guid-or-name) extracted to `ResolveNiagaraEmitterHandle` and shared with
    `remove_emitter`, so the two verbs cannot drift on `EMITTER_HANDLE_NOT_FOUND` /
    `AMBIGUOUS_EMITTER_HANDLE`.
- `Source/PinWright/Private/Handlers/Niagara/NiagaraDumpBuilder.cpp` — every
  `versionedEmitterData` object (system handles, and a standalone emitter asset) now carries
  `parent { inherited, path, version, synchronized, parentAtLastMergePath }`. `synchronized` is the
  engine's own change-id comparison, the same measurement validate and refresh use. **No aspect
  version bump:** `niagara_system.json` / `niagara_emitters.json` are no longer written as dump
  sidecars (`Docs/wiki-src/niagara.dump-files.md:15`), so this reaches only live RPC payloads.
- `Source/PinWright/Private/Handlers/Niagara/NiagaraInspectHandler.cpp` — `niagara.validate` gains
  `AddStaleParentEmitterIssues`, one `EMITTER_PARENT_STALE` **warning at every level** per handle
  whose parent has moved on, naming the handle and the parent. Warning, not error: the system is
  runnable, its content is a deliberate earlier state. A snapshot handle is deliberately silent —
  it is unlinked, not stale.
- `Source/PinWright/Private/Handlers/ErrorCodes.h` — `ERR_EMITTER_NOT_INHERITABLE`,
  `ERR_EMITTER_NOT_INHERITED`. (Call sites keep raw literals: `NiagaraHandler.cpp` is not in
  `TestErrorCodeRegistry`'s partially-converted baseline, so a single `ErrorCodes::ERR_` reference
  would flip the file to "adopting" and redden its ~15 hand-spelled codes.)
- `Docs/wiki-src/niagara.md` — new namespace-page section *A system's emitter is a child of the
  asset, not the asset*; `### niagara.add_emitter` documents `inherit` / `emitterSource` /
  `EMITTER_NOT_INHERITABLE`; new `### niagara.refresh_emitter`; `### niagara.validate` documents
  `EMITTER_PARENT_STALE`; `### niagara.inspect` documents the `parent` block;
  `### niagara.create_emitter` corrected (it produces an inheritable asset). The reporter's
  "the wiki says the opposite" is closed.
- `Docs/rpc-design.md` §1 — the generalised lesson: a verb that creates a relationship must name it;
  a parameter named like a reference must not silently produce a copy; the lossy variant is the
  named opt-in and the default refuses rather than downgrading.
- `Source/PinWright/Private/Tests/Niagara/TestNiagaraEmitterInheritance.cpp` (new, 4 tests):
  `add_emitter.InheritsFromSourceEmitter`, `add_emitter.SnapshotOptionRemovesParent`,
  `refresh_emitter.MergesParentChanges`, `refresh_emitter.RefusesWhenNothingInherits`.

**The project-side recipe named in this ticket is NOT fixed here.** "Never share one emitter asset
between two systems — editing it would change both" lives in the host project's `CLAUDE.md`, not in
the plugin, and it is now doubly wrong: sharing is the intended model, and each system takes its own
child, so editing the asset changes no system until a refresh. Whoever owns that file should correct
it.

### Reviewer verification

1. **Not compiled, not run.** A separate compile pass follows this change. Nothing below has been
   executed.
2. `add_emitter` on an emitter created by `niagara.create_emitter` (inheritable) → response carries
   `emitterSource: "inherited"` and `parentEmitterPath` equal to the emitter asset path.
3. `add_emitter` on an `asset.duplicate` of
   `/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst` → `EMITTER_NOT_INHERITABLE`, and
   `niagara.inspect` shows the system's `emitterCount` unchanged (the rollback landed). Retry with
   `inherit: false` → succeeds with `emitterSource: "snapshot"`.
4. **The ticket's own repro, end to end.** Wire an inheritable emitter into a system, then
   `niagara.add_module` a second `SpawnBurst_Instantaneous` on the *emitter asset*, compile and save
   it. `niagara.inspect {includeStack:true}` on the system still shows one burst (unchanged, by
   design), and `niagara.validate {level:"strict"}` now reports `EMITTER_PARENT_STALE` naming the
   handle — this is the signal that did not exist. Then
   `niagara.refresh_emitter {systemPath, compile:true}` → `mergesApplied: 1`,
   `refreshed[0].wasStale: true`, `refreshed[0].synchronizedWithParent: true`, and the stack
   read-back now carries **two** bursts.
5. **The non-destructive property, which is the reason refresh exists rather than remove+add.**
   Before refreshing, edit something on the system's own copy (e.g. `niagara.set_module_input` on a
   handle's spawn count). After `refresh_emitter`, that override must still be there *and* the
   parent's new module must have arrived. `remove_emitter` + `add_emitter` loses the override; this
   must not.
6. `niagara.refresh_emitter` on a system whose only handle is a snapshot → `EMITTER_NOT_INHERITED`
   with `skipped[]` naming the handle.
7. `niagara.inspect` on a Niagara System and on a standalone emitter asset: both carry
   `versionedEmitterData.parent`, and its `synchronized` agrees with what validate says.
8. Watch for `PINWRIGHT_ASSERTIONS_SKIPPED reason=niagara-parent-merge-unavailable` in the suite
   log: `refresh_emitter.MergesParentChanges` steps over its post-merge assertions if the engine
   merge manager fails on the synthetic fixture. A run carrying it has not proven the merge.

## History
- `#1-initial-repro` `OPEN` reporter — Found while assembling `/Game/FPS/VFX/NS_Blood`,
  `NS_Impact_Flesh`, `NS_Smoke_Grenade` and `NS_Explosion` from 19 emitters in the
  EAContentExamples58 checkout (map as forcing function; host `CLAUDE.md` § "What this project is
  for"), 2026-09-02, UE 5.8, live editor port 27145. Both repros above were executed end to end and
  the quoted response fields are verbatim. Two systems shipped a stale emitter copy until a
  `niagara.inspect {includeStack:true}` read caught the missing module; nothing else in the toolchain
  disagreed — `compile` said completed, `save` said saved, `validate level:"strict"` said `valid:true,
  errors:[]`, and the on-disk `.uasset` sizes grew on every re-save. Recovery by
  `remove_emitter` + `add_emitter` verified on both. Filed uncommitted per the coordinator's standing
  instruction: the board's `.git/index.lock` is stale and `board-commit.ps1` is suspended.
- `#2-third-and-fourth-stale-handle-and-a-one-line-response-fix` `OPEN` reporter — Independent third and fourth instance, different reporter, same wave and same editor session as `#1`, while building `/Game/FPS/VFX/NS_Impact_Dirt`. Confirms `#1`'s mechanism and adds a cheap fix.

  **The instances.** I added `Impact` and `Spray` to `NS_Impact_Dirt`, then reworked both emitter assets — `remove_module` on their `ScaleSpriteSize`, and `set_module_input {breakExistingLink:true}` replacing `RandomRangeFloat` / `Lerp_Float` / `RampInOut` chains with literals — then compiled and `asset.save`d each emitter (`saved:true`, sizes grew, confirmed by `ls`). I then added `Clods` and `Dust`, compiled the system (`status:"completed"`), saved it (`saved:true, sizeBytes:864094`) and ran `niagara.validate {level:"strict"}`: `valid:true`, `errors:[]`, `dataInterfaceCheck:"consistent"`, `emittersInvokedBySystemGraph == emitterCount == 4`. Everything green. The system's own stack still held the torn-out modules on two of the four handles.

  **The read that caught it, offered as the check other callers can copy.** `niagara.inspect {assetPath:"<system>", includeStack:true}`, then group every stack entry by the owner segment of its `entryKey` (`"<ownerName>:<entryId>"`, per the `entryId`-is-not-a-primary-key section of the `niagara` wiki page) and diff each group's module-name set against the emitter asset's own. Result before the fix — `Impact` and `Spray` carrying `ScaleSpriteSize`, `Lerp_Float`, `RampInOut`, and `Spray` also `RandomRangeFloat`; `Clods` and `Dust`, added *after* the rework, clean. Recovery by `remove_emitter` + `add_emitter` per handle, then recompile and save (909510 b), and the same audit re-run across all three systems in the wave: 14 emitter copies, zero occurrences of any of those four module names. The two systems whose emitters I had finished *before* any `add_emitter` (`NS_Impact_Water`, `NS_Impact_Glass`) were never stale — which is the practical rule this defect imposes today: **add emitters last, or re-add after every emitter edit.**

  **The one-line fix worth doing before the real one.** `#1` already establishes that no published field distinguishes a copy from a reference. The reason callers do not go looking is narrower than that and is fixable on its own: the parameter is named `emitterPath` and the wiki sentence is "Add one existing UNiagaraEmitter asset to a UNiagaraSystem" — both of which read as *reference*, and the response then echoes `emitterPath` back verbatim with nothing contradicting it. Naming the copy in the response (a `snapshot: true`, or an `emitterSource: "copied"` beside the existing counts) and correcting that wiki sentence would have prevented all six stale handles across this wave without touching the copy semantics at all. Both this project's `CLAUDE.md` recipe and the wiki page currently assert the opposite, so a caller who read the docs first is *more* likely to hit this, not less.

  **Why this belongs in the same class as `B-niagara-module-input-dotted-subinput-silent-noop`.** Both defects answer affirmatively on exactly the check the documentation tells the caller to trust — there, the `value` echo the `set_module_input` wiki says to confirm a literal write from; here, `validate level:"strict"` plus a matching `emittersInvokedBySystemGraph`. In both cases the only disagreeing signal was a second, undocumented read (`reset_module_input`'s `kind`, and the system's own grouped stack). Two independent instances in one afternoon suggests the review question for this namespace is not "does the verb work" but "can any published field ever report that it did not".
- `#3-first-content-visible-consequence-a-shipped-system-renders-without-the-ramp-its-emitters-carry` `OPEN` builder — **The first instance of this defect measured as a visible content regression rather than as a missing module in a stack read, and the first where the stale copy silently invalidated another board ticket's "confirmed on disk" claim.**

  Found while fixing the dense-core defect on `/Game/FPS/VFX/NS_Smoke_Grenade` (UE 5.8, EAContentExamples58, live editor port 27145, 2026-09-03). `B-niagara-set-curve-keys-unreachable-module-input-di` `#5` records that the `Lerp_Float` + `RampInOut` size-over-life route was applied to "the three `NS_Smoke_Grenade` emitters — all compiled, saved, re-added to their systems with handle parity, and confirmed on disk". On this checkout it is not in the system. Byte-grep of the packages, which needs no editor and is the cheapest audit anyone can run:

  ```
  E_Smoke_Core.uasset    ScaleSpriteSize=7  Lerp_Float=8  RampInOut=5
  E_Smoke_Smoke.uasset   ScaleSpriteSize=7  Lerp_Float=8  RampInOut=5
  E_Smoke_Wisps.uasset   ScaleSpriteSize=7  Lerp_Float=8  RampInOut=5
  NS_Smoke_Grenade.uasset  ScaleSpriteSize=0  Lerp_Float=0  RampInOut=0
  ```

  `niagara.inspect {includeStack:true}` on the system agrees: 49 stack modules across three handles, none of them `ScaleSpriteSize`. The system's mtime (03:59:51) is **later** than all three emitters' (03:57), so this is not a save-ordering slip that a later re-save would have healed — the handles were simply never re-added, and every published signal stayed green: `niagara.compile {force:true}` → `completed`, `asset.save` → `saved:true`, `niagara.validate {level:"strict"}` → `valid:true`, `errors:[]`, `dataInterfaceCheck:"consistent"`.

  **What that costs on screen, which is what makes this encounter different from `#1` and `#2`.** Those two caught stale handles as an absent module in a stack diff. Here the consequence is the effect itself: a smoke grenade whose sprites are born at their final size and never grow. Smoke that does not billow is the single clearest tell that an effect is not AAA, and the system had been through three build/review rounds with the ramp present in the emitter assets the whole time, so every reader who checked the emitter asset saw a correct ramp.

  **The second-order damage is worse than the first.** A board ticket asserts a fix landed, the emitter asset corroborates it, and the shipped system does not carry it — so the claim survives every check short of reading the system's own bytes. Any ticket that says "applied to emitter X and re-added to system Y" is unverifiable unless the reporter grepped the *system*. Recommend the audit in `#2` be stated as a byte-grep of the system package, not only as a stack diff, because the grep needs no editor and no lock and can be run by a reviewer on a different host.

  **Not fixed here, deliberately.** `remove_emitter` + `add_emitter` is the documented recovery, but on this asset it would have discarded six in-flight edits I had just made to the system's own emitter copies (spawn counts, spawn rates, sprite sizes, particle alpha, sphere radius). That is a third cost of the copy semantics worth recording: **once a system's copies have been edited in place, the recovery for staleness destroys that work**, so a caller who hits this mid-iteration has no non-destructive route. A refresh that merged, or even a verb that just *reported* the per-handle diff, would have let me choose. Reported to the stream lead instead; the system ships without size-over-life.

- `#4-the-refresh-verb-cannot-reach-any-system-built-before-the-rebuild` `IN-REVIEW` builder — **The
  rebuild's `refresh_emitter` closes this defect going forward but is structurally unreachable for
  every system already on disk, and nothing offers a migration.** Measured on
  `/Game/FPS/VFX/NS_Impact_Water` (UE 5.8, EAContentExamples58, live editor port 27145,
  2026-09-05), after pulling and rebuilding the plugin.

  `niagara.inspect {includeProperties:true}` reports `versionedEmitterData.parent` on all five
  handles — `Droplets`, `Column`, `Crown`, `Mist`, `Ring` — as `{"inherited": false}`, on both the
  `emitters[]` and `system.emitterHandles[]` projections. The system was assembled with the
  pre-rebuild `add_emitter`, which had no `inherit` parameter and always took the unlinked
  snapshot. Per `refresh_emitter`'s own contract, a system with nothing inherited is
  `EMITTER_NOT_INHERITED` — an error, not a zero-item success — so the non-destructive route `#3`
  asked for exists only for handles added after the rebuild. The route for older ones is still
  `remove_emitter` + `add_emitter`, which still destroys the system's in-place edits, which is
  still exactly what `#3` reported.

  **What is missing is a re-parent, not a refresh.** `add_emitter {inherit}` decides the link at add
  time and there is no verb that attaches a parent to an existing handle (the editor's inverse of
  *Remove Parent Emitter*). One would let a project migrate handle by handle without discarding
  work; today the only migration is the destructive re-add the ticket already documents.

  A second-order obstacle worth stating before someone attempts that migration: these emitters were
  duplicated from stock Niagara templates, and the same fix's notes record that stock templates ship
  `bIsInheritable = false` and refuse `inherit: true` with `EMITTER_NOT_INHERITABLE`. So a
  re-add-to-migrate needs `property.set bIsInheritable` on each emitter asset first. Not attempted
  here — the system's copies held the only correct version of the water velocity and shape work, and
  the brief for this slot forbade the destructive route.

  **How the reconcile was done instead, offered as the pattern for anyone else stuck on a
  pre-rebuild system.** Apply the edits *forward* onto the stale emitter asset by hand so asset and
  system agree, and verify both against the file: read the system's values with
  `niagara.inspect {includeStack:true}` grouped by `entryKey` owner, re-issue them onto the asset
  with `set_module_input` / `set_static_switch`, then byte-check the `.uasset`. Slower than a merge
  and it does not scale, but it never puts the only correct copy at risk. (The save leg of that
  pattern hits a separate defect — `B-niagara-emitter-save-guard-deadlocks-unused-emitter`.)

- `#5-verified-in-fps-build` `IN-REVIEW` VFX — **The forward path works end to end. `#4` asserted
  the rebuild "closes this defect going forward" without testing it; this is that test, and it
  passes on every property the ticket asked for.** Direct route retried once per PLAN rule 2;
  leaving `IN-REVIEW` for the tester. Live editor port 27145, UE 5.8, EAContentExamples58,
  2026-09-05, on scratch assets under `/Game/FPS/VFX/Scratch_TicketRetry/` built for this and
  deleted after. `NS_Impact_Water` and `NS_Blood` were not touched — both were being edited
  concurrently.

  **(a) The relationship is now named, and the default refuses rather than downgrading.**
  `niagara.add_emitter {systemPath:".../NS_TR_Inherit", emitterPath:".../E_TR_Parent", name:"Burst"}`
  against a raw `asset.duplicate` of `SimpleSpriteBurst` ->
  `[EMITTER_NOT_INHERITABLE] … declares itself non-inheritable … Nothing was added.` with
  `{"inheritRequested":true,"emitterSource":"snapshot","handleAdded":false,"emitterCount":0}` —
  the rollback landed, exactly as the `## Fix` predicted.

  **The error's own remedy works and is worth recording, because `#4` says no re-parent exists.**
  `property.set {objectPath:".../E_TR_Parent.E_TR_Parent", propertyName:"bIsInheritable", value:true}`
  -> `{"applied":true,"value":true}`, `asset.save {force:true}` -> `saveState:"written"`, 83698 b.
  The identical `add_emitter` then succeeded: `emitterSource:"inherited"`,
  `parentEmitterPath:"/Game/FPS/VFX/Scratch_TicketRetry/E_TR_Parent.E_TR_Parent"`,
  `emitterCount:1`, `emittersInvokedBySystemGraph:1`, `emitterNodesRebuilt:2`. So a stock-template
  duplicate can be made inheritable through the generic property writer without waiting on a
  Niagara-side verb. **This does not solve `#4`** — it fixes the *emitter asset* so future
  `add_emitter` calls inherit; it does nothing for a handle already snapshotted into a system, and
  `#4`'s ask for a re-parent verb stands unchanged.

  **(b) The stale signal that did not exist now exists.** With the handle in place, the parent was
  edited the way `#1`'s repro does — `niagara.add_module {assetPath:".../E_TR_Parent",
  modulePath:"/Niagara/Modules/Emitter/SpawnBurst_Instantaneous",
  scriptUsage:"EmitterUpdateScript"}` -> `nodeId 0617E79E494214E4C4FE089243B8D00A`, then
  `asset.save {force:true}` -> `written`, 81690 b. `niagara.validate {level:"strict"}` on the
  system then returned a `EMITTER_PARENT_STALE` **warning**: *"Emitter 'Burst' inherits from
  '/Game/FPS/VFX/Scratch_TicketRetry/E_TR_Parent.E_TR_Parent', which has changed since this system
  last merged from it … Run niagara.refresh_emitter …"*. `#1`'s central complaint — `validate
  level:"strict"` returning `valid:true, errors:[]` on a system whose emitters are stale — is
  answered; the warning names the handle, the parent, and the remedy.

  **(c) The merge propagates AND preserves the system's own override — the property the whole
  ticket turns on.** Before the parent edit, a deliberate system-side override was set on the
  handle's own copy: `niagara.set_module_input {assetPath:".../NS_TR_Inherit", emitter:"Burst",
  entryId:"Burst:68A8CD574D62C54866BE778FB68D9342", inputName:"Spawn Count", value:137}` ->
  `"value":"137.0"`. Then `niagara.refresh_emitter {systemPath:".../NS_TR_Inherit"}` ->
  `refreshedEmitters:1, mergesApplied:1, mergesFailed:0`. After `niagara.compile {force:true,
  wait:true}` -> `completed`, `niagara.inspect {includeStack:true}` shows the `Burst` handle
  carrying **both** bursts — `SpawnBurst_Instantaneous` at EmitterUpdate idx 1 and the merged-in
  `SpawnBurst_Instantaneous001` at idx 2 — **and** `Spawn Count` still
  `{"valueMode":"local","value":"137.0"}`. `remove_emitter` + `add_emitter` would have discarded
  that 137. This is precisely the non-destructive route `#3` said did not exist.
  `versionedEmitterData.parent` reads `{"inherited":true, "path":".../E_TR_Parent.E_TR_Parent",
  "synchronized":true, "parentAtLastMergePath":"…:Burst.NiagaraEmitter_1"}`.

  **On disk, per the host `CLAUDE.md` rule.** `asset.save {force:true}` ->
  `saveState:"written"`, `sizeBytes:284522`;
  `Content/FPS/VFX/Scratch_TicketRetry/NS_TR_Inherit.uasset` mtime `2026-09-05 21:01:41 +0300`,
  284522 b. `grep -a` on that file finds `SpawnBurst_Instantaneous001` (the merged module),
  `137.0` (the surviving override) and `Scratch_TicketRetry/E_TR_Parent` (the parent link). The
  byte-grep-the-*system* audit `#3` recommended is the one used here, and it passes.

  **One ergonomic wrinkle the fix should own.** `refresh_emitter` with default `compile:false`
  **returns an error**, not a success: `[NIAGARA_DATA_INTERFACE_MISMATCH] … (compiled 0, resolved
  1) … the asset was not saved`, while its own payload reports `mergesApplied:1` — i.e. the merge
  landed and the pre-save gate then refused, because a merge changes the graph and compiles
  nothing. The wiki page does say to pass `compile:true`, so this is documented, but the shape is
  wrong: the default invocation of a verb whose default is `compile:false` cannot be an error. A
  caller who stops at the error will not realise the merge is already in memory and will re-run
  it. Recommend `refresh_emitter` either default `compile` to true, or treat "mismatch caused by
  my own uncompiled merge" as a `mergedNotCompiled` success field rather than a refusal.

  **Not verified: nothing visual.** Capture is owned by the VFX lead on this stream, so there is
  no frame showing the merged module running. The evidence is structural and on-disk only.
