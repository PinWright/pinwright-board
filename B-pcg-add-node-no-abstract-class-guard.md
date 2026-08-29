---
id: B-pcg-add-node-no-abstract-class-guard
title: "pcg.add_node instantiates any caller-named abstract UPCGSettings subclass — its only gate tests parentage, never CLASS_Abstract — so the editor ensures, the node is created anyway, the handler answers success, and the engine's own message says the settings object will be nulled out on save"
status: OPEN
severity: Medium
category: bug
tags: [pcg, add-node, abstract-class, newobject, class-resolution, silent-false-success, null-on-save, ensure, source-only, not-executed]
---

# `pcg.add_node` resolves an arbitrary class path and instantiates it with no abstract-class check

> **SOURCE-ONLY. NOTHING IN THIS TICKET WAS EXECUTED.** No `pcg.add_node` call was made, no
> editor was driven, no log was captured. Every claim below is read out of C++ — plugin source
> and `C:/UE_5.8/Engine/...` — and every `file:line` was re-derived by hand for this ticket.
> **Everything downstream of the `NewObject` call is inference from engine source, not
> observation.** See [Why it was not executed](#why-it-was-not-executed-and-what-that-cost)
> — the stated reason for not running it turned out to rest on a premise this ticket disproves,
> and the decision is recorded with that cost rather than hidden.

## The gate tests parentage and nothing else

`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphAuthoring.cpp` takes a caller-supplied class
path and resolves it with no allowlist:

```
:40   RPC_PARAM_REQ("nodeClass", "string", "UPCGSettings subclass path, e.g. /Script/PCG.PCGCreatePointsSettings.")
:62   UClass* SettingsClass = FindObject<UClass>(nullptr, *NodeClassPath);
:65       SettingsClass = LoadClass<UPCGSettings>(nullptr, *NodeClassPath);
:74   if (!SettingsClass || !SettingsClass->IsChildOf(UPCGSettings::StaticClass()))
:82   UPCGNode* Node = Graph->AddNodeOfType(TSubclassOf<UPCGSettings>(SettingsClass), DefaultSettings);
```

`:74` is the **only** gate between the wire and `NewObject`. It asks one question — is this class
a `UPCGSettings` — and `IsChildOf` answers yes for `UPCGSettings` itself. `CLASS_Abstract` is
never consulted. On success the handler echoes the caller's own string back (`:95`,
`Result->SetStringField(TEXT("nodeClass"), NodeClassPath)`) and dirties the package (`:91`).

The engine end of that call is a bare construction with no guard of its own:

```cpp
UPCGNode* UPCGGraph::AddNodeOfType(TSubclassOf<class UPCGSettings> InSettingsClass, UPCGSettings*& OutDefaultNodeSettings)
{
    UPCGSettings* Settings = NewObject<UPCGSettings>(GetTransientPackage(), InSettingsClass, NAME_None, RF_Transactional);
```

`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Private/PCGGraph.cpp:1292` and `:1294`. The only
subsequent test is `if (!Settings) return nullptr;` (`:1296-1299`).

## What the engine actually does — and it is not the crash the sibling verb's comment claims

`C:/UE_5.8/Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectGlobals.cpp`,
`StaticAllocateObject`, has **two** abstract-class paths and they behave differently:

```
:3466   #if WITH_EDITOR
:3467       if (GIsEditor)
:3469           if (StaticAllocateObjectErrorTests(InClass,InOuter,InName,InFlags)) { return NULL; }
:3474       else
:3475   #endif // WITH_EDITOR
:3477           // In the editor these are handled inside StaticAllocateObjectErrorTests and they may be temporary warnings
:3478           checkf(!FScopedAllowAbstractClassAllocation::IsDisallowedAbstractClass(InClass, InFlags), ...);
```

The `checkf` at `:3478` is in the **`else`** branch (`:3474-3478`) — packaged / non-editor builds only.
PinWright runs inside the editor, so the live path is `StaticAllocateObjectErrorTests`
(defined `:3281`, itself inside `#if WITH_EDITOR` at `:3280-3326`), whose abstract branch is:

