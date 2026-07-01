---
id: E-niagara-inspect-no-param-readback-projection
title: "niagara.inspect has no parameter/projection narrowing — a single-knob default readback dumps all aspects and spills past the 10k threshold, forcing an on-disk Read every verify step"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, niagara-inspect, response-size, oversized, projection, readback, docs]
encounters: 6
lastSeen: 2026-06-25T07:14:35Z
claimedBy: fuzz1
claimedAt: 2026-07-01T07:32:31.3510976+03:00
---

# `niagara.inspect` can't read back one parameter's default — every verify spills to file

`niagara.inspect` exposes only four coarse aspect toggles —
`includeProperties` / `includeStack` / `includeGraphs` / `includeCompile`
(`NiagaraInspectHandler.cpp:207-214`, all default `true`) — and even the
narrowest aspect, `includeProperties`, is documented as "Include **system/emitter,
renderer, and parameter** aspects" (line 210): it bundles the whole user-parameter
list, every renderer, and the system/emitter properties into one payload. There is
**no parameter-name filter, no `fields`/projection, and no "just the user
parameters" mode**. So the extremely common "did the value I just set read back?"
verify — which wants exactly one number for one `User.*` parameter — has no way to
ask for a small response and always pays the full-properties dump, which on a real
system crosses the **10000-char spill threshold** and is written to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing the agent to
**Read/Grep the spilled file** just to confirm one float.

## Why it matters (process cost in this task)

The task (focus `niagara.graph.set_parameter`, tune `NS_SpawnFromIslandNDC`'s
spawn knob) is a set-then-verify loop: add a `User.*` float, set its default, and
**confirm it reads back** — three times (after `add_parameter`, after each
`set_parameter`, after `save`). Every one of those confirmations was a
`niagara.inspect` whose intent was one value (`User.SpawnRateScale == 2.5?`), and
every one overflowed. Friction note, verbatim:

> "inspect dumps overflow the 10k display so every readback required reading the
> on-disk JSON."

Call-log corroboration — **four** `niagara.inspect` calls in the task (the initial
`includeProperties+includeGraphs` survey, plus the post-add, post-set, and
post-save readbacks), each a `User.SpawnRateScale` value check that spilled and had
to be inspected off disk. The spill tax compounds because the whole task is
verify-heavy by design.

## What's wrong

`niagara.inspect`'s only size lever is *aspect-level* (turn whole sections on/off),
and the finest aspect (`includeProperties`) still emits the entire parameter list +
renderers + properties. The dominant readback case — "the value of one named
`User.*` parameter" — has no first-class narrowing, so the natural verify call is
the one that overflows. This is the same shape already accepted for the sibling
verbose readers: a verbose reader with no `limit`/projection that spills on a
perfectly normal asset, forcing an extra Read.

## What it should do

Mirror the fixes already shipped/proposed for the sibling verbose readers:

- Add an optional `parameterName` (and/or `parameterScope`) narrowing so a readback
  of one `User.*` parameter returns just that parameter's name/type/default inline,
  exactly like the "find one bone" / "read back my created volumes" cases in
  `E-skeleton-list-bones-no-limit-spills` and `E-volume-get-info-no-limit-spills`.
- Alternatively/additionally a `fields` / `parametersOnly` projection so
  `includeProperties` can return only the user-parameter list (dropping renderers
  and system/emitter properties), which is the bulk of the bytes on a verify.
- **Docs (`docs/wiki-src/niagara.md`):** note in the `niagara.inspect` section that
  a full inspect of a real system exceeds the inline budget and spills to file, and
  that callers verifying a single parameter default should use the proposed
  `parameterName`/projection to keep the readback inline. Until the narrowing
  lands, document that `includeStack:false includeGraphs:false includeCompile:false`
  is the smallest currently-possible inspect (and that it may still spill).

## Distinct from

- `E-http-response-spill` (DONE) — the *generic* server-side spill mechanism (the
  file-reference fallback itself); this ticket is that a *specific verbose reader*
  (`niagara.inspect`) has no narrowing to stay under the threshold in the first
  place — the same relationship `E-recorder-list-sessions-limit`,
  `E-graph-connections-pagination`, `E-volume-get-info-no-limit-spills`, and
  `E-skeleton-list-bones-no-limit-spills` have to the spill mechanism.
- `E-skeleton-list-bones-no-limit-spills` / `E-volume-get-info-no-limit-spills`
  (OPEN) — identical *shape* (no limit/projection → spill → forced Read) on
  different methods/handlers. Same proposed fix family, different RPC.
- `E-asset-dump-oversized-fields` (DONE) / `F-rpc-property-omit-oversized-opt-in`
  (DONE) — those add a known-huge-UPROPERTY skip to the *dump cache* and to
  `property.get`/`property.list`; they don't touch `niagara.inspect`, and the spill
  here is ordinary parameter/renderer aspect bulk, not a single known-huge property.
