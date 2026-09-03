---
id: B-performance-run-benchmark-measures-nothing
title: "performance.run_benchmark completes a SUCCESSFUL job having measured nothing — on UE 5.8 the whole payload is {captured:false}, and even on an engine that opts the legacy capture back in the verb still returns zero performance quantities, while its two siblings under the identical macro refuse with NOT_SUPPORTED; its declared `type` param is never read at all"
status: IN-REVIEW
severity: High
category: bug
tags: [performance, benchmark, run_benchmark, silent-false-success, jobs, async, stat-file, ue58, not-supported, unread-param, measurement]
encounters: 1
lastSeen: 2026-08-30T17:35:00+03:00
---

# A verb named `run_benchmark` succeeds without benchmarking

`performance.run_benchmark` is registered at `Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp:860`
with the summary **"Start a performance benchmark"**. It starts a job, waits out the caller's
`duration`, and completes it with `OnComplete(true, R, FString())` (`:902`) — a **success**. The
payload `R` is built at `:892-901` and can hold exactly two fields: `captured` (a bool) and
`statFilePath` (a string, only when non-empty). There is no frame time, no FPS, no draw count, no
counter of any kind, on any code path.

**This is not a UE 5.8 regression.** On 5.8 the payload is `{captured:false}` and nothing else. On
an engine rebuilt with the legacy stat-file capture opted back in, the payload becomes
`{captured:true, statFilePath:"….uestats"}` — a file location, still not a measurement. The verb
has never had a branch that reports a performance quantity. What the caller gets in the best case
is "a stat file exists somewhere"; what they asked for is a benchmark.

## Live measurement — running editor, 13:32 build

    performance.run_benchmark {duration: 1, type: "all"}   -> {"captured": false}

No error, no `NOT_SUPPORTED`, no warning. A second call with `type: "this-is-not-a-benchmark-type"`
returns the identical `{"captured": false}` — see § *Second defect* below.

## Source, re-derived at plugin HEAD `ef8a1f1b`

`PINWRIGHT_HAS_STATS_FILE_CAPTURE` (`PerformanceHandler.cpp:27-31`) resolves through
`UE_ENABLE_STATS_FILE_DEPRECATED_IN_5_8`, which `C:/UE_5.8/Engine/Source/Runtime/Core/Public/Stats/StatsFile.h:8-9`
defaults to **0** — so on stock UE 5.8 `'stat startfile'` no longer parses and no capture can run.
The handler knows this. Its own comment at `:886-890` says so and concludes *"report captured:false
instead of fabricating one."*

That reasoning is right about the field and wrong about the envelope. **The two sibling verbs under
the same macro refuse:**

- `performance.start_profiling` (registered `:252`) — `SendError(TEXT("NOT_SUPPORTED"), …)` at `:259-260`,
  its comment at `:256-258` stating that a *"'Profiling started' success here would be fabricated."*
- `performance.stop_profiling` (registered `:283`) — `SendError(TEXT("NOT_SUPPORTED"), …)` at `:289-290`,
  comment `:287-288`: *"'Profiling stopped' would be a fabricated success."*

`run_benchmark` sits between them, reaches the same dead capture, and returns a completed job. The
asymmetry is the defect: the same file reasons about the same dead capture three times, refuses
honestly on two of them, and succeeds on the third.

## Second defect in the same registration: `type` is declared and never read

`RPC_PARAM_OPT("type", "string", "Benchmark type (default 'all')")` is declared at `:863`. The
handler body (`:865-908`) reads `duration` at `:866` and **nothing else** — `grep 'TEXT("type")'`
over the file returns zero hits, so no branch anywhere consumes it. It is not validated either:
the live call above passed `type: "this-is-not-a-benchmark-type"` and got a clean success. A
parameter that is documented, defaulted, accepted, unvalidated and inert tells a caller the verb
has benchmark *modes*, which it does not.

## The `{avgFps, minFps, maxFps, frameCount}` payload does not exist and never did

