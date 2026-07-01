---
id: F-bpir-override-merge-mode
title: "`compile_bpir` can't extend existing override body (no merge mode)"
status: DONE
severity: ""
category: feature
tags: []
---

# `compile_bpir` can't extend existing override body (no merge mode)

`compile_bpir entry override <Name>()` errors with `Found more than one function with the same name <Name>; second occurance at Event <Name>` when the target BP already has an override for that function. Blocks MCP authors from adding logic to any common hook (`PreConstruct`, `Construct`, `BP_OnActivated`, `BP_OnDeactivated`, `Tick`) on BPs that already override it.

**Impact this session:**
- `W_SingleRaceResultsFrame` already has `entry override Construct()` with a `bind_dispatcher OnBackHandled` body — couldn't add a new subscription via another `Construct` override.
- `W_RaceWithOpponentsResultsFrame` same issue.
- `W_RaceOnlineResultsFrame` already has `Construct`, `BP_OnActivated`, `BP_OnDeactivated`, and `Tick` with bodies — had to use `PreConstruct` (orphan event node had to be manually deleted first before a fresh PreConstruct override could compile).

**Workaround today (session-tested):**
1. Find the existing `entry override <Name>()` body via `blueprint_decompile_bpir`.
2. `mode: replace` with the combined body (existing + new), risking corruption if the agent doesn't reconstruct the existing logic faithfully.
3. OR use `insert_bpir_at_node` to splice new code into the existing override's exec chain — but that requires knowing specific node IDs in the existing body, which are opaque.
4. OR pick a different, unused hook (e.g. `PreConstruct` when `Construct` is taken) — but most common hooks are already used on mature BPs.

**Proposal:** Add `mode: "extend"` (or `mode: "append-to-override"`) to `compile_bpir`. When used with an `entry override <Name>()`, the emitted body is appended to the existing override's exec chain (after the last exec node in the existing body, before any reconvergence terminator). The new body's labels must be unique, and control flow from the existing body's terminal exec flows into the new body's first exec node. Alternative implementation: a new top-level `blueprint.append_to_override` handler that takes the override name + BPIR body and does the splice.

**Why this matters for the "no manual" goal:** Any widget/actor BP that's already customized (which is most production BPs) will have at least one of the common overrides occupied. Without merge mode, MCP-driven customization is blocked on any such BP.

## History
- `#1-extend-mode-blocked` `OPEN` reporter — Hit 3 times this session on the three results modals while wiring the `OnSaveStateChanged` subscription. Workaround using `PreConstruct` worked for 2 modals but required deleting an orphan `Event PreConstruct` first on the third. For a BP with all common hooks occupied, there's no clean MCP path.
- `#2-added-extend-mode` `IN-REVIEW` developer — Added `EBpirCompileMode` enum (Default/Replace/Extend) threaded from `BpirCompilerHandler.cpp` into `FBpirCompiler::Compile`. Extend mode: Phase 0 skips deletion of the target entry; `FindTerminalExecOutputPin` (new static helper in `BpirCompiler.cpp`) walks the existing exec chain to the last pin with no outgoing link; `SetupOverride` returns that pin as the splice point. Errors cleanly on forked exec flow. Falls back to Default behavior when no existing entry exists. [Scope note: applies to entry override; other entry kinds receive Phase 0 skip but no terminal-splice — they create a fresh entry.] Test added: `FCompilerIntegrationExtendModeTest`. Docs: `bpir-language-reference.md` mode table extended.
- `#3-verification-failed` `OPEN` reporter — **Verification failed.** Session re-test on two different BPs with unforked existing `Construct` bodies:
  - `W_SingleRaceResultsFrame` (existing Construct: single `bind_dispatcher OnBackHandled(Delegate: $OutputDelegate)`): `compile_bpir { mode: "extend", code: "entry override Construct() { call PrintString(InString: \"extended\") }" }` → `COMPILE_FAILED: Found more than one function with the same name Construct; second occurance at Event Construct`. `nodeCount: 2` — the extend path created a NEW Construct entry instead of splicing into the existing one.
  - `W_RaceWithOpponentsResultsFrame` (same existing Construct shape): same error.

  The error message `Found more than one function with the same name` is the SAME error that default/non-extend mode produces when re-declaring an existing override. This strongly suggests the extend path is detecting "no existing entry" for these BPs (so it falls back to default, which creates a duplicate and errors). Possible causes: (a) the extend-path entry detection doesn't find the existing Construct because it searches by wrong criteria (e.g., matches only entries authored by the current BPIR, not pre-existing ones); (b) the skip-deletion branch triggers but then a later phase still emits a new entry. Re-verify against both fresh repro BPs, and ensure extend mode's "existing entry detection" operates on the BP's current graph (not just BPIR-authored history).
- `#4-fixed-setup-override-lookup` `IN-REVIEW` developer — Root cause confirmed (a): `SetupOverride`'s extend path used `FBlueprintEditorUtils::FindOverrideForFunction(BP, OverrideClass, FName)` to locate the existing event node. That UE helper requires an exact `OverrideClass` match against `EventReference.MemberParent`. When the existing entry on `W_SingleRaceResultsFrame` was created via the editor's right-click "Override" or by an earlier MCP call, its EventReference may store an intermediate ancestor class (not the most-derived parent that `TryResolveBlueprintOverride` returns). Result: `FindOverrideForFunction` returns nullptr, `bExistedBefore == false`, and SetupOverride's create-new branch produces a duplicate entry — fatally colliding with the existing one. Phase 0 (used by Default/Replace) is more lenient: it iterates all `UbergraphPages`, casts each node to `UK2Node_Event`, requires `bOverrideFunction == true`, and matches purely by `EventReference.GetMemberName()` (case-insensitive). Fix: replaced the `FindOverrideForFunction` call in `SetupOverride` (~line 2415-2438 in BpirCompiler.cpp) with the same ubergraph iteration Phase 0 uses — find any matching-name override event regardless of EventReference class. Now Default and Extend see the same set of "existing entries" and Extend correctly identifies the pre-existing Construct override on `W_SingleRaceResultsFrame`/`W_RaceWithOpponentsResultsFrame`.
- `#5-verified-extend-splice` `DONE` tester — User marked verified. Session re-test on `W_RaceWithOpponentsResultsFrame` with existing `Construct` body containing `bind_dispatcher OnBackHandled(...)`: `compile_bpir { mode: "extend", code: "entry override Construct() { call PrintString(InString: \"extended via merge\") }" }` returns `compiled: true`, 1 new node (only the PrintString). Decompile confirms single Construct entry with the original `bind_dispatcher` followed by the appended `PrintString` — correctly spliced after existing terminal exec node. No duplicate Construct. No `Found more than one function` error.
