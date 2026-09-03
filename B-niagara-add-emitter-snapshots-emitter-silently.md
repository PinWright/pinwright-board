---
id: B-niagara-add-emitter-snapshots-emitter-silently
title: "niagara.add_emitter copies the emitter into the system instead of referencing it, and nothing says so — later edits to the emitter asset never reach the system, which still compiles and strict-validates clean"
status: OPEN
severity: High
category: bug
tags: [niagara, add_emitter, emitter-handle, snapshot, stale, silent-noop, docs, wiki-wrong]
encounters: 2
lastSeen: 2026-09-02T23:10:00+05:00
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

## Fix

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
