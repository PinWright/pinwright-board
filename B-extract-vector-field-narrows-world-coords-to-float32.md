---
id: B-extract-vector-field-narrows-world-coords-to-float32
title: "ExtractVectorField casts every parsed JSON coordinate to float before storing it in a double FVector, so the RPC boundary silently caps a 44-million-km world at float32 precision — the same file's ParseVectorFromJson does not narrow, and Ctx.GetVector routes most location/bounds/region params through the one that does"
status: IN-REVIEW
severity: High
category: bug
tags: [json, jsonutils, extract-vector-field, precision, float32, lwc, world-partition, silent-wrong-data, rpc-boundary, internal-inconsistency]
---

# The parse produces a double, the destination is a double, and a cast in the middle throws half the bits away

`ExtractVectorField` is the shared helper every location-, bounds- and region-taking verb reaches
for. `Source/PinWright/Private/Utils/JsonUtils.cpp`:

```
:16    void ReadVectorFieldImpl(const TSharedPtr<FJsonObject>& Obj, const TCHAR* FieldName, FVector& Out, const FVector& Default)
:29        double X = Default.X, Y = Default.Y, Z = Default.Z;
:30-35     ... TryGetNumberField(TEXT("x"), X) ...          // fills the doubles
:36        Out = FVector((float)X, (float)Y, (float)Z);      // object form
:42        Out = FVector((float)(*Arr)[0]->AsNumber(), (float)(*Arr)[1]->AsNumber(),
:43                      (float)(*Arr)[2]->AsNumber());      // array form
```

```
:140   FVector ExtractVectorField(const TSharedPtr<FJsonObject>& Source, const TCHAR* FieldName, const FVector& DefaultValue)
:145       ReadVectorFieldImpl(Source, FieldName, Parsed, DefaultValue);
```

`TryGetNumberField` writes a `double` into a `double` local (`:29`). `AsNumber()` returns a
`double`. `FVector` is `double` throughout UE5. The cast at `:36` and `:42-43` sits between a
double source and a double destination and does nothing except discard 29 bits of mantissa.
Nothing on either side asks for it.

## The same file already does it the other way

`:158`, forty-odd lines below, is the same operation written correctly:

```
:158   FVector ParseVectorFromJson(const TSharedPtr<FJsonObject>& JsonObj, const FString& FieldName, const FVector& Default)
:168       double X = 0.0, Y = 0.0, Z = 0.0;
:169-171   (*VecObj)->TryGetNumberField(TEXT("x"), X); ...
:172       return FVector(X, Y, Z);
```

Same file, same parse, same destination type, no cast. Two helpers in one translation unit
disagree about whether a JSON coordinate is a float or a double. That internal disagreement is
the cheapest available proof this is an oversight and not a decision — no design note, no
comment, and no consumer could want both behaviours at once.

## The plugin already knows the exact number this cast imposes

`Source/PinWright/Private/Handlers/Level/LevelAuditUtils.cpp:60-66` reasons about float
precision in world coordinates carefully and by name:

> *"HALF_WORLD_MAX is not a usable lint bound on 5.8: EngineDefines.h:41-56 defines WORLD_MAX as
> UE_LARGE_WORLD_MAX (8.796e12 cm) unless UE_USE_UE4_WORLD_MAX is forced, so HALF_WORLD_MAX is
> ~4.4e12 cm - about 44 million km. An actor a thousand kilometres off the map passes it.
> FThresholds::WorldBoundsCm therefore defaults to 1,048,576 cm, which is simultaneously
> UE_OLD_HALF_WORLD_MAX (EngineDefines.h:38) and UE_FLOAT_HUGE_DISTANCE (EngineDefines.h:59,
> documented as the largest distance a float still holds to 1/16 cm)."*

`LevelAuditUtils.h:175` — `double WorldBoundsCm = 1048576.0;`. The engine's own definition,
`C:/UE_5.8/Engine/Source/Runtime/Engine/Public/EngineDefines.h:59`:

```cpp
#define UE_FLOAT_HUGE_DISTANCE  1048576.0  /* Maximum distance representable by a float whilst maintaining precision of at least 0.0625 units (1/16th of a cm) - Precision issues may occur for positions/distances represented by float types that exceed this value */
```

