---
id: E-niagara-validate-compile-state-uninitialized-undocumented
title: "niagara.validate surfaces a COMPILE_STATE_UNINITIALIZED info issue on a populated system, but the wiki documents that code only as a dumper artifact — agents must reason from scratch that it's compile-state honesty, not a real validation failure"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, validate, compile-state, docs, discovery, compile-status]
encounters: 6
costly: 1
lastSeen: 2026-09-06T00:31:00+03:00
---

# `niagara.validate` emits `COMPILE_STATE_UNINITIALIZED` with no doc that it's an inherent post-load artifact, not a failure

`niagara.validate` reads cached compile state via the same
`NiagaraDumpBuilder::BuildCompileJson` path the dumper uses (confirmed in
`F-niagara-compile-save-explicit` history #2: *"`niagara.validate` reads cached
compile state via `NiagaraDumpBuilder::BuildCompileJson`; does NOT trigger
compile"*). That builder adds an info-severity `COMPILE_STATE_UNINITIALIZED`
issue whenever any script reports `NCS_Unknown`
(`NiagaraDumpBuilder.cpp:1535-1546` / `:1600-1611` / `:1655-1661`). Under UE 5.6's
default `fx.Niagara.OnDemandCompile=1`, a freshly-loaded or just-edited system
is uncompiled until opened in the asset editor or spawned — so a perfectly
healthy, populated system will return `valid:true` **and** a
`COMPILE_STATE_UNINITIALIZED` info issue from `niagara.validate`.

The problem is purely discovery/interpretation. The dedicated wiki page that
explains this code — `docs/wiki-src/niagara.compile-state.md`, section
*"Compile-state honesty: `compileStatus: null` and `COMPILE_STATE_UNINITIALIZED`"*
— frames the entire behavior as a **dumper** concern:

- *"how the **dumper** reports uninitialised compile state"*
- *"The **dumper** handles this honestly..."*
- *"An info-severity issue `COMPILE_STATE_UNINITIALIZED` is added to the issues
  array on both system and emitter **dump paths**"*

It never says the live `niagara.validate` RPC surfaces the same code in its
`issues[]`. And the `niagara.validate` step in `docs/wiki-src/niagara.md`
(step 3: *"Run `niagara.validate` before relying on the asset"*) does not
cross-link to the compile-state page or warn that an info-severity
`COMPILE_STATE_UNINITIALIZED` is expected and binding/edit-independent after an
edit-then-validate flow on an on-demand-compiled system.

## Why this is friction (process, not a tool bug)

The task (focus `niagara.bind_curve_asset`, outcome **clean** — no tool error,
judge filed nothing) added two user-scope curve DI params to the real
`/Game/ExampleContent/Niagara/Textures/BindCurvesToMaterials`, bound
`2-1_CustomBlendCurve` and `CRV_Rocky`, compiled, then ran
`niagara.validate level:strict` to confirm the bindings resolved with no errors.
Validate returned `valid:true, errors:[]` — but with a non-error
`COMPILE_STATE_UNINITIALIZED` warning. Friction note, verbatim:

> "only friction was that validate emits a non-error COMPILE_STATE_UNINITIALIZED
> warning (emitter scripts report null post-load compile state) that I had to
> confirm is binding-independent vs. a real issue"

That is interpretive friction with no doc support: the agent had to reason out,
from the emitter scripts' null post-load compile state, that this info warning is
an inherent on-demand-compile artifact and not a failure of the curve bindings
it had just made. A less careful agent could read the warning as "the bindings
didn't resolve" and chase a non-bug, or dismiss a genuine future
`COMPILE_STATE_UNINITIALIZED`-adjacent signal as "probably just the compile
thing." 1 validate call, no retry, no tool error — the cost was reasoning, not
calls.

## What it should do

Downstream wiki edit (not mine):

- In `docs/wiki-src/niagara.md`, on the `niagara.validate` step/section, note
  that on a UE 5.6 on-demand-compiled system a successful validate
  (`valid:true`) can still carry an **info-severity** `COMPILE_STATE_UNINITIALIZED`
  issue, that this is the same compile-state-honesty signal the dumper emits
  (cross-link `niagara.compile-state.md`), and that it is independent of the edit
  being validated — to populate real compile state, compile/open the asset first.
- In `docs/wiki-src/niagara.compile-state.md`, broaden the "Compile-state
  honesty" section so the `COMPILE_STATE_UNINITIALIZED` description names
  `niagara.validate` (and `niagara.inspect`'s compile aspect) as live surfaces
  of the code, not just the "dump paths" — since validate reuses
  `BuildCompileJson`, the code is identical in both.

## Distinct from

- `E-niagara-validate-strict-empty-system-undocumented` (OPEN) — same *validate
  docs-gap shape* but a different code and trigger: that ticket is `NO_EMITTERS`
  (and `DISABLED_EMITTER`/`NO_RENDERERS`) escalation on a deliberately-**empty**
  system under `level:strict`. This ticket is `COMPILE_STATE_UNINITIALIZED`
  (info, not escalated) on a **populated** system, driven by on-demand compile,
  at any level. Both want a validate-section doc note; the two notes are
  complementary, not duplicate.
- `B-asset-dump-niagara-compile-state-stale` / `-followup` (DONE),
  `E-asset-dump-niagara-compile-issue-promote-to-meta` (DONE) — those fixed and
  surfaced the code on the `asset.dump` sidecar / `meta.json`. This ticket is
  that the **live `niagara.validate` RPC** emits the same code with no validate-side
  doc steer; it is a docs gap, not a correctness regression of the dumper work.
- `F-niagara-compile-save-explicit` (DONE) — added standalone `niagara.compile`,
  which is the *action* that would populate compile state and clear the issue;
  this ticket is the *documentation* that the issue is expected absent that
  compile and is not a binding failure.

## Evidence

- Friction note (verbatim above): agent had to confirm the
  `COMPILE_STATE_UNINITIALIZED` validate warning was binding-independent. Outcome
  clean; judge filed nothing — this is purely the interpretive/docs friction.
- Source: `NiagaraDumpBuilder.cpp:1535-1546` (the info issue), validate reuse via
  `BuildCompileJson` per `F-niagara-compile-save-explicit` #2; wiki framing in
  `docs/wiki-src/niagara.compile-state.md` (dumper-only) and the un-cross-linked
  validate step in `docs/wiki-src/niagara.md:11`.
- Wiki pages to improve: `docs/wiki-src/niagara.md` (the `niagara.validate`
  section) and `docs/wiki-src/niagara.compile-state.md` (broaden the
  COMPILE_STATE_UNINITIALIZED section to name live RPC surfaces).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the `niagara.bind_curve_asset` task (outcome clean; judge filed nothing). After adding/binding two user-scope curve DIs on the real `BindCurvesToMaterials` system, `niagara.validate level:strict` returned `valid:true, errors:[]` plus a non-error `COMPILE_STATE_UNINITIALIZED` info issue (emitter scripts report null post-load compile state under UE 5.6 `fx.Niagara.OnDemandCompile=1`). The agent had to manually confirm the warning was binding-independent (an on-demand-compile artifact) rather than a real validation failure of the bindings just made — interpretive friction with no doc support (1 validate call, no retry, no tool error). Root cause: `niagara.validate` reuses `NiagaraDumpBuilder::BuildCompileJson` (`NiagaraDumpBuilder.cpp:1535-1546`), which emits `COMPILE_STATE_UNINITIALIZED`, but the only wiki coverage of that code (`niagara.compile-state.md`) frames it as a **dumper**-only artifact and the `niagara.validate` step in `niagara.md` neither cross-links it nor warns the issue is expected/edit-independent. Proposed downstream wiki fix: add a validate-side note in `niagara.md` (a successful validate can still carry an info `COMPILE_STATE_UNINITIALIZED`; cross-link `niagara.compile-state.md`; it's independent of the edit) and broaden the compile-state page to name `niagara.validate`/`niagara.inspect` as live surfaces of the code. Dedup: ripgrep across OPEN/closed (qmd unavailable) found no validate→COMPILE_STATE_UNINITIALIZED docs ticket — `E-niagara-validate-strict-empty-system-undocumented` is the NO_EMITTERS/empty-system variant (different code, different trigger), the three compile-state-stale/meta tickets are the asset.dump path (DONE), and `F-niagara-compile-save-explicit` only mentions validate in passing.
- `#2-second-occurrence-move-renderer-task` `OPEN` reporter — Recurrence on a different clean task (focus `niagara.move_renderer`: build `/Game/VFX/NS_Campfire`, add Sprite/Light/Ribbon renderers, reorder Ribbon to front + Light to last, validate, compile, save — 15 calls, all ok, outcome clean). Same interpretive friction, this time on a *freshly authored* system rather than an edited existing one. Friction note verbatim: "niagara.validate's inner compile.valid:false + COMPILE_STATE_UNINITIALIZED is benign pre-compile noise (top-level errors:[]), resolved by the subsequent compile." The agent ran `niagara.validate level:strict` (which returned `valid:true, errors:[]` at top level) BEFORE the explicit `niagara.compile`, so the inner `compile.valid:false` + `COMPILE_STATE_UNINITIALIZED` was pure pre-compile noise — the agent had to reason that out and noted it was "resolved by the subsequent compile." Reinforces #1: a validate-step doc note (a successful validate can carry an info `COMPILE_STATE_UNINITIALIZED` / `compile.valid:false` until the asset is compiled; cross-link `niagara.compile-state.md`) would have removed the reasoning step. Second task confirming the gap; still Low, still docs-only on `docs/wiki-src/niagara.md` + `docs/wiki-src/niagara.compile-state.md`.
- `#3-additional-nested-valid-collision-ribbon-task` `OPEN` reporter — Third occurrence, and a sharper ERGONOMIC (not just docs) angle on the same nested-vs-authoritative confusion `#2` flagged. Clean ribbon-VFX preview task (focus `effect.create_niagara_ribbon`, namespace `effect`; the agent strict-validated the project's own `DynamicBeam_System` before editing; outcome clean, judge filed nothing). `niagara.validate {level:strict}` returned `outputTooLong` (**13194 chars**, spilled to `Saved/.../HttpResponses`) whose ONLY finding was a single benign `COMPILE_STATE_UNINITIALIZED` warning with `errors:[]` empty — but the payload carried TWO keys literally named `valid` whose booleans DISAGREE: a nested compile-block `valid:false` (line 18) sitting next to the authoritative top-level `valid:true` (line 117). The agent narrated the confusion directly (SAY, verbatim): "The validate output has a nested (non-authoritative) valid:false and a top-level valid. Let me read the structure to confirm the authoritative verdict." — then had to Read an offset of the spilled file to establish the top-level `valid:true` is authoritative: a Grep+Read tax on a zero-error verdict. NEW FIX ANGLE beyond the `#1`/`#2` docs note: the CallAnalyzer proposes an ergonomic RENAME — give the nested compile block a non-colliding key (e.g. `compileValid`) or drop its `valid` entirely so the only `valid` in the payload is the authoritative one, removing the trap for a caller scanning for `valid` without relying on them having read the docs. Also reconfirms the response-size angle: a strict verdict with `errors:[]` + one warning still spilled at 13194 chars, forcing a Read — the validate-side response-size lever tracked at `E-niagara-inspect-no-param-readback-projection` `#3`/`#8`. Severity unchanged (Low — recovers via Read, task succeeded); the two triggers (this compile.valid:false vs the strict warning-vs-error layering in `E-niagara-validate-strict-empty-system-undocumented` `#3`) share the "trust the top-level, not the nested compile block" root.

- `#4-compile-status-unverified-same-root` `OPEN` reporter — Fourth occurrence, and the first on a **different verb**: `niagara.compile_status`. VFX build-06 task on `EAContentExamples58` (UE 5.8), asset-only agent forbidden from opening asset editors. After `niagara.compile {force:true, wait:true}` returned `status:"completed", compiled:true, timedOut:false` (waited 4031 ms / 1420 ms / 1314 ms / 10462 ms / 5229 ms on the five `/Game/FPS/VFX/NS_Impact_*` systems), the mandated follow-up `niagara.compile_status` returned `status:"unverified", completed:false, successful:false, scriptCompileCheck:"unverified", failedScriptCount:0, outstandingCompilationRequests:false, cpuScriptCompilationPending:false` on every one. **A control proves it is not edit-related:** an untouched, already-saved system in the same folder (`/Game/FPS/VFX/NS_Tracer`, last written two days earlier, no edit this session) returns byte-identical `unverified`. Root cause is the same NCS_Unknown state this ticket documents: `niagara.validate {level:strict}` on the same system returns `valid:true, errors:[]` with the single info issue `COMPILE_STATE_UNINITIALIZED`, and its `compile.scripts[]` shows **12 `NCS_UpToDate` + 10 `compileStatus:null`** — the null ten being exactly the per-emitter `EmitterSpawnScript`/`EmitterUpdateScript` pairs (5 emitters x 2). `compile_status`'s `scriptCompileCheck` folds those nulls into `unverified`, so the verb can never report `completed` for a system that has not been opened in an asset editor or spawned. **Why this matters more on `compile_status` than on `validate`:** the wiki's own `niagara.compile_status` page states "`unverified` means the probe cannot prove every script has compiled and **is not a pass**", and the documented Niagara mutation recipe tells callers to prefer `compile_status` over `niagara.compile`'s own verdict — so an agent following the documentation literally concludes every compile in the session failed, on assets that are in fact fine. Suggested fix, in the spirit of `#3`'s rename angle: have `compile_status` distinguish "no compile state recorded (on-demand compile, asset never opened)" from "mixed/failed" — e.g. a distinct `status:"uncompiledOnDemand"` or an explicit `scriptsWithNoCompileState: N` counter — and cross-link `niagara.compile-state.md` from the `niagara.compile_status` wiki page, which currently never mentions `COMPILE_STATE_UNINITIALIZED` or `fx.Niagara.OnDemandCompile`. Severity stays Low (recovers by falling back to `niagara.validate`'s top-level `valid` plus `failedScriptCount:0`), but the affected surface widens from `niagara.validate` to `niagara.compile_status` and the Niagara recipe wiki. Repro on this checkout: `niagara.compile_status {assetPath:"/Game/FPS/VFX/NS_Tracer"}` with no prior edit.

- `#5-material-only-edit-still-unverified` `OPEN` reporter — Fifth occurrence, on `EAContentExamples58` (UE 5.8), and the cleanest control yet for the "is it my edit?" question: the session made **zero Niagara writes**. The task edited only a material instance (`/Game/FPS/VFX/Materials/MI_FPS_Smoke_ExplosionColumn`: `SmokeTint`, `AmbientBoost`, `SoftFadeDistance`) consumed by `NS_Explosion`'s SmokeColumn sprite renderer, then ran the mandated `niagara.validate {level:strict}` on `/Game/FPS/VFX/NS_Explosion`. Result: `valid:true`, `errors:[]`, `dataInterfaceCheck:"consistent"`, `pendingCompile:false` — but `scriptCompileCheck:"unverified"` plus the single `COMPILE_STATE_UNINITIALIZED` warning. `compile.scripts[]` shows **16 `NCS_UpToDate` + 14 `compileStatus:null`**, the nulls being exactly the per-emitter `EmitterSpawnScript`/`EmitterUpdateScript` pairs (7 emitters x 2) — the same signature `#4` measured on the `NS_Impact_*` systems. Because no `niagara.*` mutator ran at all, the `unverified` verdict cannot be attributed to the edit under any reading, which is the confirmation `#4`'s `NS_Tracer` control implied. Cost was the same interpretive tax `#1`-`#4` describe: the agent had to pull `compile.scripts[]` and count the nulls to prove `unverified` was not a regression it had introduced. Also reconfirms `#3`'s response-size angle: a zero-error strict verdict spilled at **16,875 chars** to `Saved/.../HttpResponses`, so establishing "no errors" required an out-of-band file read. No new fix angle beyond `#3`'s nested-`valid` rename and `#4`'s `scriptsWithNoCompileState` counter; filed as an encounter bump.

- `#6-remedy-does-not-terminate-after-completed-compile` `OPEN` reporter — Sixth occurrence, on `EAContentExamples58` (UE 5.8), and the cell `#4`/`#5` leave empty: a **Niagara write plus a compile that reported `completed`**, still `unverified`. Sequence on `/Game/FPS/VFX/NS_Impact_Water`: `niagara.set_module_input` (Column `InitializeParticle > Color`), then `niagara.compile {force:true, wait:true}` → `status:"completed", compiled:true, completed:true, waited:true, waitedMs:560, timedOut:false, stillCompiling:[], outstandingCompilationRequests:false`, then `niagara.validate {level:strict}` → `dataInterfaceCheck:"consistent"`, `pendingCompile:false`, **`scriptCompileCheck:"unverified"`** + `COMPILE_STATE_UNINITIALIZED`. `compile.scripts[]` shows **12 `NCS_UpToDate` + 10 `compileStatus:null`**, the nulls being exactly the per-emitter `EmitterSpawnScript`/`EmitterUpdateScript` pairs (5 emitters x 2) — `#5`'s signature. Re-validating after `asset.save` reproduced it byte-for-byte. **What is new is that this refutes the wiki's own remedy sentence.** `docs/wiki-src/niagara.validate.md` tells the caller, twice, "Run `niagara.compile` and validate again to get a verdict" (once for `dataInterfaceCheck: unverified`, once for `scriptCompileCheck: "unverified"`). The first is true — the DI check went `mismatched` → `consistent` across exactly that compile. The second is not: `scriptCompileCheck` cannot leave `unverified` by compiling, because the ten emitter-level script stubs never carry a terminal status on a system whose emitter spawn/update work is folded into the system scripts. An agent following the page literally compiles, re-validates, sees no change, and has no documented next step. Concrete asks, additive to `#4`'s `scriptsWithNoCompileState: N` counter: (a) exclude scripts the system does not own bytecode for from the `scriptCompileCheck` fold, or report them as a separate `scriptsWithNoCompileState` count so `passed` becomes reachable; (b) until then, delete or qualify the "compile and validate again" remedy on the `scriptCompileCheck` bullet of `niagara.validate.md`, since it names an action that cannot change the field. Cost here was one extra compile+validate round trip and a `compile.scripts[]` read to prove `unverified` was not caused by the edit. Severity unchanged (Low).
