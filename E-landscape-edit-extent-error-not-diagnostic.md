---
id: E-landscape-edit-extent-error-not-diagnostic
title: "landscape.edit emits bare [INVALID_LANDSCAPE] Failed to get landscape extent for a component-less landscape, naming a symptom not the cause and driving path-form trial-and-error"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [landscape, edit, error-message, diagnostic, hollow, components, extent]
---

# `landscape.edit` says "Failed to get landscape extent" without naming the real cause

When a target `ALandscape` has **zero registered `ULandscapeComponent`s** (a
hollow landscape — see the judge-filed bug
`B-landscape-create-hollow-no-components`, where UE 5.7 `landscape.create`
spawns exactly this), `landscape.edit` rejects every operation with the bare
error `[INVALID_LANDSCAPE] Failed to get landscape extent`. That message names
the *symptom* the handler hit (it couldn't compute an extent) rather than the
*cause* (the landscape has no component grid, so there is nothing to edit). The
companion surface is worse: `landscape.sculpt` on the same hollow actor returns
`success:true, modifiedVertices:0` — no error at all.

Because the extent error reads like a transient or addressing failure, a caller
who just received `success:true` from `landscape.create` cannot tell which of
several plausible explanations applies — wrong actor name form, wrong actor,
needs the material re-registered, or the landscape is genuinely non-functional.
With nothing in the error to disambiguate, the caller probes. This is the same
confusing-error-drives-trial-and-error class already filed as
`E-level-load-file-not-found-vs-in-memory-orphan` (a bare `FILE_NOT_FOUND` for
an in-memory-but-unsaved world), filed there as a distinct ergonomic angle on
top of a separate cause bug — exactly the relationship here.

## Why it matters — the process friction

In the audited `landscape` blockout task the unclear error drove a retry storm
and a deep source dive that a diagnostic message would have collapsed into one
read. From the call log:

- **5** `landscape.edit` attempts, all returning `[INVALID_LANDSCAPE] Failed to
  get landscape extent`: raise-region **by name**, then **by path**, then on a
  **pre-existing `Landscape_1`** (to rule out "my new actor is bad"), then a
  **3x3 `set` heightData** variant, then a **retry after `landscape.set_material`**
  (to rule out "material not registered"). Each is a hypothesis the caller had
  to test *because the error named no cause*.
- **2** `landscape.sculpt` no-ops (`modifiedVertices:0`, `success:true`) that
  gave the opposite false signal — apparent success with zero effect.
- **4** diagnostic calls to reverse-engineer the condition the error should have
  stated outright: `actor.get_bounding_box` (all-zero origin/extent),
  `actor.get_components` on the new actor **and** on `Landscape_1` (both
  `count:1`, only `RootComponent0`, zero LandscapeComponents),
  `actor.describe componentClass=LandscapeComponent` (`components:[]`).

The friction note records the cost directly: "I had to read the plugin's
LandscapeHandler.cpp plus engine LandscapeEdit.cpp/Landscape.cpp to diagnose
that the 5.7 create branch ... never imports/registers components ... the silent
modifiedVertices:0 with success:true is a discoverability trap (no error tells
the caller the landscape is non-functional)." Three source files plus eleven
RPCs to learn what one error string could have said.

## Distinct from the cause bug

`B-landscape-create-hollow-no-components` (judge-filed) fixes the *cause* — make
`landscape.create` build the component grid on 5.7 — and recommends `sculpt`
fail loudly instead of `modifiedVertices:0`. This ticket is the *diagnostic
wording* angle for the broader class: a hollow / component-less / unregistered
landscape can arise from sources other than that one create path (a
manually-spawned `ALandscape`, a partially-loaded streaming proxy, the
pre-existing `Landscape_1` this task also found hollow), and whenever the
extent/component lookup fails, `landscape.edit` (and the loud-failure variant of
`sculpt`) should name the actual condition. Fixing the create bug does not make
this message correct for those other cases.

## Fix (message clarity only — no behavior change required)

**Scope: `landscape.edit` only.** In the `landscape.edit` extent-failure path
(`LandscapeHandler.cpp`, the `Failed to get landscape extent` `SendError` site),
before emitting the bare `INVALID_LANDSCAPE`, check the resolved landscape's
component state — e.g. `ULandscapeInfo::XYtoComponentMap.Num()` / the registered
`ULandscapeComponent` count — and when it is zero, send a diagnostic error that
names the cause and the recovery, for example:

> `[LANDSCAPE_NO_COMPONENTS] Landscape '<name>' has no registered
> ULandscapeComponents (hollow landscape — XYtoComponentMap is empty), so it has
> no editable extent. It was likely spawned without a component grid; recreate
> it via landscape.create or verify the actor before editing.`

The companion `landscape.sculpt` silent-no-op (`modifiedVertices:0,
success:true` on a hollow landscape) is the SAME false-signal problem, but its
loud-failure remediation is already owned by the cause bug
`B-landscape-create-hollow-no-components` (its recommendations require sculpt to
"fail loudly (`INVALID_LANDSCAPE`) rather than silently returning
`modifiedVertices:0` with `success:true`"). To avoid two tickets editing the
same sculpt behavior, this ticket is **deliberately scoped to the
`landscape.edit` extent-error wording only**; the sculpt loud-failure change is
deferred to that cause bug. If the two surfaces should report the
missing-components condition with the identical `LANDSCAPE_NO_COMPONENTS` code,
that alignment belongs in the cause-bug fix, not here.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `landscape` blockout
  task. After `landscape.create` returned `success:true` but spawned a
  component-less landscape on UE 5.7 (the cause is the judge-filed
  `B-landscape-create-hollow-no-components`), `landscape.edit` rejected every
  shaping op with the bare `[INVALID_LANDSCAPE] Failed to get landscape extent`
  — a symptom message, not the cause. Because it named no cause, it drove
  trial-and-error: 5 `landscape.edit` attempts (raise-by-name, raise-by-path,
  raise on a pre-existing `Landscape_1`, a 3x3 `set` heightData, and a retry
  after `landscape.set_material`), 2 `landscape.sculpt` no-ops
  (`modifiedVertices:0, success:true` — the opposite false signal), and 4
  diagnostic calls (`actor.get_bounding_box` all-zero, `actor.get_components` on
  two actors showing `count:1`/zero LandscapeComponents, `actor.describe
  componentClass=LandscapeComponent` → `components:[]`) plus three source-file
  reads (plugin LandscapeHandler.cpp + engine LandscapeEdit.cpp/Landscape.cpp)
  to learn the landscape was hollow. A diagnostic that named the
  zero-ULandscapeComponents / empty-XYtoComponentMap condition would have
  collapsed the whole flail into one read. Distinct PROCESS/ergonomic angle from
  the cause bug (which fixes create + recommends sculpt fail loudly, but does not
  touch the edit-error wording or cover hollow landscapes from other sources).
  Direct sibling of the OPEN `E-level-load-file-not-found-vs-in-memory-orphan`
  (same symptom-not-cause error-wording class, different namespace). Dedup:
  ripgrep across OPEN/closed board files found no existing ticket on the
  `landscape.edit` extent-error wording (qmd unavailable).
- `#2-reword-scope-to-edit-wording-and-fix` `IN-REVIEW` developer — Rescoped
  (REWORD): the original Fix's second half (make `landscape.sculpt` fail loudly
  on the hollow case) duplicated the recommendation already owned by the
  IN-REVIEW cause bug `B-landscape-create-hollow-no-components`. Tightened the
  title/tags/Fix to the `landscape.edit` extent-error WORDING only and
  explicitly deferred the sculpt loud-failure to the cause bug, so the two
  tickets do not both edit sculpt. Implemented the edit-only fix:
  `LandscapeHandler.cpp` (`landscape.edit` / `modify_heightmap` handler) — at the
  `GetLandscapeExtent` failure site (the `Failed to get landscape extent`
  `SendError`), now checks `LandscapeInfo->XYtoComponentMap.Num() == 0` and, when
  the landscape is hollow, emits `[LANDSCAPE_NO_COMPONENTS] Landscape '<name>'
  has no registered ULandscapeComponents (hollow landscape — XYtoComponentMap is
  empty) ...` naming the cause + recovery; the bare `INVALID_LANDSCAPE` remains
  only for the non-hollow extent-failure fallback. Regression test added:
  `Tests/World/TestEnvironmentHandlers.cpp` →
  `EditorAutomationRpcGateway.landscape.edit.HollowEmitsNoComponentsDiagnostic`
  builds a hollow `ALandscape` (bare `SpawnActor<ALandscape>` + `SetLandscapeGuid`
  + `CreateLandscapeInfo`, no `Import` → zero components / empty
  `XYtoComponentMap`), drives the real `landscape.edit` handler through its
  AsyncTask(GameThread) path, and asserts `ErrorCode ==
  LANDSCAPE_NO_COMPONENTS` (and `bSuccess == false`); it fails (reverts to
  `INVALID_LANDSCAPE`) if the diagnostic branch is removed. Files:
  `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/LandscapeHandler.cpp`,
  `Source/EditorAutomationRpcGateway/Private/Tests/World/TestEnvironmentHandlers.cpp`.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