`B-performance-run-benchmark-no-completion-signal` `#2` says the completion timer *"collects
`{avgFps, minFps, maxFps, frameCount}`"*, and its `#3` treats the gap between that and the observed
`{captured:true}` as *"a minor doc/payload discrepancy worth tracking."* **`avgFps`, `minFps` and
`maxFps` appear nowhere in `Plugins/PinWright/`** — zero hits across `Source/` and `Docs/`. There is
no documentation to reconcile the implementation against; the shape was invented in a history entry
and has been cited since as if it were a contract.

That ticket's `**Files:**` line is also unresolvable: it names
`Source/EditorAutomationRpcGateway/Private/Handlers/Performance/RunBenchmarkHandler.cpp`, and no
`RunBenchmarkHandler.*` exists anywhere under `Source/` (the code has always been in
`Handlers/Debug/PerformanceHandler.cpp`). Recorded here rather than edited into a DONE ticket's
history.

## Fix

Two independent changes, either landable alone:

1. **Refuse, like the siblings.** When `PINWRIGHT_HAS_STATS_FILE_CAPTURE` is 0, fail the job with
   `NOT_SUPPORTED` and the same steer to `insights.start_session` the other two already carry, so
   the success envelope stops asserting something happened. This is a three-line change that makes
   the file internally consistent.
2. **Either measure or rename.** If the verb keeps its name it has to return a performance quantity
   — see `F-performance-frame-time-statistics` for the ask, which is where the real capability
   belongs. If it does not, the registered summary at `:860` should say what it actually does
   ("begin a stat-file capture for `duration` seconds"), and `type` (`:863`) should be dropped or
   implemented.

Do **not** fix this by fabricating numbers from whatever the stats system happens to hold; that
converts a silent false-success into a silent wrong datum, which is worse.

## Severity: High

Impact class **High**, "silent false-success on a normal path (the caller trusts a result that is a
lie and builds on it)". The lie is at the envelope level: the job reads `completed`, the call
returns without error, and the only tell is a `captured` field whose name is about a stat file, not
about whether a benchmark ran. A caller looking for the benchmark's numbers finds no numbers and no
failure, which reads as "the benchmark found nothing to report."

**Critical declined.** Nothing crashes; no asset data is written or lost. The damaged artefact is a
conclusion about performance, which is recoverable once identified.

**Medium declined**, and this is the closest call. Medium is the soft-blocker band — doable via a
documented workaround — and a workaround does exist (`insights.export_trace`, per
`F-performance-frame-time-statistics`). But the workaround is only reachable by a caller who
already knows this verb is broken, and the rubric's High band for silent false-success carries no
"unless a workaround exists" escape: the workaround repairs the *measurement*, not the false
success on this verb. The mitigation that `captured:false` is at least an honest field is real and
is why this is not argued above High.

**Reach modifier declined in both directions.** No bump up: `performance.*` is not an
every-session namespace. No bump down to "rare edge path": a caller reaching for this verb is by
definition trying to measure performance, which is exactly the moment a false success is most
expensive — it is the input to a decision about the level, not a stray call.

## Same shape as

The session's recurring class, stated on `B-foliage-paint-does-no-ground-projection` § *Same shape
as* — with one distinction worth naming, because it is the inverse. In the canonical members
(`B-ground-probe-hits-hull-not-render`, `B-niagara-validate-green-while-component-inactive`,
`B-mrq-render-result-omits-bitrate-and-size`) the call succeeds, every number it reports is correct,
and the output is wrong because the deciding number was never reported. Here there is **no** number
to be wrong: the deciding quantity was never computed at all, and the response's single field is
correct about a question the caller did not ask. Same failure to the caller, one link earlier in the
chain.

## Distinct from

- **`B-performance-run-benchmark-no-completion-signal`** (DONE, Medium) — the async completion
  signal, which its fix delivered and a tester verified. **Deliberately not reopened and its
  `encounters`/`lastSeen` left untouched**: the timer it added fires correctly, and the subject
  here is that the completed job carries no measurement, which is a different defect that its `#3`
  saw and set aside. Reopening a DONE ticket to change what it is about would erase a correct fix's
  record.
- **`F-performance-frame-time-statistics`** (OPEN, Medium, feature) — the missing *capability*: no
  verb in the namespace returns a measured performance quantity. This finding currently lives inside
  that ticket's `#1` for findability only. The two are deliberately split — `F-` asks for a new
  `performance.measure_frames`; this asks the existing verb to stop claiming success. A fixer can
  land either without the other, and the refusal (fix 1 above) is worth landing first because it is
  three lines and stops the bleeding while the capability is designed.