```
:3290   if (FScopedAllowAbstractClassAllocation::IsDisallowedAbstractClass(InClass, InFlags))
:3292       const FString ErrorMsg = FString::Printf(TEXT("Class which was marked abstract was trying to be loaded in Outer %s.  It will be nulled out on save. %s %s"), ...);
:3294       UE_LOGF(LogUObjectGlobals, Warning, "%ls", *ErrorMsg);
:3295       ensureMsgf(false, TEXT("%s"), *ErrorMsg);
:3296   }
```

**There is no early return in that branch.** `StaticAllocateObjectErrorTests` falls through to
`return false` at `:3324`, so allocation proceeds and `NewObject` hands back a live object.
`UPCGSettings` declares no pure virtual members (`PCGSettings.h` has no `= 0;` on a function),
so the generated class constructor runs and the object is real.

Inferred consequence, not observed: `Settings` is non-null, `AddNodeOfType` proceeds to
`AddNode(Settings)` and `Settings->Rename(nullptr, Node, ...)` (`PCGGraph.cpp:1301-1305`), the
handler returns `success` with a `nodeId`, and the engine's own warning text states what happens
next — **"It will be nulled out on save."** The caller is handed a node id for a node whose
settings object does not survive the save of a graph the handler already marked dirty.

That is precisely the shape `B-tests-order-dependent-ensure-masking` documented for the animation
notify handlers: ensure + nulled-on-save + `success: true`.

## What is reachable

`IsChildOf` is inclusive of self, so `/Script/PCG.PCGSettings` passes `:74`. Sweeping
`C:/UE_5.8/Engine/Plugins/PCG/Source` for `UCLASS(...Abstract...)` immediately above a
`: public UPCGSettings...` declaration gives the reachable abstract set:

All paths below are under `C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Public/`; the line cited is
the `UCLASS(... Abstract ...)` line, with the class declaration on the line after it.

| Class | `UCLASS` line |
|---|---|
| `UPCGSettings` | `PCGSettings.h:266` — `UCLASS(MinimalAPI, Abstract, BlueprintType, ClassGroup = (Procedural))` |
| `UPCGBaseSubgraphSettings` | `PCGSubgraph.h:22` |
| `UPCGSettingsWithDynamicInputs` | `PCGSettingsWithDynamicInputs.h:14` |
| `UPCGControlFlowSettings` | `Elements/ControlFlow/PCGControlFlow.h:11` |
| `UPCGPlatformSwitchSettingsBase` | `Elements/ControlFlow/PCGPlatformSwitchBase.h:10` — `: public UPCGControlFlowSettings`, one hop further down |
| `UPCGMetadataSettingsBase` | `Elements/Metadata/PCGMetadataOpElementBase.h:88` |
| `UPCGSubdivisionBaseSettings` | `Elements/Grammar/PCGSubdivisionBase.h:77` |
| `UPCGExternalDataSettings` | `Elements/IO/PCGExternalData.h:17` |
| `UPCGDataAttributesAndTagsSettingsBase` | `Elements/Metadata/PCGDataAttributesAndTags.h:9` |
| `UPCGFilterDataBaseSettings` | `Elements/PCGFilterDataBase.h:11` |

Ten, against 173 direct `: public UPCGSettings` declarations in the same plugin — so roughly one
reachable class in eighteen is a trap. Most of them are named `*Base`, which is a real (if
accidental) deterrent. **`UPCGSettings` itself is not**, and it is the one the verb's own
`CLASS_NOT_FOUND` message invites: *"Could not resolve UPCGSettings subclass: %s"*
(`PCGGraphAuthoring.cpp:76-77`).

`UPCGSettingsInterface` (`PCGSettings.h:178`, `UCLASS(MinimalAPI, Abstract)`) is **not** reachable
— it is `UPCGSettings`'s parent, so `IsChildOf(UPCGSettings::StaticClass())` rejects it at `:74`.
It is listed here only because a reader chasing the two abstract `UCLASS` lines in that header
will otherwise assume both get through.