- `E-niagara-modify-parameter-no-override-readback` (OPEN) — `niagara.inspect`
  reads asset *defaults* and can't read a runtime *component override*; that is a
  missing read-surface for a different store. This ticket is about the asset-default
  readback (which inspect *does* serve) being too big to read inline.
- `B-niagara-graph-set-parameter-float-clobbered-to-one` (OPEN, judge's filing for
  this task) / `E-niagara-graph-set-parameter-opaque-scope` (OPEN) — the *write*
  defect and its error message; this is purely the *readback response size / Read-tax*
  on the verify path, independent of which setter wrote the value.

## History
- `#6-misleading-index-fixed-in-sibling` `OPEN` developer — The "misleading-`index`" correctness half of `#5` (the `includeStack` `index` not reflecting in-group order) is being fixed under `B-niagara-move-module-noop` (reworded from "move_module no-op" to this exact defect, now IN-REVIEW): `NiagaraDumpBuilder.cpp` `AddGraphStackModules` now orders modules by the ParameterMap execution chain instead of `NodePosY`, so a `move_module`/add reorder is now visible in the `index` field without a `niagara.graph.get` hand-trace. This ticket stays OPEN for the *remaining* asks it tracks: the response-size/spill tax and the missing `emitter` filter + per-group `moduleOrder` projection (a faithful `index` does not shrink the ~500KB whole-system dump).
- `#5-evidence-move-module-no-order-readback` `OPEN` reporter — Fifth independent occurrence and a **sharper sub-angle on the stack/module projection from `#2`** (focus `niagara.move_module`, namespace `niagara`, outcome tool_bug — judge filed the orthogonal `B-niagara-move-module-noop` for the move being a silent no-op; this is the distinct PROCESS angle). The "fix the Particle Update simulation order" task on real `/Game/ExampleContent/EnhancedInput/VFX/Confetti/NS_Confetti` (emitter `ConfettiBurst`) needed to **read back group-relative stack order** to confirm a move/add landed, and `niagara.inspect` offers no faithful order readback at all: (1) it has **no `emitter` filter** — the call-log shows the literal attempt `niagara.inspect {emitter:…}` rejected with `[UNKNOWN_PARAMS] Unknown parameter(s) for 'niagara.inspect': [emitter]. Valid parameters: [assetPath, includeProperties, includeStack, includeGraphs, includeCompile]`, so it **dumps the whole ~500KB system 3x** across the verify steps (exactly the emitter+entryId narrowing `#2` proposes still doesn't exist); and (2) the flat per-node `index` field that `includeStack` emits **does not encode in-group order** — friction note verbatim: *"niagara.inspect's flat per-node 'index' field does NOT reflect in-group stack order (it stayed numerically identical before/after the move and a freshly-added node shows index:0 with posY:0), so confirming group position via inspect alone is ambiguous. I had to fall back to niagara.graph.get and hand-trace the ParameterMap OutputMap->InputMap link chain to authoritatively prove the new top-of-group ordering; a per-group 'moduleOrder' list (or having add_module/move_module's group-relative index surface in inspect) would have made the readback one call instead of a manual graph trace."* So beyond the size tax this ticket already tracks, the **content** of the stack projection is missing a load-bearing field: the proposed emitter+entryId stack projection should also surface a per-group ordered `moduleOrder` (or a group-relative index) so reordering can be verified inline, instead of forcing a `niagara.graph.get` ParameterMap hand-trace. Severity unchanged (Low — works, just spills / lacks projection); strengthens the `#2` stack-projection case and adds the concrete `emitter`-filter rejection + misleading-`index` evidence.
- `#4-evidence-set-module-input-explosion` `OPEN` reporter — Fourth independent occurrence (focus `niagara.set_module_input`, namespace `niagara`, outcome ergo — the judge filed the orthogonal value-correctness issue `E-niagara-inspect-params-stale-after-override`, not this one). This is the actual `SimpleExplosion → /Game/FX/NS_BigExplosion` "punchier blast" task (asset.duplicate → inspect baseline → set_module_input SpawnBurst Spawn Count 80→160 → verify → set InitializeParticle Lifetime Max 2→1 → verify → reset Lifetime → verify → compile → save → final inspect). It is the **most inspect-heavy occurrence yet — seven `niagara.inspect` calls** (baseline survey, params-only spawn verify, stack+graphs override-pin verify, spawn+lifetime verify, lifetime-reset verify, plus the final valid/compile/dirty check), each on the real duplicated system, **every one spilling 1.3 MB–4.5 MB to disk**. The Read-tax here was worse than prior occurrences because no inline Read sufficed: friction note verbatim — *"every niagara.inspect exceeded the 10k display limit and spilled to disk (1.3MB-4.5MB), so I had to parse the JSON files with scripts."* The single-input intents (read back `Spawn Count`, `Lifetime Max` on the `UpwardMeshBurst` emitter) confirm the proposed emitter+entryId stack-projection (per `#2`) would have collapsed each of the seven dumps to one module's inputs/override-pins instead of a multi-MB whole-system spill parsed by hand. Severity unchanged (Low — works, just spills).
- `#2-evidence-clear-module-overrides` `OPEN` reporter — Second independent occurrence, different task/method (focus `niagara.clear_module_overrides`, namespace `niagara`). Same friction, broader than the user-parameter case: the artist-workflow task (inspect Simple_system → set_module_input SpawnRate=42 → set_static_switch → clear_module_overrides → re-inspect) used **five** `niagara.inspect` calls (pre-edit baseline, post-edit confirm, final confirm, plus two more in the wiki-nav prefix) on `/Game/ExampleContent/Niagara/Simple/Simple_system` with `includeStack`/`includeGraphs`/`includeProperties`, and the agent needed to read back one **module's** state (entryId 87F0A1.., override pin SpawnRate.SpawnRate and static switch 'Use Spawn Probability'), not just a user parameter. Friction note verbatim: "every niagara.inspect response exceeded the 10000-char display limit and spilled to a file (~1.2 MB), so I had to Grep/Read the on-disk JSON to locate the module entry, override pin, and static switch each time rather than reading the call result inline." Confirms the narrowing should also cover **stack/module projection** (emitter+entryId → just that module's inputs/overrides/switches), not only `parameterName` — every clear/verify cycle pays the full ~1.2 MB stack+graphs dump to confirm two override pins. Outcome was clean (no tool bug; judge filed nothing), so this is purely the readback Read-tax. Strengthens the case but does not change severity (Low — works, just spills).
- `#3-validate-also-spills-curve-bind-task` `OPEN` reporter — Third independent occurrence (focus `niagara.bind_curve_asset`, namespace `niagara`, outcome clean — no tool bug, judge filed nothing), and it widens the scope: the spill tax is **not unique to `niagara.inspect`** — `niagara.validate` overflows the same way. The curve-binding task on the real `/Game/ExampleContent/Niagara/Textures/BindCurvesToMaterials` ran an initial `niagara.inspect` (props/stack/compile), a post-edit `niagara.validate level:strict`, and a re-inspect with `includeProperties:true` to confirm both new user-store curve DI params resolved with their `CurveAsset` references — and the friction note records both reads spilling: verbatim *"large inspect/validate payloads spilled to disk requiring file reads to verify the user store."* The intent was narrow (confirm `User.IntensityCurve` and `User.TintColorCurve` exist in the user store with `CurveAsset` pointing at `2-1_CustomBlendCurve` / `CRV_Rocky`), but the only available reads dump the full properties/validation payload past the 10000-char threshold, forcing an on-disk Read of each. Confirms the proposed `parameterName`/`parameterScope` narrowing (or a `parametersOnly` projection) should also keep a curve-DI-param readback inline, and that the same response-size lever is needed on `niagara.validate` (whose `issues[]`/properties payload spills on a populated real system) — not only on `niagara.inspect`. Severity unchanged (Low — works, just spills).
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `niagara.graph.set_parameter` tuning task (namespace `niagara.graph`, outcome tool_bug for the float-clobber, which the judge filed as `B-niagara-graph-set-parameter-float-clobbered-to-one`; this is a distinct PROCESS angle). `niagara.inspect` has only four coarse aspect toggles (`NiagaraInspectHandler.cpp:207-214`, all default true) and the finest one, `includeProperties`, bundles "system/emitter, renderer, and parameter aspects" (line 210) with no `parameterName` filter, no `fields`/projection, and no parameters-only mode. The task is a set-then-verify loop whose three readbacks (post-add, post-set, post-save) plus the initial survey each wanted one value (`User.SpawnRateScale == 2.5?`) but each spilled past the 10000-char threshold to `Saved/EditorAutomation/HttpResponses/...`, forcing an on-disk Read every verify step — friction note verbatim: "inspect dumps overflow the 10k display so every readback required reading the on-disk JSON" (4 niagara.inspect calls in the task). Proposes a `parameterName`/`parameterScope` narrowing and/or a `fields`/`parametersOnly` projection (per `E-skeleton-list-bones-no-limit-spills` / `E-volume-get-info-no-limit-spills`), plus a `docs/wiki-src/niagara.md` note that a full inspect spills and how to keep a single-param readback inline. Ripgrep across OPEN/closed found no existing ticket on `niagara.inspect` output spilling or lacking a parameter/projection narrowing (`F-rpc-niagara-inspect-standalone-script` is about accepting script assets; `E-niagara-modify-parameter-no-override-readback` is a missing component-override read surface — different gaps).
</content>