- **`E-performance-run-benchmark-async-poll-undocumented`** (OPEN, Low) — the wiki never teaches the
  ticket → `system.job_status` poll contract. Documentation of the async shape, not of the payload.
- **`E-stop-profiling-no-uestats-path`** — the sibling verb's path readback, on the branch where
  capture *is* compiled in.

## Dedup

Searched the board for `run_benchmark`, `avgFps`, `benchmark`, `captured`, `NOT_SUPPORTED`,
`stat startfile` and every `performance-*` filename. Twelve files mention `run_benchmark`; the four
that are about this verb are distinguished above, and the remaining eight cite it only as an example
of the async-job pattern. **Nothing on the board says a `performance.*` verb reports success
without measuring.**

## Provenance

Live call made against the running editor (13:32 build, reported as plugin commit `d8f1bc32`).
Source re-derived at plugin HEAD `ef8a1f1b` with a clean working tree;
`git diff d8f1bc32..ef8a1f1b -- Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp` is
**empty**, so the source read here is the binary that produced the measurement, verbatim. Engine
citations opened in `C:/UE_5.8`.

## Verification (code review)

**The fix in the tree is correct and complete for what this ticket asks**, and it is committed:
`2df2d8b0` (plugin, 2026-08-31) carries the handler, the tests and the wiki page; every file below is
clean against plugin HEAD `347826a6`. Review was source-only — no editor call, no live run.

**The measurement is a real per-frame frame time, not the RPC's own wall clock.**
`PerformanceHandler.cpp:990` adds a **zero-delay** `FTSTicker` element for the window; `:994-995`
appends one sample per fire (`DeltaTime * 1000` ms) and accumulates the span. That delta is the
engine's frame delta, not ticker bookkeeping:
`C:/UE_5.8/Engine/Source/Runtime/Launch/Private/LaunchEngineLoop.cpp:6103` calls
`FTSTicker::GetCoreTicker().Tick(FApp::GetDeltaTime())` once per frame, and
`Runtime/Core/Private/Containers/Ticker.cpp:121` fires every due element with **that same**
`DeltaTime` (`Element->Fire(DeltaTime)`). One fire = one frame. `measuredDurationSeconds` (`:1034`)
is the **sum of the sampled deltas** — deliberately not an `FPlatformTime::Seconds()` span, which
appears only as the `+5.0 s` runaway backstop at `:1004` — so the published window and the published
samples are on one clock and cannot disagree.

**Quantities published, all from observed frames:** `frameCount` = frames actually sampled (`:1035`);
`frameTimeMs {min, p50, p95, max, mean}` (`:944-948`) with nearest-rank percentiles indexed into the
sorted sample array, no interpolation; `avgFps` = frames / measured span (`:1037`);
`requestedDurationSeconds` named separately from the measured one, with a `warnings` entry when they
differ by more than 10% (`:1047-1062`). `statFileCaptured` (`:1041`) is now plainly a fact about the
`.uestats` side artefact, and the misleading `captured` is gone by name.

**The success-without-measurement envelope is structurally closed.** With no samples or a
non-positive span the job **fails** — `OnComplete(false, nullptr, TEXT("FRAME_TIME_NOT_MEASURED"))`
at `:1028` — and `FJobRegistry::Complete` maps `bSuccess=false` to `status:"failed"` carrying that
error (`State/JobRegistry.cpp:178-181`). A non-positive `duration` never starts a job at all
(`INVALID_ARGUMENT`, `:963`). There is no remaining path that completes a job with no numbers.

**Second defect fixed as filed.** The registration declares `duration` only (`:954`); `type` is gone,
so the dispatcher's unknown-name gate refuses it with `UNKNOWN_PARAMS`
(`Dispatch/RpcDispatcher.cpp:182-198`) rather than accepting it silently. The summary at `:952`
describes what the verb returns.