So one file in this plugin picks 10.49 km as a *lint threshold* because that is where a float
stops holding a position to 1/16 cm — and another file imposes that same limit as a *hard cap*
on every coordinate arriving over the wire, with no threshold, no warning and no note. The
default world (`EngineDefines.h:41`, `:45-54`; `UE_USE_UE4_WORLD_MAX` is not overridden anywhere
in this checkout) reaches 4,398,046,511,104 cm from the origin — 43,980,465 km. That is
**4,194,304×** further out than the precision horizon this cast installs.

## Measured

Computed against IEEE-754 binary32, not estimated:

| From origin | What happens |
|---|---|
| < 83.9 km (`2^23` cm) | Every integer centimetre round-trips exactly. Most maps never see this bug. |
| 10.49 km (`UE_FLOAT_HUGE_DISTANCE`) | Float positions stop holding 1/16 cm — the engine's own documented line, and the plugin's own lint threshold. |
| 83.9 km | Integer centimetre coordinates start quantising; ULP reaches 1 cm. |
| 10,000.03 km | A region `max.x` of `1000003000` comes back as **`1000003008`** — 8 cm of silent error. |
| 343,597.4 km (`2^35` cm) | ULP reaches 3000 cm: **a valid 3000 cm span collapses to exactly zero.** Still 128× inside `WORLD_MAX/2`. |
| 10,000,000 km (`1e12` cm) | `min` and `max` 3000 cm apart both land on `999999995904`. Span 0; each coordinate also off by 41 m. |

The second row of that lower block is the one that turns a precision bug into a wrong answer.

## The refusal that masks it

`spatial.find_clear_placement` (`Source/PinWright/Private/Handlers/Spatial/MeasureHandler.cpp`,
registered `:638`) parses the region through this helper and immediately tests it for degeneracy:

```
:804        RegionMin = ExtractVectorField(RegionObj, TEXT("min"), FVector::ZeroVector);
:805        RegionMax = ExtractVectorField(RegionObj, TEXT("max"), FVector::ZeroVector);
:806        if (RegionMax.X <= RegionMin.X || RegionMax.Y <= RegionMin.Y || RegionMax.Z <= RegionMin.Z)
:808            Ctx.SendError(TEXT("INVALID_PARAMS"),
:809                TEXT("region.max must exceed region.min on every axis."));
```

Past `2^35` cm the caller's `min` and `max` are truncated onto the same float, `:806` is true,
and the caller is told **"region.max must exceed region.min on every axis"** — a statement about
their payload that is false. Their region was fine; it was flattened one line earlier, by the
helper, without a word. The handler's own dedicated degenerate-region message is the diagnostic
that hides the real cause.

`spatial.scatter_layout` has the identical shape:
`ScatterLayoutHandler.cpp:116-117` (the two `ExtractVectorField` calls), `:118` (the `<=` test),
`:120` (`"max must exceed min on X and Y"`), surfaced at `:235-236` as `region: %s.`.

## Reach — counted, not estimated

`ExtractVectorField` appears **50 times** in the plugin tree. Removing the definition
(`JsonUtils.cpp:140`) leaves **49**; removing the declaration (`JsonUtils.h:36`) leaves
**48 references outside the helper's own `.cpp`/`.h` pair**, which confirms the reported figure.
Broken down:

- **21** in `Source/PinWright/Private/Tests/Core/TestJsonUtils.cpp`.
- **27** in handler `.cpp` files, of which 3 are comment mentions (`AudioHandler.cpp:632`,
  `SCSHandler.cpp:261`, `:263`) → **24 real production call sites across 13 files**:
  `Handlers/HandlerContext.cpp` (1), `Handlers/AI/AIHandler.cpp` (1),
  `Handlers/Actor/ActorTransformHandler.cpp` (2), `Handlers/Actor/SpawnBatchHandler.cpp` (2),
  `Handlers/Audio/AudioHandler.cpp` (2), `Handlers/Blueprint/SCSHandler.cpp` (1),
  `Handlers/Render/PreviewViewportCaptureUtils.cpp` (1), `Handlers/Render/ZFightingHandler.cpp` (1),
  `Handlers/Spatial/MeasureHandler.cpp` (5), `Handlers/Spatial/PlacementHandler.cpp` (3),
  `Handlers/Spatial/ScatterLayoutHandler.cpp` (3), `Handlers/Systems/GameFrameworkHandler.cpp` (1),
  `Handlers/Volume/VolumeHandler.cpp` (1).

