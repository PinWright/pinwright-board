---
id: B-sound-class-parent-no-child-link
title: "set_class_parent / create_sound_class set ParentClass but never maintain the parent's ChildClasses array"
status: IN-REVIEW
severity: High
category: bug
tags: [audio, soundclass, hierarchy]
---

# set_class_parent / create_sound_class set ParentClass but never maintain the parent's ChildClasses array

`audio.authoring.set_class_parent` (and `audio.authoring.create_sound_class`
when `parentClass` is supplied) wire the SoundClass hierarchy by writing
**only** the child side of the relationship: `SoundClass->ParentClass =
ParentClass;`. They never add the child to the parent's `ChildClasses`
array, and on a re-parent they never remove the child from the old parent's
`ChildClasses`. The result is a half-maintained hierarchy: the child points
up at its parent, but no parent knows about any of its children.

`USoundClass` stores the parent/child relationship in **two** UPROPERTY
fields that the engine keeps in sync:
- `USoundClass::ParentClass` (child -> parent)
- `USoundClass::ChildClasses` (parent -> [children])

The engine's authoritative mutator is `USoundClass::SetParentClass(InParent)`
(`Engine/Source/Runtime/Engine/Private/SoundClass.cpp`), which on reparent
does `OldParent->ChildClasses.Remove(this)` + `Modify()`, and the editor's
`PostEditChangeProperty` for the `ParentClass` property then does
`NewParent->ChildClasses.Add(this)`. The handlers bypass both: they do a raw
field assignment, so `ChildClasses` is left empty/stale.

Impact for the exact task that triggered this (build a Master -> {SFX, Music,
Dialogue} mixing tree, with SFX reparented under an intermediate Gameplay
bus, "so we get cumulative volume/pitch control per category" and can
"confirm the parent/child tree"):
- Opening any of these classes in the SoundClass editor shows a flat list of
  disconnected roots, not the intended tree — the graph builds from
  `ChildClasses` (`Editor/AudioEditor/Private/SoundClassGraph.cpp`), which is
  empty for every parent.
- `USoundClass::GetDesc()` reports `Children: 0` for every parent.
- The readback path doesn't catch it: `audio.authoring.get_audio_info` only
  emits `parentClass` (from `ParentClass`) and never `ChildClasses`, and
  `asset.dump` only serializes overridden fields — so an attempt agent that
  verifies "all parents correct" sees green while the tree is half-broken on
  the parent side.

This is a silent success-with-incomplete-effect: every call returns
`{"message":"Sound class parent updated", existsAfter:true}` and the
child-side pointer is genuinely correct, masking the missing parent-side link.

**Workaround:** none from the MCP surface — `ChildClasses` is not writable
through any audio.authoring method, so the parent-side link cannot be
repaired after the fact. Editing the parent in the SoundClass editor by hand
and re-adding the child fixes it.

**Fix:** route both handlers through `USoundClass::SetParentClass(NewParent)`
instead of `SoundClass->ParentClass = NewParent;`, then add the child to the
new parent's list when non-null (mirroring the editor's
`PostEditChangeProperty` path: `if (NewParent && !NewParent->ChildClasses.Contains(SoundClass)) { NewParent->Modify(); NewParent->ChildClasses.Add(SoundClass); }`),
with a `RecurseCheckChild` cycle guard, and `Modify()` + save on every
touched class (old parent, new parent, child). `set_submix_parent` should be
audited for the same `ChildSubmixes` symmetry. `get_audio_info` should also
emit a `childClasses` array so the tree is verifiable through the MCP
readback (see related ergonomic gap noted in the submix ticket).

