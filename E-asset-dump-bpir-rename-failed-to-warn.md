---
id: E-asset-dump-bpir-rename-failed-to-warn
title: "`BPIR_FAILED:` marker in bpir.txt overstates severity for source-asset oddities"
status: DONE
severity: Low
category: ergonomic
tags: [bpir, asset-dump, diagnostics, severity-labels]
---

# `BPIR_FAILED:` marker in bpir.txt overstates severity for source-asset oddities

`BuildBpirText` in `Plugins/PinWright/Source/PinWright/Private/Utils/AssetDumpBuilder.cpp:152-165` emits

```
# BPIR_FAILED: <first warning>
```

after each graph's body whenever `FBpirDecompileResult::Warnings` is non-empty. The literal text `BPIR_FAILED` reads as a fatal decompiler error, but the overwhelming majority of warnings that trigger it are **diagnostics about oddities in the source Blueprint** — not failures of the BPIR decompiler itself. A consumer (LLM agent, audit script) grepping the dump corpus for `failed`/`FAILED` lights up on hundreds of benign source-asset notes and alarms wrongly.

## Current corpus signal

`find … -name bpir.txt | xargs grep BPIR_FAILED:` on the fresh App+Game cache returns ~170 marker lines across ~43 widget BPIRs. Top categories:

| Count | Marker text |
|------:|-------------|
| ~138 | `Data pin 'self' has N connections but only the first is used` |
| ~30  | `Orphaned node not reachable from any entry point: <graph> :: <class> '<title>' nodeId=<guid> @(x,y)` |
| 1    | `widget_event <Name>.<Event>: widget variable '<Name>' not found - it may have been removed or recreated without recompiling` |

All three are **source-asset oddities** (multi-connected self pin, dangling node, stale widget reference) — the decompiler did its job and is reporting graph hygiene. The label `FAILED` is wrong; `WARN` / `NOTE` would match the actual severity.

## All warning sites — warn vs. error inventory

A full pass over `Decompiler/BpirDecompiler.cpp` shows the `Warnings` array is fed from ~13 sites. Most are source-side diagnostics, but **two sites genuinely represent an internal decompiler failure**:

| Line | Warning | Severity |
|-----:|---------|----------|
| 267  | `Skipped N delegate signature graph(s) (not supported by decompiler)` | source/feature-gap (warn) |
| 277/292/307 | `Skipped anim-graph X — use AGIR (anim.decompile_agir) instead` | wrong-tool (warn) |
| 608/616 | `widget variable not found` / `delegate binding is invalid` | source-side (warn) |
| 811  | `Orphaned node not reachable from any entry point: …` | source-side (warn) |
| 873  | `Composite inline depth exceeded 1024 — aborting flatten pass` | mixed (could be pathological source OR a missed cycle in the inliner) |
| 909  | `Composite '%s' had un-twinned boundary pins during inline; partial flatten applied` | source-side (warn) |
| 959  | `Exec pin '%s' has N connections but only the first is followed (BPIR constraint)` | source-side / IR-limitation (warn) |
| **1068** | `Reconvergence: could not find line index for node '%s'` | **decompiler-internal failure (error)** — state bookkeeping bug |
| 1293 | `Cast node has no resolved target type (emitted as cast<Unknown>): …` | source-side (warn) |
| 1895 | `Data pin '%s' has N connections but only the first is used` | source-side (warn) |
| 1914 | `Unresolvable value: pin '%s' connected through knot to null source` | source-side (warn) |
| **2043** | `Unresolvable value: source node '%s' was visited but produced no value name` | **decompiler-internal failure (error)** — the inline comment itself says `(shouldn't normally happen)` |
| 2280 | `Optional pin '%s' has no connection or default and no typed-default literal; emitted as <unresolved>.` | source-side (warn) |

Because at least two cases are real decompiler-internal failures, **a blanket rename `BPIR_FAILED` → `BPIR_WARN` would be wrong**: it would hide the cases where the dump genuinely indicates an emitter bug.

## Proposed fix

Tag the marker at emit time based on severity, not a single blanket prefix. Concretely:

1. Extend `FBpirDecompileResult` (or wrap each warning) with a severity tag: `Warn` vs `Error`. `Error` is reserved for the two internal-bookkeeping sites above (1068, 2043) and any future "shouldn't happen" path; everything else is `Warn`.
2. In `AssetDumpBuilder.cpp:120-123`, emit:
   - `# BPIR_WARN: <message>` for `Warn`-tagged warnings (the common case).
   - `# BPIR_ERROR: <message>` for `Error`-tagged warnings (the rare decompiler-internal failure).
3. When a graph has mixed severities, emit the highest-severity marker (still `# BPIR_ERROR:` if any error is present), then optionally one `# BPIR_WARN:` per remaining warning, or fold them into a count summary — pick whichever is least noisy.
4. Preserve the existing warning *message text* unchanged so existing grep patterns over the body of the message still work; only the `BPIR_FAILED:` prefix changes.

Consumers grepping the dumps for "real decompiler failures" then filter on `BPIR_ERROR:`; consumers wanting graph-hygiene notes grep `BPIR_WARN:`. Today both signals are collapsed into the same overstated `BPIR_FAILED:` token.

## Scope