Two multipliers on top of that count:

1. **`Ctx.GetVector` is one of the 24.** `HandlerContext.cpp:53` is
   `FVector FHandlerContext::GetVector(...)` and `:59` is `return ExtractVectorField(Payload, *Key, Default);`
   — so every verb that reads a vector through the context helper inherits the narrowing.
   `GetVector(` has **20 call sites across 13 further files**.
2. **`ReadVectorField` shares the implementation.** The public mutating wrapper
   (`JsonUtils.h:25`, "Backward-compatible mutating helpers used by legacy call sites") calls the
   same `ReadVectorFieldImpl`, adding 3 more production sites:
   `Handlers/Blueprint/BlueprintComponentHandler.cpp:360`, `:362`, and
   `Handlers/Editor/ViewportHandler.cpp:203`.

The practical statement is the one that matters for severity: **no coordinate can enter PinWright
through a `location`, `bounds`, `region`, `center`, `min`, `max`, `scale`, `offset` or
`world_point` parameter without being truncated to float32 first**, with `ParseVectorFromJson`
as the sole exception.

## Same shape as

`ReadRotatorFieldImpl`, immediately below in the same file, narrows identically
(`JsonUtils.cpp:71`, `:77-78`), and `FRotator` is also `double` in UE5 — so the defect is
literally adjacent. It is called out here rather than filed, because rotator components are
bounded by ±360: float32 holds 360 to about 3e-5 degrees, so the loss is real but has no
reachable consequence. **Fix it in the same edit for consistency; do not claim it as a second
bug.** That asymmetry — same cast, harmless there, harmful here — is exactly why the vector case
matters: the magnitude of the *values*, not the shape of the code, is what makes it a defect.

## Not the same as

`B-declared-param-guard-blind-to-nested-keys` (OPEN, High) is the closest textual neighbour and
cites the same lines — `Utils/JsonUtils.cpp:16-49` at its `:93`, and
`ReadVectorFieldImpl`'s capitalised `X`/`Y`/`Z` and bare 3-element array form at `JsonUtils.cpp:31-44`
at its `:167`. **The mechanisms do not overlap.** That ticket is about *which keys the declared-param
guard can see* — a nested key is accepted, inert, and named by no test. This one is about *what
type a key that is read correctly lands in*. Its fix (widen the guard to nested keys) changes
nothing here; this fix (drop three casts) changes nothing there. Both can land independently, in
either order, and the fixer of either should not assume the other covers it.

`B-create-ambient-sound-location-object-dropped` (IN-REVIEW, Medium) is not contradicted and is
not at fault — its object-vs-array claim is correct and its fix is the right one. It is
cross-linked because that fix **routed one more verb onto this helper**: history `#2` replaced
`create_ambient_sound`'s array-only `TryGetArrayField` parse with
`ExtractVectorField(RawPayload, TEXT("location"), FVector::ZeroVector)`, now at
`AudioHandler.cpp:637` (`:632` is the comment naming the change; the sibling
`create_audio_component` is `:904`). A fixer should know the lossy path gained a call site
rather than lost one — that is a note about direction of travel, not a defect in that ticket.
Two incidental observations from verifying it, neither of which changes its own claim: it names
the pre-rename module path `Source/EditorAutomationRpcGateway/...`, and the regression test it
reports adding, `FExtractVectorFieldAmbientSoundObjectLocationTest` /
`...extract_vector_field.AmbientSoundObjectLocation`, does not exist anywhere under
`Plugins/PinWright/` — `grep -rn AmbientSoundObjectLocation Source/ Docs/` returns nothing. That
ticket is IN-REVIEW and unverified, so this is for its tester, not a claim against it here.

## Severity

