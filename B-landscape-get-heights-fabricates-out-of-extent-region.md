---
id: B-landscape-get-heights-fabricates-out-of-extent-region
title: "landscape.get_heights clamps an out-of-extent region to a single pixel without a warning, then echoes the engine's uninitialised INT_MAX/INT_MIN out-params as the region it sampled, and publishes the uint16 midpoint as a real height — the same hardening its sibling paint verb received reached this verb's input side and not its output side"
status: OPEN
severity: High
category: bug
tags: [landscape, get_heights, region, clamp, sentinel, int-max, silent-wrong-data, no-warning, fabricated-readback, half-applied-fix]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The fix is in this handler already. It stops one line short of the response.

`landscape.get_heights` with `region: {minX: 2500, minY: 1000, maxX: 2540, maxY: 1040}` on a
505x505 landscape returned:

- `success: true`
- **one** sample, at the uint16 midpoint 32768, reported as world `Z 0`
- `region: {minX: 2147483647, minY: 2147483647, maxX: -2147483648, maxY: -2147483648}`

`2147483647` is `INT_MAX` and `-2147483648` is `INT_MIN`. Those are not coordinates. They are the
initial values of a min/max accumulator that never accumulated, returned to the caller as data.

## Where each number comes from

**The clamp is real and silent.** `Handlers/Environment/LandscapeHandler.cpp:1917-1918` calls the
shared helper:

```cpp
1917:  const LandscapeHeightStats::FResolvedHeightRegion Region = LandscapeHeightStats::ResolveHeightRegion(
1918:      ReqMinX, ReqMinY, ReqMaxX, ReqMaxY, FullMinX, FullMinY, FullMaxX, FullMaxY);
```

`ResolveHeightRegion` (`Handlers/Environment/LandscapeHeightStats.cpp:17-37`) clamps all four
coordinates into the landscape extent at `:30-33` and sets `bValid` at `:35`. For the measured
request every coordinate is above `FullMaxX`/`FullMaxY` = 504, so all four clamp to 504: the region
becomes `(504,504)..(504,504)`, `bValid` is true, `SizeX = SizeY = 1` (`:1930-1931`) and
`sampleCount` is 1. **The helper has no warning channel** — it returns a plain struct — and
`get_heights` has no `warnings` array anywhere in its body (`:1848-1978`). A caller who asked for a
41x41 window and got a 1x1 one is told nothing.

**The sentinels come from the engine, not from this file.** `grep` for `INT_MAX` / `MAX_int32` /
`TNumericLimits<int32>` across all 2694 lines of `LandscapeHandler.cpp` returns exactly one hit and
it is a comment (`:2157`). The handler copies the resolved region into four **mutable** locals and
hands them to the engine as out-params:

```cpp
1923:  // Non-const: FLandscapeEditDataInterface::GetHeightData takes these by int32& and writes back
1924:  // the actual sampled sub-region, which is then reported in the response's "region" object.
1925:  int32 MinX = Region.MinX;   // .. 1926-1928 MinY / MaxX / MaxY
1952:  LandscapeEditRead.GetHeightData(MinX, MinY, MaxX, MaxY, Heights.GetData(), 0);
1970:  RegionResp->SetNumberField(TEXT("minX"), MinX);   // .. 1971-1973
```

`FLandscapeEditDataInterface::GetHeightDataInternal`
(`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeEditInterface.cpp:859`) opens by
seeding those references with sentinels:

```cpp
862:  int32 X1 = ValidX1, X2 = ValidX2, Y1 = ValidY1, Y2 = ValidY2;
863:  ValidX1 = INT_MAX; ValidX2 = INT_MIN; ValidY1 = INT_MAX; ValidY2 = INT_MIN;
```

and narrows them **only inside `if (Component)`** (`:908-918`, the updates at `:914-917`). On the
missing-data path it clamps back toward the request at `:1291-1294` — `Max(X1, INT_MAX)` is
`INT_MAX` and `Min(X2, INT_MIN)` is `INT_MIN`, so with zero components found the sentinels survive
intact. (On the normal path `:1296-1300` returns the request verbatim, which is why this has never
been seen before.) The handler then echoes them at `:1970-1973` with no re-assertion that
`MinX <= MaxX`.