- Source touch points: 1 emit site (`AssetDumpBuilder.cpp:122`) plus a small severity classification on each `Warnings.Add(...)` call in `BpirDecompiler.cpp` (~13 sites). Most flip to `Warn`; sites 1068, 2043 (and arguably 873) become `Error`.
- Tests: the existing `B-bpir-stub-no-empty-marker` regression test asserts the `# BPIR_FAILED:` substring — update it to the new prefix(es). Add one new test that pins severity classification on a known-error site (e.g. construct a scenario that trips site 2043 and assert `# BPIR_ERROR:` is emitted, not `# BPIR_WARN:`).
- No format-version bump needed; this is a text-marker cosmetic change in the dump output.

**Workaround until fixed:** consumers should not treat `BPIR_FAILED:` as fatal. Grep for the message tail (e.g. `Data pin 'self' has`, `Orphaned node not reachable`, `source node .* was visited but produced no value name`) to discriminate severity.

## History
- `#1-initial-survey` `OPEN` reporter — Marker text `BPIR_FAILED:` is emitted at one site (`AssetDumpBuilder.cpp:122`) whenever any decompiler warning fires. Corpus sweep of the App+Game asset-dump cache shows ~170 occurrences in ~43 widget BPIRs, dominated by `Data pin 'self' has N connections but only the first is used` (~138) and `Orphaned node not reachable from any entry point: …` (~30) — both pure source-asset oddities, not decompiler failures. Inventory of all 13 `Warnings.Add` sites in `BpirDecompiler.cpp` shows two true internal-failure cases (line 1068 `Reconvergence: could not find line index`, line 2043 `source node was visited but produced no value name` — the latter's own comment says "shouldn't normally happen"). A blanket rename to `BPIR_WARN:` would silence those legitimate error signals, so the proposed fix is a two-tier tagging scheme: `BPIR_WARN:` for source-side notes (the overwhelming majority), `BPIR_ERROR:` for the two internal-failure sites. Preserves the warning text itself; only the prefix token changes.
- `#2-severity-tagged-markers` `IN-REVIEW` developer — Added `EBpirWarningSeverity` + `FBpirWarning` to `BpirDecompiler.h`; `FBpirDecompileResult::Warnings` and `FEntryState::Warnings` are now `TArray<FBpirWarning>`. Tagged the 16 `Warnings.Add`/`OutWarnings.Add`/`AllWarnings.Add` sites in `BpirDecompiler.cpp` (Error: L873 composite-inline-depth-exceeded, L1068 reconvergence-line-index-missing, L2043 source-node-visited-no-value-name; Warn: L267 delegate-signature-graph-skipped, L277/L292/L307 anim-graph-skipped, L608/L616 component-bound-event-stale, L811 orphaned-node, L909 composite-un-twinned, L959 exec-pin-multi-followed, L1293 cast-unknown-target, L1895 data-pin-multi-used, L1914 unresolvable-value-knot-null, L2280 optional-pin-unresolved). `AssetDumpBuilder::FormatBpirWarningMarkers` (new namespace-scope helper exported via `AssetDumpBuilder.h`) emits `# BPIR_WARN:` / `# BPIR_ERROR:` per warning instead of a single `# BPIR_FAILED:` line. `BlueprintDecompilerHandler` JSON wire format now emits warnings as `{"text":..., "severity":"warn"|"error"}` objects (BPIR wire format is ephemeral per `editor_automation_ir_ephemeral.md`). New `TestBpirSeverityMarkerEmit.cpp` pins the helper's severity→prefix mapping; orphan-warnings test extended to assert orphan warnings are tagged `Warn` severity. All downstream BPIR test iterations of `Result.Warnings` updated from `const FString&` to `const FBpirWarning&` accessing `.Text`.
- `#3-skip-editor-offline` `SKIP` tester — Editor not running (TCP 19880 refused connection), so live `blueprint.decompile` / `asset.dump` calls to confirm the emitted prefix are unavailable. Asset-dump cache under `.editor-automation/asset-dumps/` is stale (bpir.txt files dated 2026-05-13, source change at 2026-05-19) and grep returns 0 hits for both old `BPIR_FAILED:` and new `BPIR_WARN:`/`BPIR_ERROR:` — cache predates the fix and can't disprove it. Source inspection confirms the code change is in place (`AssetDumpBuilder.cpp:157-159` emits the two new prefixes via `FormatBpirWarningMarkers`, `TestBpirSeverityMarkerEmit.cpp` pins the mapping and asserts absence of `BPIR_FAILED:`), but per protocol, source-only verification is not sufficient; needs a live editor + re-dump to PASS.
- `#4-verify-live-dump` `DONE` tester — Ran `asset.dump` on `/App/App/UI/W_AppMapEditor_ActionPanel.W_AppMapEditor_ActionPanel` (representative widget known to trigger orphan-node warnings). Fresh `bpir.txt` at `C:/Unity/unreal-fpv-pluginwork/.editor-automation/asset-dumps/App/App/UI/W_AppMapEditor_ActionPanel/bpir.txt` contains 5 `# BPIR_WARN: Orphaned node not reachable…` lines and 0 occurrences of the old `BPIR_FAILED:` token. Confirms the severity-tagged marker emit is live end-to-end through `AssetDumpBuilder::FormatBpirWarningMarkers`.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. Doubled stale prefix (`Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/…`), so both segments are rewritten. **Not just a path — the marker this ticket is about is retired.** `BPIR_FAILED` appears nowhere in production code; `:122` is now the `redirectsTo` field. The severity-split replacement is `FormatBpirWarningMarkers` (`:152-165`), emitting `# BPIR_ERROR:` and `# BPIR_WARN:`, called from `BuildBpirText` at `:194`, and `Tests/Bpir/TestBpirSeverityMarkerEmit.cpp:32-33` asserts the old prefix never returns. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
