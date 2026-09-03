---
id: E-material-verbs-have-no-shader-compile-signal
title: "Only material.authoring.compile_material reports shader compile errors; every other material write verb returns success for a material that fails to compile and draws nothing, and none of them mention it"
status: IN-REVIEW
severity: Medium
category: enhancement
tags: [material, compile_mgir, add_custom_expression, connect_nodes, shader-compile, verification, silent-false-success, docs]
encounters: 1
lastSeen: 2026-09-03T02:00:00Z
---

# Material write verbs give no signal that the shader failed

## Symptom

`material.compile_mgir`, `material.graph.*` and `material.authoring.*` all return
success-shaped payloads (`blocksCompiled`, `expressionsCreated`, `nodeId`,
`"Nodes connected."`) that describe the **graph** write. None of them says anything about
whether the material's **shader** compiles. A material with a malformed Custom HLSL node
writes a valid `.uasset`, passes `asset.save`, appears with the right domain, blend mode
and connected `mainInputs` under `material.authoring.get_material_info`, and renders
nothing at all.

`material.authoring.compile_material` is the one verb that surfaces the truth:

```
call("material.authoring.compile_material", {materialPath:"/Game/FPS/UI/Materials/M_HUD_RadarBack"})
-> {"compileSucceeded":false, "compiledWithErrors":true, "compileStatus":"failed",
    "compileErrors":["/Engine/Generated/Material.ush:3746:12: error: use of undeclared identifier 'Input'", ...]}
```

It is excellent — it names the file, line, shader type and permutation. The gap is that
nothing routes an author to it. Its own wiki page is titled as a compile trigger, and
`material.compile_mgir.md` never mentions it, so the natural reading of "compile_mgir"
is that compiling is what it already did.

## Why it matters here

Verifying a material by looking at it costs a PIE session, and on a shared editor a PIE
session costs a world-lock slot behind a queue. `compile_material` answers the same
question in 3-10 seconds with no world, no lock and no PIE — but only if you know to ask.
Two UI materials shipped through a full authoring, saving, disk-verification and PIE
capture cycle before this verb was tried; it found both failures immediately.

## Suggested change

- `material.compile_mgir`, `material.graph.add_expression`/`create_nodes` and
  `material.authoring.connect_nodes` should either return the shader compile status or
  carry a `hint` naming `material.authoring.compile_material`, the way
  `blueprint.inspect` already hints at `asset.dump_folder`.
- Cross-link it from `material.compile_mgir.md` and `material.mgir.md`, and say plainly
  that a successful MGIR compile is a graph write, not a shader compile.
- `visual-review.md` should list "compile the material first" ahead of "capture it",
  since the cheap check strictly dominates the expensive one.

## Fix

Confirmed TRUE against source before changing anything: `MaterialAuthoringHandler.cpp:3629`
(`compile_material`) was the only material verb reading `FMaterialResource::GetCompileErrors()`,
via `MaterialCompileErrorCollector::WaitAndCollect`. `MGIRCompileHandler.cpp:76-117` built
`blocksCompiled` / `expressionsCreated` / `consumerRefresh` with no shader field, and all 12
`material.graph.*` response paths plus the 23 `material.authoring.*` ones ended on a bare
`AddAssetVerification(Result, …)`.

**One shared helper**, `Handlers/Material/MaterialShaderState.h`, publishing one field
`shaderCompile: {status, succeeded, failed, errorCount, errors[], waited, waitedMs,
rendersDefaultMaterial, hint?, materials[]?}`. `status` reuses `compile_material`'s existing five
spellings verbatim (`completed` / `failed` / `outstanding` / `timedOut` / `notCompiled`) so the
namespace has one vocabulary, not two that nearly agree; `outstanding` is the async-compiling case.

Two entry points, deliberately split by cost:

- `Probe()` — non-blocking, reads the verdict already on the `FMaterialResource`. Cheap enough that
  every write verb publishes it unconditionally, which is the point: the defect was a convention
  nobody remembered, so the funnel makes forgetting impossible (`rpc-design.md`: structural
  guarantees over discipline).
- `ProbeAndWait()` — forces `CacheShaders(Synchronous)` + the bounded drain through the existing
  `MaterialCompileErrorCollector`. Opt-in via `waitForShaderCompile` on the two batch entry points
  (`material.compile_mgir`, `material.graph.create_nodes`); `compile_material` remains the
  always-blocking verb. Not wired to a job ticket because `FHandlerContext::StartJob` runs its bind
  delegate on the caller's own stack (`rpc-design.md` §9) — a ticket buys bookkeeping, not deferral,
  and the transport's `wait: false` already gives a caller the non-blocking path.