**The height is the neutral no-data value.** With no component the engine's `bHasMissingValue`
branch runs `CalcMissingValues` (`LandscapeEditInterface.cpp:1283`) over the zeroed buffer
(`LandscapeHandler.cpp:1944-1945`), and `BuildHeightStatsJson`
(`LandscapeHeightStats.cpp:39-99`) reduces whatever lands there into `minHeight` / `maxHeight` /
`meanHeight` (`:73-75`) and converts through the 32768/128 convention into `minZ` / `maxZ` /
`meanZ` (`:80-82`). 32768 is the midpoint, so it publishes as world `Z 0` — a number that reads
like a measurement of flat terrain at sea level and is in fact the absence of a measurement.

Why no component was found for a region that passed the clamp: `GetLandscapeExtent`
(`LandscapeHandler.cpp:1902-1915`) reports the bounding box of the landscape's component map, so
the clamped corner `(504,504)` is inside the reported extent by construction — but the engine's
`CalcComponentIndicesNoOverlap` (`LandscapeEditInterface.cpp:866`) has to map that far-edge vertex
onto a component index, and the sentinels prove it found none. That the inclusive max vertex lands
one component past the last component is the obvious candidate and is **an inference, not a
measurement** — it was not verified, and it does not change the defect: whatever the reason, the
handler cannot tell "sampled nothing" from "sampled everything" and reports the second.

## The headline: this verb already has the fix, on the wrong side

`B-create-procedural-terrain-paints-nothing` (DONE, High) hardened exactly this shape, and its
`#2` introduced `ResolveHeightRegion`, the `INVALID_ARGUMENT` refusal, and clamps reported in
`warnings[]`. **The two verbs do share the helper** — there is one overload, called from four
production sites: `landscape.edit` (`LandscapeHandler.cpp:1735`), **`landscape.get_heights`
(`:1917`)**, `landscape.create_procedural_terrain` (`:2173`) and `landscape.audit_shape`
(`:2519`). So the brief's guess that the hardening never reached this caller is **wrong**, and the
truth is worse: it reached the input side and stopped there.

Set the two verbs side by side. Same helper, same resolved region, opposite handling of the result:

| | `create_procedural_terrain` (hardened) | `get_heights` (this ticket) |
|---|---|---|
| resolve | `:2173` | `:1917` |
| empty after clamp | `INVALID_ARGUMENT` `:2175-2183` | `INVALID_ARGUMENT` `:1919-1922` |
| clamp reported | warning `:2195-2202`, emitted `:2391-2392` | **nothing — no `warnings` array exists** |
| region echoed from | **`const` copies** `:2185-2188`, echoed `:2386-2389` | **mutable engine out-params** `:1925-1928`, echoed `:1970-1973` |

The paint verb's own comment at `:2193-2194` states the principle this verb violates: *"Report a
region the caller asked for but did not get, instead of silently substituting the clamped one."*
And its `const` at `:2185-2188` is what makes the sentinel echo structurally impossible there.
Both halves of the answer are already written, thirty lines apart in the same file.

## Expected behaviour

1. **Refuse a region with no overlap.** The brief's expectation — `INVALID_ARGUMENT`, as the
   sibling paint verb does — needs one correction to be implementable: `ResolveHeightRegion`
   `:30-35` clamps *before* testing validity, so a wholly out-of-extent request never comes back
   `bValid == false`; it comes back as the nearest edge pixel. Detecting no-overlap needs a test
   against the **requested** rectangle (`ReqMinX > FullMaxX || ReqMaxX < FullMinX || ...`) before
   or alongside the clamp. Adding that to the helper fixes all four callers at once.
2. **Warn on a partial clamp**, with the same message the paint verb already emits
   (`:2199-2201`), and give `get_heights` the `warnings[]` array it currently lacks.
3. **Echo `Region.MinX..Region.MaxY`, not the post-call locals** — i.e. take the paint verb's
   `const` copies at `:2185-2188`. This alone removes the sentinels from the wire.
