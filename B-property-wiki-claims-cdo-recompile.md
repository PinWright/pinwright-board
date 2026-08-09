---
id: B-property-wiki-claims-cdo-recompile
title: "The property.set wiki page claims it recompiles the Blueprint when the target is a CDO, but the handler has no compile call at all, so callers skip the explicit compile step their CDO edit actually needs"
status: OPEN
severity: Low
category: bug
tags: [property-set, docs, wiki, cdo, blueprint-compile]
encounters: 1
lastSeen: 2026-08-09T04:36:42Z
---

# The `property.set` wiki page falsely claims it recompiles the Blueprint on a CDO edit

`Plugins/PinWright/docs/wiki-src/property.md:29` states that `property.set` *"Recompiles the Blueprint when the target is a CDO."* It does not. The claim is mirrored into the generated page at `Saved/PinWright/wiki/property.set.md:25`.

## What's wrong

Grepping `Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp` for `FBlueprintEditorUtils`, `FKismetEditorUtilities`, `MarkBlueprintAsModified`, `MarkBlueprintAsStructurallyModified`, and `CompileBlueprint` returns **no matches anywhere in the file**. The generic reflected path calls only `RootObject->Modify()` (`:1097`) and `RootObject->PostEditChange()` (`:1107`). `PostEditChange()` builds an empty `FPropertyChangedEvent` (`Obj.cpp:513`) that carries no `MemberProperty`, so even the engine's `PostEditChangeChainProperty` CDO→instance propagation is not reached — nothing compiles, and nothing propagates to existing instances.

## Impact

A caller reading the method page believes CDO edits are compiled and propagated, and will skip an explicit compile step it actually needs. `property.set` is an every-session method, so the wrong belief is cheap to acquire and reaches a wide audience.

## Fix

Delete the sentence from `docs/wiki-src/property.md:29` and regenerate the mirror at `Saved/PinWright/wiki/property.set.md` (regenerates at editor startup). The neighbouring sentence at `:31` already points callers at `blueprint.set_default` for the "set CDO default, compile, save, read back" workflow, which is the correct verb — that pointer stays.

The alternative — implement recompile-on-CDO-edit and keep the doc — is not recommended. Silently recompiling a Blueprint would be a surprising side effect for a property setter, and it would conflict with the `markDirty:false` transient-probe workflow (see `B-property-set-markdirty-false-still-dirties`): a compile dirties the Blueprint package unconditionally, which is exactly what that workflow is trying to avoid.

## Distinct from

`B-property-set-markdirty-false-still-dirties` — found in the same handler during the same read, but that is a behavioral contract violation (`markDirty:false` still dirties the package, and the response echoes the request param instead of observed state). This ticket is a documentation-only defect: no code behaves wrongly, the page describes behavior that was never implemented.

## Severity

`Low` — pure documentation defect: nothing is corrupted, nothing silently fails, the described feature simply does not exist. Reach argues for the top of the `Low` band rather than the bottom, since the false claim sits on an every-session method's page, but it does not warrant a bump to `Medium`: the correct verb (`blueprint.set_default`) is named two sentences later on the same page, so the misdirection is self-correcting for anyone who reads on.

## History
- `#1-found-during-markdirty-investigation` `OPEN` reporter — Found while reading `UtilityPropertyHandler.cpp` for the `markDirty` defect (`B-property-set-markdirty-false-still-dirties`), not reported by any customer. `docs/wiki-src/property.md:29` claims `property.set` "Recompiles the Blueprint when the target is a CDO"; grepping the whole of `Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp` for `FBlueprintEditorUtils`, `FKismetEditorUtilities`, `MarkBlueprintAsModified`, `MarkBlueprintAsStructurallyModified`, and `CompileBlueprint` returns zero matches — the generic path runs only `Modify()` (`:1097`) and `PostEditChange()` (`:1107`), and the empty `FPropertyChangedEvent` built at `Obj.cpp:513` carries no `MemberProperty`, so the engine's `PostEditChangeChainProperty` CDO→instance propagation is not reached either. The claim is mirrored into the generated page at `Saved/PinWright/wiki/property.set.md:25`. Recommended fix is deleting the sentence and regenerating the mirror rather than implementing the behavior: a silent recompile inside a property setter would surprise callers and would dirty the Blueprint package, conflicting with the `markDirty:false` transient-probe workflow; `blueprint.set_default`, already named at `property.md:31`, is the correct verb for compile-and-save CDO edits. Dedup: checked the board for `property-set`/`dirty`/`markdirty` near-duplicates — no existing ticket covers this doc claim.