## Verbatim repro (live, replay-confirmed)
1. `audio.authoring.create_sound_class {name:"FuzzMaster", path:"/Game/FuzzAudio/Classes"}` -> ok
2. `audio.authoring.create_sound_class {name:"FuzzGameplay", path:"/Game/FuzzAudio/Classes"}` -> ok
3. `audio.authoring.create_sound_class {name:"FuzzSFX", path:"/Game/FuzzAudio/Classes"}` -> ok
4. `audio.authoring.set_class_parent {assetPath:".../FuzzGameplay", parentPath:".../FuzzMaster"}` -> `{"message":"Sound class parent updated"}`
5. `audio.authoring.set_class_parent {assetPath:".../FuzzSFX", parentPath:".../FuzzMaster"}` -> ok
6. `audio.authoring.set_class_parent {assetPath:".../FuzzSFX", parentPath:".../FuzzGameplay"}` (reparent) -> ok
7. `asset.dump {assetPath:".../FuzzMaster"}` -> `properties.json` is `{}` (no `ChildClasses`), even though FuzzGameplay.ParentClass points at FuzzMaster.
8. `asset.dump {assetPath:".../FuzzGameplay"}` -> `properties.json` has `ParentClass = .../FuzzMaster` only; no `ChildClasses`, even though FuzzSFX.ParentClass points at FuzzGameplay.

Expected: FuzzMaster.ChildClasses == [FuzzGameplay], FuzzGameplay.ChildClasses == [FuzzSFX]. Actual: both empty.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live: set_class_parent and create_sound_class write only USoundClass::ParentClass via raw field assignment (AudioAuthoringHandler.cpp lines 1196 and 1291), never `ChildClasses` on the parent, and never remove the child from the old parent on reparent. Engine path is `USoundClass::SetParentClass` + `PostEditChangeProperty`. asset.dump of every parent shows empty ChildClasses; get_audio_info doesn't surface children so the readback hides it. Half-maintained hierarchy: SoundClass editor graph and GetDesc see 0 children.
- `#2-fix-childclasses-wiring` `IN-REVIEW` developer — Fixed. Added a `SetSoundClassParentMaintainingChildren(SoundClass, NewParent, bSave)` helper in `Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp` that routes through the engine's authoritative `USoundClass::SetParentClass` (removes the child from the OLD parent's ChildClasses + sets the child pointer), then mirrors the editor's `PostEditChangeProperty(ParentClass)` path by adding the child to the NEW parent's ChildClasses when absent, guarded by `RecurseCheckChild` so a reparent that would create a loop is rejected (new `PARENT_CYCLE` error). It Modify()+saves all three touched classes (old parent, new parent, child). `set_class_parent` and the `create_sound_class` parentClass branch now both call it; a non-loadable parentPath on set_class_parent now returns `PARENT_CLASS_NOT_FOUND` instead of silently no-op'ing. Also fixed the symmetric submix raw-write defects (audit note in the Fix): both `set_submix_parent` and the `create_sound_submix` parentSubmix branch now route through the engine's two-sided `USoundSubmixWithParentBase::SetParentSubmix` (maintains ChildSubmixes) and save the touched parent submix(es); the existing `EditorAutomationRpcGateway.Audio.SubmixRouting` test gained `Parent->ChildSubmixes.Contains(Child)` assertions on create / clear / restore that regress to false if the submix raw write returns. Regression test: `EditorAutomationRpcGateway.audio.authoring.set_class_parent.MaintainsChildClasses` in `Source/EditorAutomationRpcGateway/Private/Tests/Media/TestAudioHandlers.cpp` builds Master->{Gameplay} via create_sound_class parentClass, reparents SFX Master->Gameplay via set_class_parent, then clears, asserting on each parent's `ChildClasses.Contains(...)` (both the add-to-new-parent and remove-from-old-parent sides) plus the PARENT_CYCLE rejection — every ChildClasses assertion regresses to false if reverted to the raw `ParentClass =` write. Deliberately did NOT widen `get_audio_info` to emit `childClasses` here — that readback enrichment overlaps the OPEN ticket E-audio-get-info-soundclass-mix-readback-thin, which independently owns the get_audio_info widening; coordinate there to avoid doing it twice. The regression test asserts on the ChildClasses arrays directly on the loaded objects, so it needs no readback change.