**Fix 1 of this ticket (refuse like the siblings) was correctly NOT taken.** The refusal was only
right if the legacy stat file were the measurement; it never was. `start_profiling` / `stop_profiling`
exist solely to drive that capture and still refuse (`:259-260`, `:289-290`), which stays correct for
them. The asymmetry this ticket named is resolved in the other direction, which is the stronger one.

**Tests.** `Tests/EditorOps/TestDebugHandlers.cpp:729` (`…ReportsMeasurementOrFails`) drives the real
handler, pumps the core ticker to close the window (`:717`), reads the ticket out of the live
`FPluginState::Get().GetJobRegistry()` and admits exactly two terminal states — completed **with**
`frameCount > 0`, `measuredDurationSeconds > 0`, a separate `requestedDurationSeconds`, five
`frameTimeMs` fields all `> 0`, `avgFps > 0`, no `captured`, `statFileCaptured` present — or failed
**with** a non-empty error. `:830` (`…RejectsNonPositiveDuration`) pins the up-front refusal. Ticker
pumping inside a test is the established pattern here (`Tests/TestUtils.h:447` and eight other TUs),
and `TestUtils.h:19-20` supplies `Containers/Ticker.h` / `HAL/PlatformProcess.h`. Both tests are in
the suite `2df2d8b0` reports green (4818/4818).

**Why history `#4` still saw `{captured:false}` — a stale binary, not a failed fix.** The built module
in this checkout, `Binaries/Win64/UnrealEditor-PinWright.dll` (2026-08-31 19:03), contains the UTF-16
literal `"Start a performance benchmark"` — the pre-fix summary — and contains **none** of
`FRAME_TIME_NOT_MEASURED`, `measuredDurationSeconds`, `statFileCaptured`, or the new summary text. So
the editor answering that probe was running code older than `2df2d8b0` regardless of the DLL's
timestamp. **`#4` does not refute the fix, and it is not re-verified either**: this ticket still owes
one live `performance.run_benchmark {duration: 5}` against a freshly built editor before it can go
`DONE`.

**Caveats, recorded rather than fixed** (none of them re-opens the defect this ticket is about):
- The quantity is **total wall frame time**. No game/render/RHI/GPU split, no draw counts — that is
  `F-performance-frame-time-statistics`, deliberately left OPEN, and `#4`'s `performance.read_stats`
  suggestion belongs there too.
- A frame-rate limiter bounds the answer. Under VSync, `t.MaxFPS`, editor frame-rate smoothing or
  background CPU throttling, `frameTimeMs` reports the **delivered** frame time, which is a real
  measurement of the editor and not a measurement of what the scene costs. The wiki page does not say
  so yet; worth one sentence there when someone next touches it.
- If the engine loop stops ticking entirely the sampler never fires and the job stays `running`
  forever — there is no job-level timeout. Unreachable in practice: the request pump is itself a
  core-ticker element (`PinWrightSubsystem.cpp:195`), so a frozen ticker means no RPCs are served at
  all.
- The first sample is taken on the frame the request arrives: `AddTicker` sets
  `FireTime = CurrentTime` for a zero delay (`Ticker.cpp:14-16`) and `Tick`'s added-elements pump
  inside its `do`/`while` (`Ticker.cpp:139`) fires it in the same pass. That sample is a real frame,
  so this is correct, not a duplicate.