`notCompiled` is not a pass and the `hint` says so. In a headless editor the permutation jobs are
deferred until the material is first *drawn*, so a material that can never compile probes as
`notCompiled` forever — that is exactly why the wait exists and why the hint routes to
`compile_material`.

For `B-capture-verbs-silent-default-material-fallback`: `RendersDefaultMaterial(const
UMaterialInterface*)` and `shaderCompile.rendersDefaultMaterial` are the probe that ticket needs,
callable with no handler context and no wire parameter. It reads `GetMaterial_Concurrent()` +
`IsDefaultMaterial()` + the resource's compile errors rather than `GetRenderProxy()` — the proxy's
`FMaterial` is render-thread data and reading it from a handler is a race, while both game-thread
facts answer the same question. An instance reports its master's state, which is the correct answer
for the sampler-type case that ticket hit (two very different instances of one broken master).

Files changed:

- `Source/PinWright/Private/Handlers/Material/MaterialShaderState.h` — NEW. The helper, the wire
  block, `PinWright::Material::AddMaterialVerification` (asset verification + shader verdict), and
  the shared `waitForShaderCompile` param spec.
- `Source/PinWright/Private/Handlers/Material/MaterialGraphHandler.cpp` — 12 response sites routed
  through the funnel; `create_nodes` declares and honours `waitForShaderCompile`.
- `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp` — 23 response sites
  routed through the funnel; `compile_material` also publishes the shared block, built from its own
  wait outcome so the two cannot disagree; `get_material_info` publishes the probe; summaries of
  `connect_nodes` and `get_material_info` updated.
- `Source/PinWright/Private/Handlers/Material/MGIRCompileHandler.cpp` — per-asset aggregate over
  `AssetPaths` (worst status wins, `materials[]` breaks it down), `waitForShaderCompile`, and a
  `warnings[]` entry on failure so it survives a caller reading only the top level.
- `Source/PinWright/Private/Tests/Material/TestMaterialShaderStateReport.cpp` — NEW, 3 tests.
- `Docs/wiki-src/material.compile-state.md` — NEW topic page (the field contract, the five statuses,
  why `notCompiled` is not a pass, compile-before-capture).
- `Docs/wiki-src/material.md`, `material.graph.md`, `material.mgir.md`, `material.authoring.md`,
  `visual-review.md` — cross-links; `visual-review` Recipe now has "compile the material first" as
  step 2, ahead of capture, and the black-surface ladder gains step 0.

Not done, and stated rather than hidden: the per-verb `waitForShaderCompile` flag is on the two
batch entry points only. Adding an optional wait parameter to all 35 write verbs would be surface
for its own sake — every one of them publishes the free probe and a `hint` naming the blocking verb.
`material.authoring.set_*_parameter_value` on an instance reports its master's state, which is the
only shader state an instance has.

### Suite fallout, diagnosed and fixed

`PinWright.material.authoring.compile_material.ReportsShaderErrors` went red in
`Saved/Logs/pw_wave_suite.log`. **It is not a lost wire contract and not caused by the rewiring.**
Evidence, all from that log and from source:

- `compiledWithErrors`, `compileErrors`, `compileStatus` and `compileSucceeded` are still set,
  unchanged, at `MaterialAuthoringHandler.cpp:3691-3698`; `MaterialCompileErrorCollector.h` and
  `Utils/AssetCompilePump.h` are byte-identical to HEAD (`git diff --stat` empty).
- The test took **276 s**. It took 3.0 / 2.9 / 4.1 / 2.4 s and passed in the four archived suite
  runs before this one (`PinWrightFullSuite.log`, `Automation_PinWright_savefix.log`,
  `Automation_PinWright_verify2.log`, `PinWrightWave1_6b79f912.log`).