## The sibling verb already has the guard, and it is one line

`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphCreate.cpp:59-63`:

```cpp
// CLASS_Abstract is checked here because NewObject's own guard is a checkf
// (UObjectGlobals.cpp StaticConstructObject_Internal), i.e. a crash rather than a null.
if (!GraphClass
    || !GraphClass->IsChildOf(UPCGGraph::StaticClass())
    || GraphClass->HasAnyClassFlags(CLASS_Abstract))
```

`pcg.create_graph` and `pcg.add_node` sit in the same module, resolve a class the same way, and
differ by exactly `|| Class->HasAnyClassFlags(CLASS_Abstract)`. The fix is that clause plus the
matching "concrete" wording in the error string.

**The comment's stated reason is wrong and should be corrected in the same change.** In the
editor the guard is `ensureMsgf` + nulled-on-save, per the `UObjectGlobals.cpp:3290-3296` reading
above; the `checkf` at `:3478` only applies outside the editor. The guard is right; its
justification is not, and a wrong justification propagates — it is the reason this ticket's own
investigation was deferred (below).

## Why it was not executed, and what that cost

The decision was: do not run `pcg.add_node` with an abstract class, because the failure was
believed to be a `checkf` — an immediate editor abort — and **five agents had unsaved work in
that editor**. Weighed that way it was the right call: a shared editor is not a place to test a
suspected `appError`.

The cost is that the belief was never checked against the engine before it drove the decision,
and re-deriving `UObjectGlobals.cpp` for this ticket shows it is false for the editor process.
The abort risk was overstated; the experiment was probably safe. Two things follow, and both are
stated rather than smoothed over:

1. This ticket's severity rests on inference, not observation. Nobody has seen the ensure fire
   from this call site, seen the response, or opened the saved graph.
2. The `checkf` claim did not originate here. It is written into
   `PCGGraphCreate.cpp:59-60` and restated in `F-pcg-create-graph-class-parameter` history `#2`
   ("an editor crash rather than a null return"). It propagated from a comment into a ticket into
   a scheduling decision without anyone reading the `#if WITH_EDITOR` around it.

**What an executor should do:** call `pcg.add_node` with
`nodeClass: "/Script/PCG.PCGSettings"` on a scratch graph in an editor holding no other agent's
work, then check three things — the response (expected `success` with a `nodeId`), the log (a
`LogUObjectGlobals: Warning` naming the class plus a one-shot ensure callstack through
`AddNodeOfType`), and the graph asset after a save (expected: a node with a null settings
object). That converts every inference above into an observation and settles the severity band.

## Same shape as

`B-tests-order-dependent-ensure-masking` (IN-REVIEW, Medium) is the closest member and the same
mechanism at a different site: `animation.authoring.add_notify` / `add_notify_state` /
`add_montage_notify` fell back to the abstract `UAnimNotify` / `UAnimNotifyState` base and
`NewObject`'d it; the notify is nulled out on save while the handler returns `success: true`.
That ticket's stated fix is the generalisation this one wants — *"never `NewObject` an abstract
class"* — and it reached it independently, from a test-order symptom, without touching PCG. Its
trigger is stronger than this one's (the **default** `notifyClass` value hit the abstract path,
where `pcg.add_node` needs the caller to name one) and it is rated Medium; this ticket is
deliberately not rated above it.

`F-source-effect-preset-authoring` (IN-REVIEW, Medium) is the positive precedent in the same
tree: `audio.authoring.create_source_effect_preset` resolves a class by reflection and rejects
"a non-`USoundEffectSourcePreset`/abstract class with `INVALID_EFFECT_CLASS`" **before**
`NewObject` (history `#2`). Same problem, already solved, in a namespace that had no more reason
to solve it than PCG did.

