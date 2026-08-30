---
id: E-module-rename-citation-sweep
title: "515 board citations under the retired EditorAutomationRpcGateway module root, swept"
status: DONE
severity: Low
category: ergonomic
tags: [citations, maintenance, module-rename, board-hygiene, pinwright]
---

# 515 board citations under the retired `EditorAutomationRpcGateway` module root, swept

Record of the sweep flagged but not done by `E-propertyutils-split-by-concern` `#6`
(*"Not swept, and much larger: `Source/EditorAutomationRpcGateway/` appears in 276 board files
across 516 citations … That one wants its own decision rather than being folded in here"*). This
is that decision and its result. **No ticket's claim or status was changed by the sweep itself**;
what it *found* is a separate matter and is listed at the end.

## The mapping is not one-to-one

The obvious rule — `Source/EditorAutomationRpcGateway/` → `Source/PinWright/`, plugin commit
`8962f163`, plus `Source/EditorAutomationRpcGatewayTests/` folded into
`Source/PinWright/Private/Tests/` — covers most of it: **436 of 515** citations resolve by that rule
alone, **43 more only through the exception table below**, and **36 not mechanically at all**. Every
one of the 479 that resolves was confirmed to exist at plugin HEAD `ef8a1f1b`. Derived from the plugin's
own rename graph (`git log --diff-filter=R -M` across `8962f163^..HEAD`), not from the shape of
the string. **21 distinct paths move by more than the prefix**, in three kinds:

- **Left the module entirely** — plugin `b02e4fc7` split optional engine-plugin integrations into
  gated sub-modules. `Handlers/Geometry/{GeometryTransformHandler,MeshOpsHandler,PrimitiveHandler}.cpp`
  and three geometry tests → `Source/PinWrightGeometry/`; `Handlers/PCG/` →
  `Source/PinWrightPCG/Private/Handlers/PCG/`; `Handlers/Chooser/` and its test →
  `Source/PinWrightChooser/`. A prefix rewrite on these produces a path that does **not** exist.
  `Handlers/Geometry/` is the trap: the directory still exists in the main module but holds only
  `SplineHandler.cpp` / `SplineHelpers.h`, so a directory-level rule would be wrong.
- **Renamed with the module** — `Private/EditorAutomationRpcGatewaySubsystem.cpp` →
  `Private/PinWrightSubsystem.cpp`; `Public/EditorAutomationRpcGatewaySettings.h` →
  `Public/PinWrightSettings.h`.
- **Moved inside the module** — `Handlers/UI/SCSTextEmitter.{h,cpp}` → `Handlers/Blueprint/`;
  `Private/IrCore/IrTextUtils.h` → `Public/IrCore/`; `State/JobMonitorLog.h` → `Utils/`;
  `Test/AssetDumpHandlerInternal.h` → `Handlers/Asset/`; `Utils/NiagaraGraphResetUtils.{h,cpp}`
  and `Utils/NiagaraInstanceUtils.{h,cpp}` → `Handlers/Niagara/`; `Public/Handlers/HandlerContext.h`
  → `Private/Handlers/`.

Checked and clean: there is **no** path where the prefix rule resolves to an existing file while
the rename graph points somewhere else — so wherever the prefix rule lands on a real file, it is
the right file.

## What was done

- **173 citations sit in ticket bodies. 165 were repointed in place**; each rewritten path was
  confirmed to exist at plugin HEAD `ef8a1f1b`. Two carried a **doubled** stale prefix
  (`Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/…`) and needed both
  segments rewritten — a module-segment-only rewrite would have left them broken.
- **342 citations sit in `## History` rows and were not touched**, per the append-only rule. Each
  affected ticket instead carries a new `#N-repoint-citations-after-module-rename` row giving the
  map, and naming any exception, corrected line, or broken claim on that ticket.
- **8 body citations were deliberately left as filed**, because rewriting them would make an
  unresolvable path look resolvable — or destroy the sentence:
  - `B-extract-vector-field-narrows-world-coords-to-float32` `:179` and
    `B-performance-run-benchmark-measures-nothing` `:73` **quote** the stale path as an
    observation about a sibling ticket. Rewriting would falsify the observation.
  - `E-propertyutils-split-by-concern` `:12`, `:68` and `E-animation-authoring-handler-split` `:12`
    cite the pre-split file that is each ticket's **own subject**.
  - `B-pipeline-run-ubt-no-completion-signal` `:16` and `F-test-progress-protocol-removed` `:16`
    name files that never existed / were deliberately deleted; a `Source/PinWright/…` spelling
    would be equally nonexistent and more misleading.
  - `B-sequencer-create-dangling-else-hangs` `:34` now lands on **innocent compiling code** — the
    failure mode `E-propertyutils-split-by-concern` `#6` called worse than a deleted path.
- **A repointed path is worth nothing if the line is wrong.** Only **41** of the 515 carry a
  `file:line`; every one was read at HEAD against the claim it supports. **12 land** (drift ≤ 5
  lines), **18 had drifted onto unrelated code and were corrected**, and **11 name a construct
  that no longer exists**. The 474 without a line number were verified by path existence only,
  and their rows say so.
- **Not swept, measured instead.** The rename travelled past file paths into symbols this sweep
  did not touch: automation-test ids (`EditorAutomationRpcGateway.<ns>.<Test>` → `PinWright.*`,
  ~40 occurrences), the API and feature-guard macros (`EDITORAUTOMATIONRPCGATEWAY_API` →
  `PINWRIGHT_API`, already recorded on `E-propertyutils-split-by-concern` `#6`), and bare
  basenames outside a path (`EditorAutomationRpcGatewaySubsystem.cpp`,
  `EditorAutomationRpcGatewayHelpers.h`, `EditorAutomationRpcGateway_BlueprintCreationShim.cpp` —
  the last has no successor at HEAD). Two basenames renamed by the same commit *were* swept where
  they appear beside a repointed path, both verified present at HEAD:
  `EditorAutomationRpcGateway.Build.cs` → `PinWright.Build.cs` and
  `EditorAutomationRpcGateway_SCSHandlers` / `_BlueprintHandlers_List` → `PinWright_*`.
- **Commit shape: one commit per ticket**, per `README.md` (*"the same one-file-per-commit
  convention keeps `git blame` per-ticket useful"*) and matching the immediate precedent — the
  eleven-ticket `PropertyUtils` repoint was one commit each. The board's history does carry
  grouped audit sweeps, but nothing nearer in kind than that cluster.

## Citation health: the 96% figure was never about these

A prior pass reported board citation health at **96%**. **That figure does not cover this class
and never did.** There is no board citation checker — the board repo contains no scripts, and the
only citation gate in the tree, `Docs/tools/check_hazards.py`, is hard-rooted at the host map
project (`:43-44`, `:2308`) and covers `Docs/scripts/*.py` only; `F-citation-checker-repo-wide`
(OPEN) says as much. The 96% came from a 25-citation hand-check whose own scope was **engine**
citations into `C:/UE_5.8/Engine/...`; the word "engine" was lost across four relay hops. Its
mechanical companion (99.92%, 1280/1281) was **path-blind by construction**: it tokenised
`basename:line` with a regex that cannot capture a directory, and *skipped* any basename found
under `Plugins/PinWright`. Of the 515, **192 distinct basenames were skipped as plugin files, 27
more had no engine match, and 478 carry no line number at all** — every one scored neither pass
nor fail.

Re-derived directly, resolving every path-bearing citation on the board against the plugin tree,
the host project and the engine (excluding generated roots, elided `.../` paths, and paths rooted
in other checkouts): **5,823 of 6,975 resolve — 83.5%**, with 1,152 failures across 499 ticket
files. A path-blind checker scores the same board at 90.4%. **The real figure was 83.5%, not
96%**, and this cluster alone accounted for 6.9 points of the gap. The next-largest failing
clusters are unrelated and want their own look: ~48 `Source/PinWright/...` paths that no longer
exist, 29 `Utils/PropertyUtils.cpp`, and 67 `docs/wiki/*` (now `Docs/wiki-src/`).

## What the sweep found that is not a stale path

**28 tickets carry a claim that no longer holds at HEAD.** Each is recorded on its own ticket's
repoint row; no status was changed. The ones that matter most:

- **A live false promise in the shipped wiki, three times.** `system.run_tests`' registration
  summary (`SystemControlHandler.cpp:477`) advertises "report pass/fail counts" while
  `MakeRunTestsResult` (`:145-156`) returns none; `system.run_ubt`'s (`:366`) advertises "capture
  stdout/stderr" while `ProcPollBind.h:37-38` sets `{exit_code}` alone and `:448` comments that it
  polls "no output pipe"; `system.inspect.inspect_class`'s (`EnvironmentHandler.cpp:1632`)
  advertises only "name, full path, and parent class" while the handler emits `functions[]`
  (`:1681`) and `properties[]` (`:1693`). Filed as `B-registration-summaries-promise-absent-fields`.
- **`B-editor-save-all-no-completion-signal` is DONE on a fix that was undone.** `editor.save_all`
  is fully synchronous at HEAD; the async job envelope its body and `#3` describe is gone.
- **`B-pipeline-run-ubt-no-completion-signal` is DONE on a verb that no longer exists.**
  `pipeline.run_ubt` has zero registrations; `Handlers/BuildTools/UbtEntryPoint.h:14` calls it
  "since-removed".
- **`F-job-control-rpcs` documents the retention semantics backwards** — completed tickets are
  retained until a TTL, not evicted on completion, and the error code is `ERR_TICKET_NOT_FOUND`.
  A caller following it would read a completed job's real payload as an error.
- **`E-dump-rpc-parity`'s `data_table` gap is reopened in fact but not on the board** —
  `data_table.describe`, one of the four RPCs `#3` reports shipping, was culled in `e0d0fe2c`.
- **`B-level-build-all-no-completion-signal`'s payload was invented** — the body promises
  `{lightingOk, navigationOk}`; the completion payload is an empty `FJsonObject`.
- **Six IN-REVIEW tickets describe defects already fixed in source** —
  `B-blocking-volume-no-brush-geometry`, `B-create-datalayer-transient-asset-not-persistable`,
  `B-sequencer-create-dangling-else-hangs`, `E-describe-sound-wave-omits-compression`,
  `B-sequence-add-keyframe-location-property-rejected`, `F-no-response-handler-guardrail`.
- **`F-agir-interface-function-decls` has no test coverage at HEAD** — the round-trip test its `#2`
  added was retired in `c827caf6` for depending on Lyra content; the shipped code
  (`AGIRCompiler.cpp:1638`) is now unguarded.
- **Two citations name something that never existed.** `B-asset-dump-bpir-stub-on-graphless-classes`
  quotes the marker `# (empty: no event/function bodies)`, which greps to nothing (HEAD emits
  `# (graph has zero nodes)`); `B-bpir-bind-dispatcher-external-target-local-event` `#6` credits
  `FCodePinResolver::ConvertCppTypeToPinType`, which exists nowhere outside tests.
- **17 cited handler basenames never existed in the plugin's visible history at all** — the
  `*-no-completion-signal` family names a per-verb file (`SaveLevelHandler.cpp`,
  `BuildAllHandler.cpp`, `RunTestsHandler.cpp`, …) for a codebase that groups verbs into
  `LevelHandler.cpp`, `SystemControlHandler.cpp` and so on. Caveat on that negative: the plugin's
  history is squashed at root `17a331d7`, so "never existed" means "never in the visible history".

**The lesson `E-propertyutils-split-by-concern` `#6` drew generalises further than a file split.**
A rename that moves a *module* invalidates every board citation into it at once, invisibly from
inside the codebase, and nothing at the time noticed. It also travels past paths into test ids and
macros, so grepping the old directory name finds only part of the damage. The durable fix is not
another sweep: it is `F-citation-checker-repo-wide`, which would have failed on all 515 the day
the rename landed.

## History
- `#1-sweep-and-record` `DONE` reporter — Swept the class flagged by `E-propertyutils-split-by-concern` `#6`. **515 citations across 277 ticket files** under `Source/EditorAutomationRpcGateway/` (495) and `Source/EditorAutomationRpcGatewayTests/` (24) — 173 in bodies, 342 in history rows, only 41 carrying a `file:line`. Mapping derived from the plugin's rename graph, not from the string: **481 resolve by the prefix rule** with the target existence-checked at plugin HEAD `ef8a1f1b`, **21 distinct paths move by more** (three gated sub-modules from `b02e4fc7`, two module-named files, seven intra-module moves), and no path exists where the prefix rule resolves but the rename graph disagrees. **165 body citations repointed in place; 8 deliberately left** (two are meta-references quoting the stale path, three are their own ticket's subject, two name deliberately absent files, one now lands on innocent code). **342 history citations left verbatim** per the append-only rule, each covered by a per-ticket mapping row. **All 41 line-bound citations hand-read against HEAD: 12 land, 18 had drifted and were corrected, 11 name a construct that no longer exists.** The other 474 were verified by path existence only and every row says so. One commit per ticket, per `README.md` and the `PropertyUtils` precedent. **Corrected the citation-health figure**: the reported 96% was an engine-only 25-citation hand-check whose scope was lost in relay, backed by a mechanical pass that skipped every plugin basename — all 515 scored neither pass nor fail. Re-derived path-resolution health of the whole board is **83.5%** (5,823/6,975), of which this cluster was 6.9 points. **28 tickets carry a claim that no longer holds**, listed above and recorded individually; the sharpest are three registration summaries advertising fields the handler never returns (filed as `B-registration-summaries-promise-absent-fields`), a DONE ticket whose fix was undone, and a DONE ticket for a verb that no longer exists.
- `#2-correct-the-resolution-split` `DONE` reporter — Correcting one number in `#1` rather than editing it: "481 resolve by the prefix rule" conflated two populations. The exact split of the 515 is **436 by the prefix rule alone, 43 only through the exception table** (the gated sub-modules, the two module-named files and the seven intra-module moves — a prefix rewrite lands on a nonexistent path for every one of those), and **36 that do not resolve mechanically at all**. 479 resolve in total, each existence-checked at plugin HEAD `ef8a1f1b`; the body now states the split. The other figures in `#1` stand.