**High.** Impact class is the rubric's High band verbatim — *"silent wrong ... data on a normal
path (the caller trusts a result that is a lie and builds on it)"*. There is no error, no
warning, no response field recording that the value was changed, and no readback that catches it:
a `property.get` returns the truncated double the plugin actually applied, so the value is
self-consistently wrong everywhere the caller can look. The one place it does surface, it
surfaces as a **different** error about a different thing (`MeasureHandler.cpp:809`), which is
worse than silence.

**Both reach modifiers were considered and both are declined; here is each.**

*Declining the bump down to Medium.* The honest argument for Medium is that at ordinary map scale
nothing is wrong: below 83.9 km every integer centimetre round-trips exactly, so most sessions on
most maps never observe this. That is true and it is the strongest counter-argument in the
ticket. It is declined because the engine, not this ticket, picks the line: `UE_FLOAT_HUGE_DISTANCE`
is **10.49 km**, and this plugin's own level-audit code adopts that number as its world-bounds
threshold for exactly this reason (`LevelAuditUtils.cpp:60-66`). A defect whose safe zone ends
where the engine's own documented float-precision horizon ends is not an edge path; it is the
normal path with a low ceiling, and World Partition exists to build past that ceiling.

*Declining the bump up to Critical.* `Ctx.GetVector` plus `ExtractVectorField` plus
`ReadVectorField` genuinely do run in almost every session, which is the rubric's stated trigger
for a one-band bump — and one band up from High is Critical. Declined, because Critical is
defined as "editor crash, or a write that corrupts or loses asset data", and this is neither: it
writes a wrong value, not a corrupt one, and the affected assets stay loadable and internally
consistent. Applying a mechanical reach bump into a band whose definition the defect does not
meet would mis-sort it above real crashes in the picker. The reach is instead the reason the
downward bump is refused, which is where it does honest work.

Net: **High**, no modifier applied, both named.

## Fix

Delete the three casts — `JsonUtils.cpp:36` and `:42-43` — so the doubles that `TryGetNumberField`
and `AsNumber()` already produced land in the double `FVector` unchanged. `ReadRotatorFieldImpl`
(`:71`, `:77-78`) should go in the same edit.

**What to check before doing it,** in order:

1. **Whether any consumer depends on the truncation.** They should not: `FVector` is `double`
   throughout UE5, every call site assigns straight into an `FVector`, `FBox`, an actor location
   or a trace argument, and none of the 24 production sites re-narrows afterwards. Confirm by
   reading them rather than assuming — a site that itself casts to `float` for a legacy API would
   be unaffected either way, but a site comparing against a hardcoded truncated constant would
   change behaviour.
2. **Whether any existing test pins the truncated value.** `Tests/Core/TestJsonUtils.cpp` holds
   21 of the 48 references. Its current vector cases (`FExtractVectorFieldValidObjTest` `:13`,
   `ValidArray` `:33`, `Partial` `:53`, `Missing` `:73`, `NullSource` `:88`, `ShortArray` `:239`)
   all use small values that are exact in both float and double, so none should move — verify,
   do not assume.
3. **Add the differential test, in that same file.** A value that survives a double round-trip
   and fails a float one: `1000003000` in, `1000003000` expected out, which returns `1000003008`
   against today's code. Pair it with a span case — `min.x = 1e12`, `max.x = 1e12 + 3000`,
   asserting the span is 3000 and not 0 — because the span is the case that produces a wrong
   *answer* rather than a wrong *digit*, and it is the one `MeasureHandler.cpp:806` turns into a
   misleading refusal. Assert exact equality, not `IsNearlyEqual`; a tolerance test would pass
   against the bug.