## History
- `#1-successful-job-with-no-measurement` `OPEN` reporter — `performance.run_benchmark` completes a **successful** job whose entire payload is `{captured:false}`. Live-verified twice against the running editor (13:32 build, `d8f1bc32`): `{duration:1, type:"all"}` → `{"captured": false}`, no error and no `NOT_SUPPORTED`. Source re-derived at plugin HEAD `ef8a1f1b`, whose `PerformanceHandler.cpp` is byte-identical to the build's (`git diff d8f1bc32..ef8a1f1b` on that file is empty): registration `:860` summary "Start a performance benchmark"; payload built `:892-901` carrying only `captured` and an optional `statFilePath`; `OnComplete(true, …)` at `:902`. **The strongest form of the claim is engine-independent** — even with `PINWRIGHT_HAS_STATS_FILE_CAPTURE` non-zero the payload becomes `{captured:true, statFilePath:…}`, a file location and still not a performance quantity, so this is not a UE 5.8 regression but a verb that never measured anything. The 5.8 half re-derived to its root: `PINWRIGHT_HAS_STATS_FILE_CAPTURE` (`:27-31`) resolves through `UE_ENABLE_STATS_FILE_DEPRECATED_IN_5_8`, defaulted to 0 at `C:/UE_5.8/.../Stats/StatsFile.h:8-9`, so `'stat startfile'` no longer parses. **The asymmetry that makes it a bug rather than a gap:** the two siblings under the identical macro refuse — `start_profiling` (`:252`) errors `NOT_SUPPORTED` at `:259-260` with a comment calling a success there *"fabricated"*, `stop_profiling` (`:283`) at `:289-290` the same — while `run_benchmark`'s own comment at `:886-890` reasons only about the *field*, not the envelope. **Second defect in the same registration, live-verified:** `RPC_PARAM_OPT("type", …)` at `:863` is never read (zero `TEXT("type")` hits in the file) and never validated — `type:"this-is-not-a-benchmark-type"` returned the same clean `{"captured": false}` — so a documented, defaulted, inert parameter advertises benchmark modes that do not exist. **Third finding, recorded not filed:** the `{avgFps, minFps, maxFps, frameCount}` payload that `B-performance-run-benchmark-no-completion-signal` `#2` claims the timer collects, and `#3` calls "documented", **exists nowhere in the plugin** — zero hits for `avgFps`/`minFps`/`maxFps` across `Source/` and `Docs/`; and that ticket's `**Files:**` line cites `Handlers/Performance/RunBenchmarkHandler.cpp`, which has never existed under `Source/`. Fix asked in two independently-landable parts: refuse with `NOT_SUPPORTED` like the siblings when the capture is compiled out (three lines, stops the false success now), and either measure or rename — with an explicit warning not to fabricate numbers from the stats system, which would trade a false success for a wrong datum. Cross-linked into the session's recurring class with the inversion stated rather than restated: the canonical members report correct numbers and omit the deciding one, whereas here the deciding quantity is never computed. Dedup: searched `run_benchmark`, `avgFps`, `benchmark`, `captured`, `NOT_SUPPORTED`, `stat startfile` and every `performance-*` filename; four adjacent tickets distinguished in § *Distinct from*, and nothing on the board says a `performance.*` verb reports success without measuring. Severity High: the silent-false-success band, with Critical declined (no crash, no asset data lost — the damaged artefact is a conclusion) and Medium declined as the closest call, since the `insights.export_trace` workaround is only reachable by a caller who already knows the verb is broken and repairs the measurement rather than the false success; reach declined in both directions — `performance.*` is not every-session, but a caller here is by definition trying to measure, which is when the lie costs most.
- `#2-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. **Deliberately not rewritten.** The only occurrence of the old root is at body line 73, where this ticket *quotes* the sibling `B-performance-run-benchmark-no-completion-signal`'s unresolvable `**Files:**` path in order to say that no `RunBenchmarkHandler.*` has ever existed. Rewriting it would falsify the observation. That sibling ticket is repointed in this sweep to `Handlers/Debug/PerformanceHandler.cpp:860`, which is the file this ticket names as the real home. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
- `#3-run-benchmark-now-measures-frame-time` `IN-REVIEW` developer — Fixed by MEASURING, not by refusing. The root cause of the 5.8 half is confirmed exactly as filed and re-derived from engine source in this checkout: `C:/UE_5.8/Engine/Source/Runtime/Core/Public/Stats/StatsFile.h:8-9` defaults `UE_ENABLE_STATS_FILE_DEPRECATED_IN_5_8` to 0 and gates the entire header body on it, and `Runtime/Core/Private/Stats/StatsCommand.cpp:2049-2092` gates the `stat startfile`/`stat stopfile` command PARSING on the same macro — so on a stock 5.8 editor (this project's `EngineAssociation` is 5.8 and nothing force-defines the macro) the command does not parse and no `.uestats` can exist. **But the legacy capture was never the measurement** — it produced a FILE for the standalone Profiler GUI — so a `NOT_SUPPORTED` refusal (this ticket's fix 1) would have shipped a permanently dead verb when a live measurement path exists on 5.8. `FEngineLoop::Tick` calls `FTSTicker::GetCoreTicker().Tick(FApp::GetDeltaTime())` once per frame (`LaunchEngineLoop.cpp:6103`) and `FTSTicker::Tick` passes that same delta to every element due that frame (`Ticker.cpp:121`), so a **zero-delay ticker element is a per-frame frame-time sampler** — no stats thread, no engine rebuild, no version gate. `run_benchmark` now samples every frame for `duration` and completes with `requestedDurationSeconds`, `measuredDurationSeconds`, `frameCount`, `avgFps` and `frameTimeMs {min,p50,p95,max,mean}` (nearest-rank percentiles, no interpolation, so every number is a frame that was observed). Requested and measured are separate fields and a `warnings` entry fires when they differ by >10% (whole frames are sampled, so the window overruns by up to one frame). **The success-with-no-measurement envelope is structurally gone**: when nothing was observed the job now calls `OnComplete(false, nullptr, "FRAME_TIME_NOT_MEASURED")` and a non-positive `duration` is refused up front with `INVALID_ARGUMENT` before any job starts. `Ctx.SendUnsupportedEngineVersion` was checked and **rejected as the wrong helper**: it renders "<Feature> requires Unreal Engine <X> or newer (this editor is 5.8)" (`HandlerContext.cpp:557-562`), and the legacy capture needs an engine OLDER than 5.8 or one rebuilt with the deprecated macro on — the message would have been false. The stat file is now a clearly secondary artefact: `statFileCaptured` + `statFilePath` where the engine still compiles it in, and on 5.8 `statFileCaptured:false` plus a warning naming the macro and steering to `insights.start_session`. The misleading `captured` field is gone by name (nothing else in the tree referenced it). **Second defect fixed:** `RPC_PARAM_OPT("type", ...)` is DROPPED rather than invented — the dispatcher's `UNKNOWN_PARAMS` gate now rejects it loudly instead of accepting it silently, and the registered summary was rewritten from "Start a performance benchmark" to describe what the verb actually returns. **Siblings checked, as this ticket asks:** `performance.start_profiling` (`:252`) and `performance.stop_profiling` (`:283`) are NOT silently empty and are NOT the same defect — both take the `#if !PINWRIGHT_HAS_STATS_FILE_CAPTURE` branch and `SendError("NOT_SUPPORTED", …)` at `:259-260` / `:289-290`. That remains correct for them: they exist only to drive the legacy capture, so they have no measurement to fall back on the way a verb named `benchmark` does. Nothing to file. **Regression test** `PinWright.performance.run_benchmark.ReportsMeasurementOrFails` (`Tests/EditorOps/TestDebugHandlers.cpp`) drives the real handler, pumps the core ticker to close the window, reads the job ticket out of the live `FPluginState::Get().GetJobRegistry()` and admits exactly two terminal states — completed WITH measured numbers, or failed WITH a typed error — asserting `frameCount > 0`, `measuredDurationSeconds > 0`, a separate `requestedDurationSeconds`, all five `frameTimeMs` fields > 0, `avgFps > 0`, and that `captured` is gone. **It fails against the pre-fix handler twice over** (no `frameCount`; `captured` still present), which is the required fails-today property. Second test `…RejectsNonPositiveDuration` pins the up-front `INVALID_ARGUMENT`. `check_test_ids.py` re-run clean (4800 ids, no dot-prefix collision). Error-code trap respected: `PerformanceHandler.cpp` is a raw-literal-only file (zero `ErrorCodes::` references) and stays that way — `INVALID_ARGUMENT` is already registered and spelled raw like its neighbours, and `FRAME_TIME_NOT_MEASURED` travels as a job-completion error string, the same shape as `LEVEL_GONE` / `NO_NAV_SYSTEM` / `CREATEPROC_FAILED`, which no emission pattern in `TestErrorCodeRegistry.cpp` scans and none of which is registered. `ErrorCodes.h` deliberately NOT touched. Docs: a `### performance.run_benchmark` section added to `Docs/wiki-src/performance.md` (single `###`, no `##` after it, so all four `infra.wiki_src.SourcePagesFollowRenderingRules` rules hold). **Not compiled and not run** — the orchestrator builds and runs after all agents return. `F-performance-frame-time-statistics` stays OPEN: its ask is broader (thread split, GPU time, draw calls, render resolution, a dedicated `performance.measure_frames`) and none of that is delivered here; what is delivered is that the verb named `benchmark` now returns a benchmark.
- `#4-ue58-still-captured-false-and-no-alternate-route` `IN-REVIEW` reporter — Reproduced on UE 5.8
  during the ENV round-2 critic pass, and adding the finding that there is **no other route to a
  frame-time number through PinWright**, so this ticket currently gates every performance claim any
  stream can make. `performance.run_benchmark {duration: 8}` over
  `/Game/FPS/Maps/FPS_Compound` (1560 actors, 1197 rendered primitives) returned the entire payload
  `{"captured": false}` — no error, no job ticket, no partial numbers, unchanged from this ticket's
  original account. Three fallbacks were then tried and all failed to yield a quantity:
  `performance.show_stats {category:"unit"}` answers only `{"message":"Stat 'unit' toggled"}` — it
  toggles the HUD and returns no data; with `stat unit` toggled on,
  `render.capture_open_level`'s `sceneViewportReadPixels` path does not include the stat overlay
  (already recorded by the ENV builder), and **`editor.screenshot` on the level editor viewport does
  not include it either** — I took one at 1916x1081 with the stat on and the returned PNG carries no
  overlay, so the engine-screenshot path is not a workaround for the scene-readback path;
  `insights.snapshot {}` does succeed and writes a real trace
  (`Saved/Profiling/20260903_065348_365240.utrace`), but nothing in the plugin parses a `.utrace`,
  so the number is on disk and unreadable without UnrealInsights. Net: after four verbs across three
  namespaces the only performance quantity obtainable was `viewDistance.survey.primitives` = 1197
  with 0 distance-culled, which is a scene-complexity count and not a frame time. Two consecutive
  critic reviews of the same map have now had to record "performance unmeasured" for this reason.
  Suggested, in addition to whatever fix is in review: either have `run_benchmark` return the same
  `imageStats`-style measured block the capture verbs do, or expose the stat values as data
  (`performance.read_stats`) rather than only as a HUD toggle, since the HUD is unreachable from
  every capture surface the plugin offers.
