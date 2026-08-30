---
id: B-performance-run-benchmark-measures-nothing
title: "performance.run_benchmark completes a SUCCESSFUL job having measured nothing — on UE 5.8 the whole payload is {captured:false}, and even on an engine that opts the legacy capture back in the verb still returns zero performance quantities, while its two siblings under the identical macro refuse with NOT_SUPPORTED; its declared `type` param is never read at all"
status: OPEN
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

## History
- `#1-successful-job-with-no-measurement` `OPEN` reporter — `performance.run_benchmark` completes a **successful** job whose entire payload is `{captured:false}`. Live-verified twice against the running editor (13:32 build, `d8f1bc32`): `{duration:1, type:"all"}` → `{"captured": false}`, no error and no `NOT_SUPPORTED`. Source re-derived at plugin HEAD `ef8a1f1b`, whose `PerformanceHandler.cpp` is byte-identical to the build's (`git diff d8f1bc32..ef8a1f1b` on that file is empty): registration `:860` summary "Start a performance benchmark"; payload built `:892-901` carrying only `captured` and an optional `statFilePath`; `OnComplete(true, …)` at `:902`. **The strongest form of the claim is engine-independent** — even with `PINWRIGHT_HAS_STATS_FILE_CAPTURE` non-zero the payload becomes `{captured:true, statFilePath:…}`, a file location and still not a performance quantity, so this is not a UE 5.8 regression but a verb that never measured anything. The 5.8 half re-derived to its root: `PINWRIGHT_HAS_STATS_FILE_CAPTURE` (`:27-31`) resolves through `UE_ENABLE_STATS_FILE_DEPRECATED_IN_5_8`, defaulted to 0 at `C:/UE_5.8/.../Stats/StatsFile.h:8-9`, so `'stat startfile'` no longer parses. **The asymmetry that makes it a bug rather than a gap:** the two siblings under the identical macro refuse — `start_profiling` (`:252`) errors `NOT_SUPPORTED` at `:259-260` with a comment calling a success there *"fabricated"*, `stop_profiling` (`:283`) at `:289-290` the same — while `run_benchmark`'s own comment at `:886-890` reasons only about the *field*, not the envelope. **Second defect in the same registration, live-verified:** `RPC_PARAM_OPT("type", …)` at `:863` is never read (zero `TEXT("type")` hits in the file) and never validated — `type:"this-is-not-a-benchmark-type"` returned the same clean `{"captured": false}` — so a documented, defaulted, inert parameter advertises benchmark modes that do not exist. **Third finding, recorded not filed:** the `{avgFps, minFps, maxFps, frameCount}` payload that `B-performance-run-benchmark-no-completion-signal` `#2` claims the timer collects, and `#3` calls "documented", **exists nowhere in the plugin** — zero hits for `avgFps`/`minFps`/`maxFps` across `Source/` and `Docs/`; and that ticket's `**Files:**` line cites `Handlers/Performance/RunBenchmarkHandler.cpp`, which has never existed under `Source/`. Fix asked in two independently-landable parts: refuse with `NOT_SUPPORTED` like the siblings when the capture is compiled out (three lines, stops the false success now), and either measure or rename — with an explicit warning not to fabricate numbers from the stats system, which would trade a false success for a wrong datum. Cross-linked into the session's recurring class with the inversion stated rather than restated: the canonical members report correct numbers and omit the deciding one, whereas here the deciding quantity is never computed. Dedup: searched `run_benchmark`, `avgFps`, `benchmark`, `captured`, `NOT_SUPPORTED`, `stat startfile` and every `performance-*` filename; four adjacent tickets distinguished in § *Distinct from*, and nothing on the board says a `performance.*` verb reports success without measuring. Severity High: the silent-false-success band, with Critical declined (no crash, no asset data lost — the damaged artefact is a conclusion) and Medium declined as the closest call, since the `insights.export_trace` workaround is only reachable by a caller who already knows the verb is broken and repairs the measurement rather than the false success; reach declined in both directions — `performance.*` is not every-session, but a caller here is by definition trying to measure, which is when the lie costs most.
