---
id: B-tests-wipe-host-map-foliage
title: "Two foliage tests called foliage.remove {removeAll:true} against the live editor world, deleting the host map's own foliage on every suite run"
status: IN-REVIEW
severity: High
category: bug
tags: [tests, hygiene, host-safety, foliage, shared-state, destructive-teardown]
encounters: 1
lastSeen: 2026-08-29T09:00:00Z
---

# A test's teardown emptied the host map, not its own fixture

`foliage.remove` takes either a `foliageTypePath` (scoped) or `removeAll:true` (every instance of
every type in the editor world). Two tests in
`Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp` used the second form against the
**live editor world**, because `TestUtils.h::InvokeHandler` runs the real registered handler against
`GEditor`'s world — there is no sandbox:

- `PinWright.foliage.remove.ValidParamsNoCrash` (`FFoliageRemoveValidParamsTest`) — its entire body
  was a `removeAll:true` call. The only assertion is `TestTrue("foliage.remove handler found", ...)`;
  the destructive payload was never asserted on.
- `PinWright.foliage.get_instances.RoundTripsScaleAndRotation`
  (`FFoliageGetInstancesRoundTripsScaleTest`) — `removeAll:true` twice, once as a pre-clear ("so our
  written instance is the only one read back") and once in `ON_SCOPE_EXIT`.

Every suite run on a host whose map carries authored foliage therefore deleted it. Unlike
`B-tests-destroy-host-assets` this needs no client RPC: the suite reaches it directly.

The correct pattern already existed in the same tree —
`Tests/World/TestFoliageAddInstancesPrecedenceHonesty.cpp:82` scopes its teardown by
`foliageTypePath` with the reason written out — and the two newer behavioural files
(`Tests/Environment/TestFoliagePlacementBehaviour.cpp:197`,
`TestFoliagePaintGroundProjection.cpp:214`) do the same. Only these two sites were left.

## Fix

Both converted to type-scoped removal; `SetBoolField(TEXT("removeAll"), true)` now appears **nowhere**
in any test source.

`RoundTripsScaleAndRotation` needed one further change to stay correct without the world-wide clear:
its unfiltered `foliage.get_instances` branch read `instances[0]` and relied on the world holding
exactly one instance. That assumption dies with the pre-clear, so the shared assertion helper
(`AssertFirstInstanceTransform` → `AssertOurInstanceTransform`) now takes the fixture's own
`foliageType` and locates that entry in the response instead of taking index 0; the filtered branch
passes an empty type because its query is already scoped. This strengthens the test — it no longer
depends on the world being empty — and asserts the same fields.

`ValidParamsNoCrash` now sends a GUID-unique nonexistent `foliageTypePath`. Its assertion ("the
handler is registered and answers without crashing") is unchanged; what is lost is smoke coverage of
the `removeAll` branch, which cannot be exercised against a live host map at all. That branch's
behaviour is covered by `PinWright.foliage.remove.MissingScopeErrors`
(`Tests/Environment/TestFoliageRemoveEdgeInputHonesty.cpp`) and
`.ScopedRemovalSparesOtherTypes` (`Tests/Environment/TestFoliagePlacementBehaviour.cpp`).

## History
- `#1-found-and-scoped` `OPEN` reporter — "Found while sweeping for unmarked conditional skips (B-tests-warn-and-pass-without-skip-marker). Three `removeAll:true` call sites in Tests/World/TestEnvironmentHandlers.cpp run the real foliage.remove handler against GEditor's world: one is the whole body of foliage.remove.ValidParamsNoCrash, two are the pre-clear and ON_SCOPE_EXIT teardown of foliage.get_instances.RoundTripsScaleAndRotation. Suite-reachable, unlike B-tests-destroy-host-assets."
- `#2-type-scoped-teardown` `IN-REVIEW` developer — "All three scoped to foliageTypePath; grep confirms no `SetBoolField(TEXT(\"removeAll\"), true)` remains under any PinWright*/Private/Tests tree. RoundTripsScaleAndRotation's unfiltered branch rewritten to find its own instance by `foliageType` instead of assuming instances[0], since the world is no longer emptied first. ValidParamsNoCrash sends a GUID-unique nonexistent type; its single assertion is unchanged. NOT COMPILED and NOT RUN — a full automation suite held the DLLs for the duration of this pass, so the compile of the new lambda signature and the unfiltered branch's new selection logic are both unverified."