4. **Do not report interpolated fill as samples.** The engine already knows: after `:1952` a
   `MinX > MaxX` means nothing was sampled. At minimum turn that into an error rather than a
   response; better, publish the fact, since `sampleCount: 1` and `meanZ: 0` are indistinguishable
   from a genuine one-pixel read of flat ground.

## Same shape as

`B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) carries the fullest statement of the
class: the call succeeds, every number it reports is correct, and the output is wrong because the
deciding number was never reported. This one is a sharper variant — the deciding number *is*
reported, as `INT_MAX`, and is self-describing to anyone who recognises it. It is returned as data
rather than as an error, so nothing consuming the response programmatically will ever notice.

Nearest members:

- `B-create-procedural-terrain-paints-nothing` (DONE, High) — the fix that exists one call site
  over. The most important link on this ticket: a fixer should read its `#2` and then port the
  output half, not re-derive it.
- `F-landscape-height-readback` (IN-REVIEW, Low) — the ticket whose `#2` shipped this verb. The
  defect arrived with the feature; the `region` echo was specified as "the actual sampled
  sub-region" (`:1923-1924`) and that specification is exactly the vector, because the engine
  expresses "no sub-region" as a sentinel rather than a failure.
- `E-landscape-edit-extent-error-not-diagnostic` (OPEN) — the same namespace's complaint that an
  extent-related error does not say what the extent was. Here there is not even an error to be
  undiagnostic: the caller has to recognise `2147483647` on sight.

## What was NOT done

- No source was modified.
- Only the fully-out-of-extent case was measured. A **partially** overlapping region — which
  clamps to a real multi-pixel sub-rectangle, finds components, and therefore takes the engine's
  `:1296-1300` verbatim-return path — was not called. Source says it returns correct heights and
  a correct region with the clamp still unreported, so the expected symptom there is the missing
  warning only. Untested.
- The reason no `ULandscapeComponent` was found at the clamped far-edge vertex is an inference
  (see above), not a measurement.
- `landscape.edit` (`:1735`) and `landscape.audit_shape` (`:2519`) share the helper and were not
  called. `landscape.edit` does not hand the resolved region to `GetHeightData` as out-params in
  the same shape, so it is not implicated by source read; `audit_shape` was not checked.

severity rationale: impact=High — the README's High band verbatim, "silent wrong / stale / hardcoded data on a normal path (the caller trusts a result that is a lie and builds on it)". A height readback exists to be trusted: `success: true`, `sampleCount: 1`, `meanZ: 0` is a well-formed answer that a caller verifying a sculpt will read as "this area is flat at sea level", when the true answer is "nothing in your region exists and nothing was measured". The uint16 midpoint publishing as exactly `0` is the worst possible fabricated value, because 0 is a plausible ground height rather than an obvious tell. NOT Critical: no crash, and the verb is a pure read — `MakeLandscapeEditInterfaceReadOnly` (`:1951`) is applied before the call and nothing is written, so no asset data is corrupted or lost. NOT Medium: Medium is a soft blocker reachable via a documented workaround, and there is no workaround here because there is no signal to work around — the caller does not know the answer is wrong. I weighed and reject the argument that `INT_MAX` in the echo makes this self-announcing: it announces itself to a human who recognises the constant, and this board's callers are agents that consume `region` as four numbers; the field is typed as a region and carries a syntactically valid region. It is also not in the field a caller checks — `sampleCount` and `meanZ` look fine, and `region` is the field you read *after* you already believe the result. x reach: BOTH modifiers declined. Not a bump up — `landscape.get_heights` is one verb in one namespace, not an almost-every-session method. Not a bump down — reading heights back is the standard way to verify any sculpt or terrain edit, so it sits on the verification path, and a region argument that overshoots the extent is the ordinary consequence of working in world units against a landscape whose vertex extent the caller has not looked up; that is a normal path, not a rare edge one. High stands unmodified.