- The log's own shader stats explain the 90x: 10 jobs, `Job execution time: average 120.67 s, max
  256.92 s`, `Average time worker was idle: 24.59 s`, `Effective parallelization: 0.45` against 12
  workers. The ShaderCompileWorker processes were starved by other work on the box.
- `WaitAndCollect` is bounded at `CompileWaitTimeoutSeconds = 90.0`, so it returned `timedOut` with
  an empty error list; the engine wrote the errors at t+271 s
  (`LogShaderCompilers: Warning: Failed to compile Material ...`), long after the assertions ran.
  The remaining time is `CleanupTestAsset`'s force-delete blocking on the in-flight compile.
- Decisive control: the new `PinWright.material.shader_state.BrokenHlslReportsFailedWithTheErrorText`
  drives the SAME broken HLSL through the SAME `WaitAndCollect`, ran 4 minutes later in the same
  editor, and passed in **3.0 s** — the first run had already paid for those shader jobs. Same code
  path, same fixture, 3 s vs 276 s: the variable is host and cache state, not the response shape.

The real defect the run exposed is in the pre-existing test: it asserted a measurement a 90 s-bounded
wait cannot promise, so a contended host turns a correct response into three red assertions — the
`B-tests-host-dependent-fixtures-hard-fail` class. Fixed in
`Source\PinWright\Private\Tests\Material\TestCompileMaterialShaderErrors.cpp` by splitting the
assertions: the wire-contract half (`compiledWithErrors` / `compileErrors` / `compileStatus` present,
`compileSucceeded` false) is asserted on every host, which is what preserves the counterfactual; the
error-text half runs only when a compile LANDED. `timedOut` / `outstanding` report through
`PinWrightTestSkip::SkipAssertions` with the status and `compileWaitedMs` in the message.
`notCompiled` is excused **only** when `GShaderCompilingManager->IsShaderCompilationSkipped()` is
true — otherwise it stays red, because that is the shape a broken collector would produce.

`CompileWaitTimeoutSeconds` was deliberately NOT raised: it is 90 s because the transport's response
timeout is 120 s and the wait must expire first. No handler change is warranted either — the verb
already reports `timedOut` plus a warning telling the caller to call it again, which is the correct
answer to "the compile is still running".

### Reviewer verification

Not compiled and not run — a separate compile pass follows. To verify:

1. Build, then run `PinWright.material.shader_state` (3 tests) plus
   `PinWright.material.authoring.compile_material` (3 existing tests, which the shared-block change
   touches). `BrokenHlslReportsFailedWithTheErrorText`, `ValidMaterialReportsCompleted` and now
   `ReportsShaderErrors` all emit a `PINWRIGHT_ASSERTIONS_SKIPPED` marker on a host that cannot
   finish the platform shader compile inside the bounded wait — a `COMPLETED_WITH_SKIPS` there is
   the fixture reporting honestly, not a regression, and per the plugin's own rule it must not be
   "fixed" by turning the skip into an error. Prefer an idle box for the run: these three are the
   only tests in the suite that block on real shader compilation.
2. Live: author a material with a Custom node holding
   `return GetPrimitiveData(Parameters).LocalToWorld[2].xyz;` driving `EmissiveColor` through
   `material.compile_mgir`, and check the response carries `shaderCompile.status: "failed"` with the
   HLSL text in `shaderCompile.errors` and a `warnings[]` entry — previously it returned
   `blocksCompiled: 1` and nothing else. Then the same document with
   `waitForShaderCompile: false` on a fresh editor should read `notCompiled` with a `hint`, not a
   silent success.
3. Confirm no response-shape regression: `material.graph.*` and `material.authoring.*` responses
   gain exactly one additive `shaderCompile` object; verbs whose subject is a `UMaterialFunction`,
   a landscape layer info object or a parameter collection publish no block at all.
4. `PinWright.infra.wiki_src.SourcePagesFollowRenderingRules` and
   `PinWright.infra.declared_params.*` cover the wiki and parameter-declaration halves.

## History

- `#1-fixed-shader-compile-signal` `IN-REVIEW` developer — Verified TRUE by source reading only (no
  editor was launched). Added `Handlers/Material/MaterialShaderState.h` as the single shared verdict
  and routed all 35 material response sites plus `compile_mgir` and `get_material_info` through it,
  with an opt-in blocking wait on the two batch entry points. Vocabulary reuses
  `compile_material`'s existing `compileStatus` spellings so the namespace has one field and one set
  of values. `rendersDefaultMaterial` is exposed for
  `B-capture-verbs-silent-default-material-fallback` to consume without a handler context. Three
  tests added; not compiled and not run — a separate compile pass follows.
- `#2-compile-material-test-timeout` `IN-REVIEW` developer — Suite run `pw_wave_suite.log` reported
  `compile_material.ReportsShaderErrors` red. Diagnosed from the log and source only (no editor
  launched): NOT a lost wire contract and not caused by this ticket's rewiring — the shader compile
  took 271 s on a starved host against a 90 s bounded wait, so the verb correctly returned
  `compileStatus: "timedOut"` with no errors yet collected. The same fixture through the same code
  path passed in 3.0 s four minutes later in the same editor. Fixed the pre-existing test to assert
  the host-independent wire contract unconditionally and to report the un-measurable branch through
  `PinWrightTestSkip` instead of three red assertions; `notCompiled` stays red unless the engine
  says shader compilation is skipped. See "Suite fallout" above for the full evidence.