`B-niagara-create-node-unfinalized-graph-node-creator-fatal` (DONE, Critical) is the severity
anchor at the top of the band: a rejected payload taking the editor down through an engine
assert. It is cited here to mark the distance rather than the likeness — that one's `appError`
was **observed**, with the assertion text and the ~10s delay between the clean typed error and
process death. This ticket has no such observation, and on the engine reading it should not
produce one in-editor at all. That is the main argument against Critical here.

`F-search-api-native-uclasses` (DONE) is the loose end worth one line: it shipped
`system.inspect.search_classes` with `includeAbstract?: bool, // default false — exclude
CLASS_Abstract`. The surface that **lists** classes filters abstract ones by default; the surface
that **instantiates** them does not filter at all. A caller who turns that flag on to see the
full hierarchy is handed the exact set of strings `pcg.add_node` will accept and mishandle.

## Severity

**Medium.** Impact class is **High** — silent false-success on the rubric's own terms: the
handler answers `success` with a `nodeId`, dirties the package, and the engine's message says the
settings object will be nulled on save, so the caller trusts a result that is a lie and builds a
graph on it. Reach modifier applied **downward one band**: `pcg.add_node` is not an
every-session verb, and unlike the notify family the abstract path is not the default — the
caller must name one of ~10 abstract classes out of ~174 that pass the `:74` gate. High × rare
edge path = Medium, which also lands it level with `B-tests-order-dependent-ensure-masking`, the
identical mechanism with a *stronger* trigger.

**Critical was considered and declined, with the reasoning stated so it can be overturned.** The
Critical band is "editor crash, or a write that corrupts or loses asset data". The crash half
does not hold: the `checkf` cited for it lives in `UObjectGlobals.cpp`'s non-editor branch
(`:3474-3478`), and the editor takes the `ensureMsgf` path. The data half is arguable — a saved
graph carrying a null-settings node is corrupted asset data — but it is **inference from the
engine's warning string, never observed**, and the loss is confined to the node the call just
created rather than reaching pre-existing content. Filing Critical on an unexecuted inference
would put this ahead of every observed Critical in the picker on the strength of a sentence in
engine source. If the executor's check above shows the saved graph is broken beyond the new node,
re-rate.

The counter-argument for a bump **up** rather than down, recorded because it is not weak: the
caller cannot be blamed for the class name here. `:76-77` tells them "Could not resolve UPCGSettings
subclass", which names `UPCGSettings` — the one abstract class in the reachable set that is not
signposted by a `Base` suffix — and `search_classes` will hand them the rest on request. If the
executor observes a `success` response for `/Script/PCG.PCGSettings`, the "rare edge path"
modifier is much harder to defend and High is the right band.

## Fix

Add `|| SettingsClass->HasAnyClassFlags(CLASS_Abstract)` to the `:74` condition and reword `:76-77`
to say **concrete** `UPCGSettings` subclass, matching `PCGGraphCreate.cpp:65-67`. While there,
correct the `PCGGraphCreate.cpp:59-60` comment: in the editor the engine guard is an
`ensureMsgf` + nulled-on-save, not a `checkf`; the `checkf` is the non-editor branch.

**Do not fix only this site.** There is no umbrella ticket auditing class resolution across the
verb surface and, per board policy, there should not be one — but the pattern has now been
rediscovered three times independently and none of the three found the others.
`F-pcg-create-graph-class-parameter` found it while adding `graphClass` and scoped itself to that
parameter; its `#1` dedup note enumerates every `F-pcg-*` / `B-pcg-*` / `E-pcg-*` file on the
board and still did not raise `add_node`'s missing guard, in a file it was reading and quoting at
the time. `B-tests-order-dependent-ensure-masking` found it from a test-order symptom in
animation. `F-source-effect-preset-authoring` handled it correctly in audio without noting it as
a class of defect. A fixer landing this change should grep the whole `Source/` tree for
`NewObject` reached from a caller-supplied `UClass*` and check each for `CLASS_Abstract`, rather
than closing one site and leaving the next rediscovery to another symptom.