## History
- `#1-float32-narrowing-at-rpc-boundary` `OPEN` reporter — `ExtractVectorField` casts every parsed JSON coordinate to `float` before storing it in a `double` `FVector`, so world coordinates are silently truncated to float32 at the RPC boundary. `Utils/JsonUtils.cpp`: `ReadVectorFieldImpl` at `:16`, `double` locals at `:29` filled by `TryGetNumberField`, object-form cast `Out = FVector((float)X, (float)Y, (float)Z);` at `:36`, array-form cast at `:42-43`; `ExtractVectorField` at `:140` calls it at `:145`. The cast sits between a double source and a double destination and is pure loss. **`ParseVectorFromJson` in the same file at `:158` declares its locals `double` (`:168`) and returns `FVector(X, Y, Z)` at `:172` with no narrowing** — two helpers, one translation unit, disagreeing about the same operation, which is the cheapest proof this is an oversight. Measured against IEEE-754 binary32: a region `max.x` of `1000003000` returns `1000003008` (8 cm error at 10,000.03 km); from `2^35` cm = 343,597 km a valid 3000 cm span collapses to exactly zero (still 128× inside `WORLD_MAX/2`), and at `1e12` cm both bounds land on `999999995904`. The collapse is then reported as the handler's own degenerate-region refusal — `spatial.find_clear_placement` (`MeasureHandler.cpp:638`) parses at `:804-805`, tests at `:806`, and answers `"region.max must exceed region.min on every axis."` at `:808-809`, telling the caller their region was empty when their coordinates were truncated into each other; `spatial.scatter_layout` repeats it at `ScatterLayoutHandler.cpp:116-118`, `:120`, surfaced `:235-236`. Grounding: `EngineDefines.h:59` defines `UE_FLOAT_HUGE_DISTANCE` as 1,048,576 cm (10.49 km) — "the largest distance a float still holds to 1/16 cm" — and **this plugin already adopts that exact number as its world-bounds lint threshold** (`LevelAuditUtils.h:175`, reasoned out at `LevelAuditUtils.cpp:60-66`), while the default world reaches 43,980,465 km from origin (`EngineDefines.h:41`, `:45-54`; `UE_USE_UE4_WORLD_MAX` not overridden in this checkout) — 4,194,304× further out than the cap this cast installs. Reach, counted: 50 references in the tree; 48 outside `JsonUtils.cpp`/`.h` (confirming the reported figure), of which 21 are `Tests/Core/TestJsonUtils.cpp` and 27 are handler `.cpp` — 3 of those comments, leaving **24 production call sites across 13 files**. `Ctx.GetVector` is one of them (`HandlerContext.cpp:53`, `:59`) with 20 call sites in 13 further files, and the public `ReadVectorField` wrapper (`JsonUtils.h:25`) shares the impl for 3 more (`BlueprintComponentHandler.cpp:360`, `:362`, `ViewportHandler.cpp:203`). Severity High: impact class High per the rubric's "silent wrong data on a normal path" — no error, no warning, and a `property.get` readback returns the same truncated value, so the lie is self-consistent — with **both** reach modifiers considered and declined: not down to Medium (the safe zone ends at the engine's own 10.49 km float horizon, which is the normal path with a low ceiling, not an edge path), and not up to Critical (the helper does run in almost every session, but Critical means crash or corrupt/lost asset data and this writes a wrong value, not a corrupt one). Fix is deleting `:36` and `:42-43` (plus `ReadRotatorFieldImpl` `:71`/`:77-78` for consistency — harmless there, rotators are bounded by ±360); a fixer must first confirm no consumer depends on the truncation (`FVector` is double throughout, so they should not) and that no existing `TestJsonUtils.cpp` case pins a truncated value, then add a differential test there — `1000003000` exact through the round trip, and a `1e12` / `1e12+3000` span asserting 3000 not 0, with exact equality rather than `IsNearlyEqual`. Dedup: `B-declared-param-guard-blind-to-nested-keys` (OPEN, High) cites the same lines (`JsonUtils.cpp:16-49` at its `:93`, `:31-44` at its `:167`) but is about which keys are validated, not what type they land in — the two fixes are independent in both directions. `B-create-ambient-sound-location-object-dropped` (IN-REVIEW, Medium) is not contradicted; its shipped fix routed `create_ambient_sound` onto this helper (`AudioHandler.cpp:637`), adding a call site to the lossy path. Two incidental notes for that ticket's tester, not claims against it: it cites the pre-rename module path `Source/EditorAutomationRpcGateway/...`, and the regression test its `#2` reports adding (`AmbientSoundObjectLocation`) does not exist anywhere under `Plugins/PinWright/`.
- `#2-dropped-the-narrowing-casts` `IN-REVIEW` developer — "Deleted all four casts. `Private/Utils/JsonUtils.cpp` `ReadVectorFieldImpl` object form is now `Out = FVector(X, Y, Z);` and array form `Out = FVector((*Arr)[0]->AsNumber(), (*Arr)[1]->AsNumber(), (*Arr)[2]->AsNumber());`; `ReadRotatorFieldImpl` got the identical treatment for `FRotator` in the same edit, for consistency and not as a second bug — rotator components are bounded by ±360 where float32 still holds ~3e-5 degrees. `grep -n '(float)' Utils/JsonUtils.cpp` now returns nothing. Nothing else in the file was touched; `ParseVectorFromJson`/`ParseRotatorFromJson` were already correct and are unchanged. The reported line numbers were still accurate (`:36`, `:42-43`, `:71`, `:77-78`). **The three pre-checks the ticket asked for, run before the edit.** (1) *No consumer depends on the truncation.* The 24 `ExtractVectorField` production sites across 13 files and the 3 `ReadVectorField` sites reconcile exactly with the reported counts; `grep -A4` over every one of them for `==`, `Equals`, `FMath::IsNearlyEqual` or a `(float)` cast returns **zero** hits, so no site compares a parsed vector against a hardcoded constant or re-narrows it — every one assigns straight into an `FVector`/`FBox`/actor location/trace argument. (2) *No existing test pins a truncated value.* All six vector cases and all six rotator cases in `Tests/Core/TestJsonUtils.cpp` use values exact in both binary32 and binary64 (1/2/3, 4/5/6, 10/99, 7/8/9, 15/90/45, 11/22/33, 44/55/66), as does `Tests/Infra/TestHandlerContext.cpp` `GetVector` (1/2/3, 10/20/30) — none moves. (3) *Differential regression tests added*, both in `Tests/Core/TestJsonUtils.cpp`: `PinWright.core.json.extract_vector_field.FarFromOriginObject` (object form; `x=1000003000`, `y=-1000003000`, `z=1000000003000` in, the same values expected out) and `.FarFromOriginSpan` (array form; `min=1e12`, `max=1e12+3000` on all three axes, asserting each span is exactly 3000 and that `Max > Min` on every axis — the shape `MeasureHandler.cpp:806` and `ScatterLayoutHandler.cpp:118` turn into a false 'max must exceed min' refusal). Both assert through `TestEqual(..., 0.0)`, i.e. exact equality with tolerance zero rather than `IsNearlyEqual`, so a tolerance could not mask the loss. Against pre-fix code they fail by arithmetic: `1000003000` sits in `[2^29,2^30)` where the binary32 ULP is 64 and rounds to `1000003008` (8 cm), and `1e12` and `1e12+3000` both sit in `[2^39,2^40)` where the ULP is 65536 and both round to `999999995904`, collapsing the span to 0. Neither test was executed here — per instruction the suite is run by the wave owner after the build, so this is an unrun-but-derived claim, not a measured one. Both new ids are leaves and neither is a dot-prefix of the other or of any existing id in the file (`ValidObject`, `ValidArray`, `PartialObject`, `Missing`, `NullSource`, `ShortArray`). **No `AssetDumpCache` aspect-version bump is needed and none was made:** `GetAspectVersion` governs dumper *serialized output*, and `ExtractVectorField` has zero call sites under `Handlers/Asset/` — it is an inbound parse helper, so no sidecar bytes change. The wire format is likewise unchanged (JSON numbers in both directions; only the precision retained between them changes), no Python helper or wiki page documents a float32 coordinate contract (`grep -rn 'float32\|single.precision' Docs/` finds only an unrelated Sequencer channel note), and no doc edit was made. **One mechanism the ticket did not name, fixed by the same edit:** the helper was inconsistent about the *default* as well as the payload — an absent field returned `Default` unnarrowed via `Out = Default`, but a present object missing one axis pushed that same default through the narrowing `FVector` constructor. So `actor.set_transform` with `{"location":{"x":…}}` and no `y`/`z` truncated the actor's own current location, which `ActorTransformHandler.cpp:52` passes in as the default. Files changed: `Source/PinWright/Private/Utils/JsonUtils.cpp`, `Source/PinWright/Private/Tests/Core/TestJsonUtils.cpp`. Nothing in the ticket was contradicted; every count, line number and quoted engine constant checked out against the current tree."