## History
- `#1-sentinel-region-echoed-as-data` `OPEN` reporter — Measured live against a running editor. `landscape.get_heights {region:{minX:2500,minY:1000,maxX:2540,maxY:1040}}` on a 505x505 landscape returned `success:true`, **one** sample at the uint16 midpoint 32768 (published as world `Z 0`), and `region:{2147483647, 2147483647, -2147483648, -2147483648}` — `INT_MAX`/`INT_MIN`. All line numbers re-derived this session at HEAD. Mechanism, three parts: (1) **the clamp is real and unreported** — `ResolveHeightRegion` (`LandscapeHeightStats.cpp:17-37`, clamps `:30-33`, `bValid` `:35`) called at `LandscapeHandler.cpp:1917-1918` pulls all four coordinates to the extent max 504, giving a 1x1 region (`SizeX/SizeY` `:1930-1931`), and `get_heights` has **no `warnings` array anywhere in its body** (`:1848-1978`); (2) **the sentinels are the engine's, passed through** — the handler copies the region into MUTABLE locals `:1925-1928` under a comment `:1923-1924` that documents the intent, hands them to `GetHeightData` as out-params `:1952`, and echoes the post-call values `:1970-1973` with no `MinX<=MaxX` re-check. `FLandscapeEditDataInterface::GetHeightDataInternal` (`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeEditInterface.cpp:859`) seeds them `:863` `ValidX1=INT_MAX; ValidX2=INT_MIN; ...`, narrows only inside `if (Component)` `:908-918` (updates `:914-917`), and on the missing-data path clamps back at `:1291-1294` where `Max(X1,INT_MAX)==INT_MAX` — so zero components found means the sentinels reach the wire verbatim; the normal path returns the request verbatim at `:1296-1300`, which is why this was never seen. `grep` for `INT_MAX`/`MAX_int32`/`TNumericLimits<int32>` over all 2694 lines of `LandscapeHandler.cpp` returns one hit and it is a comment (`:2157`) — no sentinel is written in plugin code. (3) **the height is the no-data value** — engine `CalcMissingValues` `:1283` over the zeroed buffer (`:1944-1945`), reduced by `BuildHeightStatsJson` (`LandscapeHeightStats.cpp:39-99`, min/max/mean `:73-75`, world-Z conversion `:80-82` via the 32768/128 convention), so the midpoint publishes as exactly `Z 0`. **HEADLINE, and it corrects the framing this was filed under:** the two verbs DO share the helper — one overload, four production callers, `landscape.edit` `:1735`, `get_heights` `:1917`, `create_procedural_terrain` `:2173`, `audit_shape` `:2519` — so `B-create-procedural-terrain-paints-nothing` `#2`'s hardening DID reach this caller, on the input side only: both refuse an empty region with `INVALID_ARGUMENT` (`:1919-1922` vs `:2175-2183`), but the paint verb copies the region into `const` locals `:2185-2188` and echoes those `:2386-2389`, warns on any clamp `:2195-2202` into `warnings[]` `:2391-2392`, and states the principle in its own comment `:2193-2194` ("Report a region the caller asked for but did not get, instead of silently substituting the clamped one"). Both halves of the fix are already written thirty lines away in the same file. One correction to the expected behaviour as briefed: `INVALID_ARGUMENT` for a wholly out-of-extent region is NOT reachable through `bValid`, because `:30-35` clamps before testing, so a no-overlap request always resolves to a valid edge pixel — detecting it needs a test against the REQUESTED rectangle, best added inside the helper so all four callers get it. NOT DONE: no source modified; only the fully-out-of-extent case was called, the partially-overlapping case (which finds components and takes the engine's verbatim-return path, so the predicted symptom is the missing warning only) is untested; the reason no component was found at the clamped far-edge vertex is an inference about `CalcComponentIndicesNoOverlap` (`LandscapeEditInterface.cpp:866`), not a measurement. Dedup: `grep -ril` over the board for landscape/height/region tickets returns 18 files; none claims a sentinel region echo. `B-landscape-get-heights-dirties-map` is a different defect in the same verb (an EDIT interface on a read path) and is not this; `F-landscape-height-readback` shipped the verb and is cited; `E-landscape-edit-extent-error-not-diagnostic` is the adjacent diagnostics gap. No umbrella filed; the class statement stays in `B-foliage-paint-does-no-ground-projection` and is referenced.