- `#5-code-review-verification-stale-binary` `IN-REVIEW` reviewer — Source-only re-verification at plugin HEAD `347826a6`; **status deliberately left `IN-REVIEW`** because no live call was made. The `#3` fix is in the tree and committed (`2df2d8b0`), and it does what a verb named `benchmark` must: a zero-delay `FTSTicker` element samples the engine's own per-frame delta (`LaunchEngineLoop.cpp:6103` -> `Ticker.cpp:121`), and the job completes with `frameCount`, `measuredDurationSeconds` (the SUM of the sampled deltas, not a wall-clock span — wall clock is only the `+5 s` backstop at `:1004`), `avgFps` and nearest-rank `frameTimeMs {min,p50,p95,max,mean}`, or FAILS with `FRAME_TIME_NOT_MEASURED` (`:1028` -> `JobRegistry.cpp:178-181`). `type` is gone from the registration, so `UNKNOWN_PARAMS` now refuses it. Full evidence in § *Verification (code review)*. **The finding that changes what `#4` means:** the built `Binaries/Win64/UnrealEditor-PinWright.dll` in this checkout still carries the pre-fix summary literal `"Start a performance benchmark"` and none of the new field names, so the editor that answered `{captured:false}` on 2026-09-03 was running code older than the fix — `#4` is a stale-binary reading, not a failed fix, and is neither a re-open nor a re-verification. What is still owed before `DONE`: one live `performance.run_benchmark {duration: 5}` against a freshly built editor. Nothing implemented and nothing reverted in this pass; three caveats recorded in the new section (total frame time only, so `F-performance-frame-time-statistics` stays OPEN and owns `#4`'s `performance.read_stats` ask; a frame-rate cap or editor smoothing bounds the reported number, which the wiki page does not yet say; and a frozen engine loop would leave the job `running` with no timeout, unreachable because the request pump is itself a core-ticker element).
