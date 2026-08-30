---
id: E-lighting-set-ao-exposure-no-echo
title: "lighting.set_ambient_occlusion / set_exposure success responses echo only {success, actorName} + actor verification, none of the AO/exposure values they just wrote — forces a property.get round-trip on FPostProcessSettings to confirm the write"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [lighting, set-ambient-occlusion, set-exposure, post-process, readback, round-trip, response-shape, echo, consistency]
---

# `lighting.set_*` PostProcessVolume setters don't echo the values they wrote, so confirming the AO/exposure write costs a separate `property.get` on `FPostProcessSettings`

`lighting.set_ambient_occlusion` writes up to three `FPostProcessSettings`
fields onto a find-or-spawned PostProcessVolume — `AmbientOcclusionIntensity`
(+ its `bOverride_` flag) and `AmbientOcclusionRadius` (+ its `bOverride_`) —
but its success response carries **none of them**. It returns only
`{"success":true,"actorName":"PostProcessVolume0"}` plus whatever
`AddActorVerification(Resp, PPV)` appends (the actor identity), discarding the
exact values the handler set one line earlier. The sibling
`lighting.set_exposure` has the identical shape: it writes
`AutoExposureMinBrightness` / `AutoExposureMaxBrightness` / `AutoExposureBias`
and echoes only `{success, actorName}` + actor verification. A caller who set
AO `intensity:0.6, radius:120` and wants a true round-trip confirm that the
write landed (and that the paired `bOverride_*` flag flipped — the silent
foot-gun documented in `F-post-process-typed-setters`) cannot get it from the
setter's own response; it must read the values back off the volume.

This is the same "the mutator already holds the answer but doesn't carry it,
so a working readback verb gets spammed" response-shape family as
`E-set-transition-settings-no-echo` (OPEN), `E-geometry-deformer-echo-mesh-counts`
(OPEN), `E-create-procedural-terrain-no-material-echo` (OPEN), and
`E-niagara-modify-parameter-no-override-readback` (OPEN) — here the omitted echo
is the **applied AO/exposure values** on a `lighting.set_*` setter, a distinct
method not covered by any of those tickets. Each member is filed per-method
because the fix is per-handler-response (echo the values the handler already
holds), not a shared util.

## Source (handler confirmed)

`Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp` (the module
was renamed `EditorAutomationRpcGateway` → `PinWright`; same code):

- `lighting.set_ambient_occlusion` (:777) sets
  `Settings.AmbientOcclusionIntensity` / `Settings.AmbientOcclusionRadius` and
  their `bOverride_*` flags (:792-808), then the response (:810-814) is only:
  ```cpp
  Resp->SetBoolField(TEXT("success"), true);
  Resp->SetStringField(TEXT("actorName"), PPV->GetActorLabel());
  AddActorVerification(Resp, PPV);
  ```
  No `intensity` / `radius` / override-flag echo, although both values are in
  hand at this point.
- `lighting.set_exposure` (:746) is identical: writes
  `AutoExposureMinBrightness`/`MaxBrightness`/`AutoExposureBias` (:760-766),
  response (:768-772) echoes only `{success, actorName}` + verification.

The values needed to confirm the write are exactly the
`FPostProcessSettings.AmbientOcclusionIntensity` / `AmbientOcclusionRadius` /
`bOverride_AmbientOcclusionIntensity` fields that the agent had to read via
`property.get` — i.e. the answer was on the same struct the handler already held.

## Friction evidence (this task — focus `lighting.set_ambient_occlusion`, moody-interior build, outcome clean)

The story's step 9 *explicitly* required a true round-trip readback: "read back
the result to confirm the ambient occlusion settings actually took … verify
enabled is true with the intensity/radius we asked for." Because
`set_ambient_occlusion` echoes none of the values it wrote, the agent had to
fall back to three `property.get` calls on `PostProcessVolume0` —
`Settings.AmbientOcclusionIntensity` (=0.6), `Settings.AmbientOcclusionRadius`
(=120), and `Settings.bOverride_AmbientOcclusionIntensity` (=true) — before
re-calling the setter to "re-confirm." Friction note (verbatim):

> "lighting.set_ambient_occlusion returns only the actor identity, not the
> written AO values, so a true round-trip readback required falling back to
> property.get on FPostProcessSettings fields
> (AmbientOcclusionIntensity/Radius/bOverride_) — the AO setter has no
> values-echoing or read verb of its own, which is a small
> discoverability/verification gap."

Call log: the three `property.get` calls (PPV0 AmbientOcclusionIntensity /
AmbientOcclusionRadius / bOverride_AmbientOcclusionIntensity) are the recovery
round-trip a values-echo in the `set_ambient_occlusion` response would have
eliminated. All calls succeeded — this is pure PROCESS overhead, not an outcome
bug; the seed method landed `clean` in the ledger and the judge filed nothing
(`filed_id` empty).

## What it should do

Have `lighting.set_ambient_occlusion` and `lighting.set_exposure` echo the
values they actually applied in their success responses, only for the params
that were present in the call (both are optional-param "update settings" RPCs):

- `set_ambient_occlusion` → echo `intensity` and `radius` (numbers) and, since
  the `bOverride_*` foot-gun is the load-bearing thing a verifier wants to
  confirm, an `enabled`/override indicator. The handler holds
  `Settings.AmbientOcclusionIntensity` / `AmbientOcclusionRadius` at
  response-build time.