**Test:** a regression test asserting `pcg.add_node` with `nodeClass: /Script/PCG.PCGSettings`
returns `CLASS_NOT_FOUND` and adds no node — resolved by reflection so it does not hard-code a
class list. Assert against the graph's node count, not against the response alone; the response
is the half that already lies.

## History
- `#1-source-only-no-abstract-guard` `OPEN` reporter — **Source-read only; nothing executed, deliberately.** `pcg.add_node` resolves a caller-supplied class path (`PCGGraphAuthoring.cpp:62`, `:65`) and gates it on parentage alone (`:74`, `IsChildOf(UPCGSettings::StaticClass())`, inclusive of self), then hands it to `Graph->AddNodeOfType` (`:82`) → `NewObject<UPCGSettings>(GetTransientPackage(), InSettingsClass, NAME_None, RF_Transactional)` (`PCGGraph.cpp:1292`, `:1294`) with no `CLASS_Abstract` check anywhere. Ten abstract classes pass `:74` (table in body; `UPCGSettings` itself at `PCGSettings.h:266` is one, and is the class the verb's own `CLASS_NOT_FOUND` string at `:76-77` names); `UPCGSettingsInterface` (`PCGSettings.h:178`) does **not** — it is the parent, not a child. The sibling `pcg.create_graph` already carries the one-line guard at `PCGGraphCreate.cpp:59-63`. **Not executed, and the reason it was not is itself a finding:** the plan was that this is a `checkf` — an immediate editor abort — and five agents had unsaved work in the shared editor at the time, so running it was refused. Re-deriving the engine for this ticket disproves that premise: the `checkf` (`UObjectGlobals.cpp:3478`) is in the **non-editor** `else` branch, while the editor takes `StaticAllocateObjectErrorTests` (`:3281`), whose abstract branch logs a Warning and `ensureMsgf(false, ...)` at `:3290-3295` and **falls through without returning** (`return false`, `:3324`) — so allocation succeeds and the object is real (`UPCGSettings` has no pure virtuals). The engine's message states the consequence: *"It will be nulled out on save."* Everything downstream of `NewObject` is therefore **inference from engine source, not observation** — no call was made, no response seen, no log captured, no graph opened. The same false `checkf` claim is written into `PCGGraphCreate.cpp:59-60` and restated in `F-pcg-create-graph-class-parameter` history `#2`; it should be corrected in whichever change lands first. Severity Medium: High impact class (silent false-success — `success` + `nodeId` + `MarkPackageDirty` over an object the engine says will be nulled), reach modifier applied downward one band because `pcg.add_node` is a niche verb and the abstract path is not its default (~10 abstract of ~174 classes passing `:74`), landing level with `B-tests-order-dependent-ensure-masking` (Medium, identical mechanism, *stronger* trigger — its abstract fallback was the default parameter value). Critical declined and the reasoning recorded: the crash half of that band does not hold in-editor, and the data-loss half is an unobserved inference confined to the node just created. Dedup: no board ticket names `pcg.add_node`'s class guard — `F-pcg-create-graph-class-parameter` (IN-REVIEW, High) scoped itself to `create_graph`'s new `graphClass` parameter and its `#1` dedup note enumerated every `*-pcg-*` file without raising it, while quoting `PCGGraphAuthoring.cpp:62/:65/:67/:75` (stale line numbers — the guard is now `:74` and the call `:82`); `B-tests-order-dependent-ensure-masking` (IN-REVIEW, Medium) and `F-source-effect-preset-authoring` (IN-REVIEW, Medium) are the same mechanism in animation and audio; `B-niagara-create-node-unfinalized-graph-node-creator-fatal` (DONE, Critical) is a different engine assert cited only as the severity anchor; `F-search-api-native-uclasses` (DONE) ships `includeAbstract` defaulting false, so the listing surface filters what the instantiating surface accepts. Repro for whoever runs it, in an editor holding no other agent's work: `pcg.add_node` with `nodeClass: "/Script/PCG.PCGSettings"` on a scratch graph; check response, `LogUObjectGlobals` warning + ensure callstack, and the graph asset after a save.