- `set_exposure` → echo `minBrightness` / `maxBrightness` / `compensationValue`
  from the `Settings.AutoExposure*` fields it just wrote.

Cheap: a few `SetNumberField`/`SetBoolField` calls on the `Resp` object the
handler already builds, removing the `property.get` round-trip entirely. Mirrors
how `subdivide`/`simplify_mesh` already echo post-op counts and how
`E-set-transition-settings-no-echo` argues the applied values belong inline on
the setter's own response.

**Workaround:** confirm an AO write via `property.get` on the PostProcessVolume:
`Settings.AmbientOcclusionIntensity`, `Settings.AmbientOcclusionRadius`,
`Settings.bOverride_AmbientOcclusionIntensity` (and the `AutoExposure*` fields
for exposure). The override flag must be read as a separate leaf field; the
setter gives no echo of it.

## Not a duplicate of

- `F-post-process-typed-setters` (DONE) — that ticket *added* the typed PPV
  setters and used `lighting.set_ambient_occlusion` as the pattern *model*; it
  explicitly listed "Reading current values back; `asset.dump` … covers that"
  as **out of scope**. This ticket is exactly that deferred readback/echo gap,
  on the lighting setters specifically, and proposes the cheaper response-echo
  fix rather than an `asset.dump`.
- `E-set-transition-settings-no-echo` / `E-geometry-deformer-echo-mesh-counts` /
  `E-create-procedural-terrain-no-material-echo` /
  `E-niagara-modify-parameter-no-override-readback` — same response-echo family,
  different methods; none touches the `lighting.set_*` PPV setters.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `lighting.set_ambient_occlusion` moody-interior task (17 calls, outcome clean,
  judge filed nothing — `filed_id` empty). PROCESS finding: `set_ambient_occlusion`
  (`LightingHandler.cpp:776`) writes `AmbientOcclusionIntensity`/`Radius` + their
  `bOverride_*` flags (:792-808) but its success response (:810-814) echoes only
  `{success, actorName}` + `AddActorVerification` — none of the values it set,
  all of which it holds at response-build time; `lighting.set_exposure` (:745) is
  identical for the `AutoExposure*` fields. The story's step-9 round-trip confirm
  ("verify enabled is true with the intensity/radius we asked for") therefore
  forced three `property.get` calls on `PostProcessVolume0`
  (`Settings.AmbientOcclusionIntensity`=0.6, `AmbientOcclusionRadius`=120,
  `bOverride_AmbientOcclusionIntensity`=true). Friction note (verbatim above):
  "the AO setter has no values-echoing or read verb of its own". Same
  "mutator should carry the answer" shape as `E-set-transition-settings-no-echo`
  and `E-geometry-deformer-echo-mesh-counts`; the readback/echo gap that
  `F-post-process-typed-setters` (DONE) explicitly deferred as out of scope. Fix:
  echo the applied AO/exposure values (and an override indicator) in the setter's
  own response. Workaround: `property.get` the `FPostProcessSettings` leaf fields.
  Deduped via ripgrep across OPEN/closed: no existing ticket covers the
  `lighting.set_*` setter response-echo angle. Severity Low (recoverable via
  `property.get`; never blocks). Page note for the docs-first half: the
  `docs/wiki-src/lighting.md` overlay could state that `set_ambient_occlusion` /
  `set_exposure` don't echo values and name the `property.get`
  `Settings.AmbientOcclusion*` / `Settings.AutoExposure*` confirm path, but the
  primary fix here is the response echo, not a doc line.
- `#2-retriage` `OPEN` triage — Low→Medium: setter readback omits applied AO/exposure values, forcing a property.get round-trip on a rare lighting-setup path.
- `#3-reword-and-fix` `IN-REVIEW` developer — Reworded then implemented. **Reword:** severity Medium→**Low** — the `#2` retriage cited the rubric's "a readback omits a field and forces a fallback" (README.md:158), but that clause is for READBACK verbs; these are SETTERS on a *rare* lighting-setup path (the reporter's own `#1` rated Low, "recoverable via property.get; never blocks"). Under the impact×reach rubric a rare-path response-spill that never blocks is Low with the rare-path down-modifier, consistent with every IN-REVIEW sibling in this no-echo family. Also fixed the stale `Source/EditorAutomationRpcGateway/...` path prefix (module renamed to `PinWright`) and the off-by-one register line numbers (set_exposure :746, set_ambient_occlusion :777). **Fix:** `lighting.set_exposure` and `lighting.set_ambient_occlusion` now echo the values they applied in their success responses, only for params present in the call — `set_exposure` adds `minBrightness`/`maxBrightness`/`compensationValue` from the `AutoExposure*` fields; `set_ambient_occlusion` adds `intensity`/`radius` plus `enabled` (the `bOverride_AmbientOcclusionIntensity` state, the load-bearing foot-gun a verifier wants), all read off `PPV->Settings.*` the handler already holds at response-build time. Removes the `property.get` round-trip. File: `Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp`. **Test:** added `FLightingSetAmbientOcclusionEchoesAppliedValuesTest` and `FLightingSetExposureEchoesAppliedValuesTest` in `Source/PinWright/Private/Tests/Environment/TestPostProcessHandlers.cpp` — each invokes the real registered handler via `InvokeHandlerWithCapture` and asserts the response echoes the applied intensity/radius/enabled (resp. min/max/compensation); they fail if the echo is reverted to `{success, actorName}` + verification only.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
