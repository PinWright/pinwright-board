---
id: B-asset-dump-widget-tree-xml-slot-recursion-explosion
title: "asset.dump tree.xml recurses Slot.Parent/Slot.Content causing exponential redundant emission (10.9 MB worst-case)"
status: DONE
severity: High
category: bug
tags: [asset-dump, widget, tree-xml, slot, size-bloat, recursion]
---

# `asset.dump` `tree.xml` recurses Slot.Parent/Slot.Content causing exponential redundant emission

Widget Blueprint `tree.xml` dumps walk `UPanelSlot::Parent` (back-reference to
the parent widget) and `UPanelSlot::Content` (forward-reference to the slot's
child widget) as full subtrees inside each `Slot="{...}"` attribute. Because
every sibling slot in a panel emits the same `Parent={Slots={...}}` block —
which itself contains every sibling's `Content` — the output blows up
quadratically per panel level. Cycle / max-depth guards inside
`PropertyUtils::ExportObjectPropertyToJsonValueWithInheritance` (depth limit
3, owner-walk cycle detection) eventually break the recursion, but only after
many redundant copies of the subtree have already been emitted.

## Symptoms

Sweep-wide markers across 344 `tree.xml` files under
`C:/Unity/unreal-fpv/.editor-automation/asset-dumps/`:

- 16,395 `{_kind=cycle, ...}` markers
- 16,381 `{_kind=max_depth, ...}` markers
- ~95 of each per file on average

Top-5 file sizes (confirmed via PowerShell `Get-ChildItem -Recurse -Filter tree.xml | Sort-Object Length -Descending | Select -First 5`):

| Size  | Path |
|------:|------|
| 10.9 MB | `App\App\UI\LobbyAndMenu\HUD\Training\Stabilized\W_HUD_TrainingStabilized_01\tree.xml` |
|  5.8 MB | `App\App\UI\LobbyAndMenu\W_TrackSelect\tree.xml` |
|  5.7 MB | `App\App\UI\LobbyAndMenu\Popups\W_ChangeAvatar\tree.xml` |
|  5.6 MB | `App\App\UI\W_AppMapEditor_ActionPanel\tree.xml` |
|  5.3 MB | `App\App\UI\LobbyAndMenu\TrackEnd\W_MultiplayerUsersFrame\tree.xml` |

Per the wider audit in `E-asset-dump-oversized-fields`, 20 `tree.xml` files
already exceed the 500 KB single-`Read` budget; this ticket addresses the
structural root cause shared across all of them, not just the worst outliers.

## Repro

Sample the head of the worst offender:

```
C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/HUD/Training/Stabilized/W_HUD_TrainingStabilized_01/tree.xml
```

The very first widget's `Slot="{...}"` attribute contains
`Parent={Slots={_kind=cycle,...},{Padding=...,Parent={_kind=max_depth,...},Content={_kind=max_depth,...}},{...},{...},...}` —
a full enumeration of every sibling slot under the parent `Overlay_0`, with
each sibling's own `Parent` and `Content` resolved as `_kind=max_depth`
markers because the depth limiter (`InstancedSubobjectMaxDepth = 3` in
`PropertyUtils.cpp:56`) finally kicks in.

That information is structurally redundant:

- `Parent={Slots=...}` re-emits siblings the XML tree already enumerates as
  child elements of the same panel.
- `Content={...}` points back at the same widget the XML element wraps.
- Both serve no inspection purpose — they exist only because UMG models the
  Slot/Widget relationship as bidirectional instanced UPROPERTYs.

## Root cause

`UPanelSlot::Parent` and `UPanelSlot::Content` are declared
`UPROPERTY(Instanced)`. The asset-dump path calls
`WidgetXmlExporter::BuildWidgetTreeXmlWithDiagnostic(WBP, /*bIncludeDefaults=*/false)`
(`AssetDumpHandler.cpp:1725`, `AssetDumpBuilder.cpp:174`), which has no
`bOmitSlotChain` parameter at all — it always passes `false` into the
recursive `BuildXmlString`.

The plumbing to suppress this already exists. Per the fix shipped for
`E-widget-export-xml-token-limit`:

- `WidgetXmlExporter.cpp:296-301` defines
  `SlotChainSkipSet{TEXT("Parent"), TEXT("Content")}` and threads it through
  `CollectOverriddenAttributes` whenever `bOmitSlotChain == true`.
- `BuildXmlString` (`WidgetXmlExporter.cpp:258`) takes `bool bOmitSlotChain`.
- The MCP `widget.export_xml` handler exposes `omit_slot_chain` / `compact`
  flags (`WidgetXmlExportHandler.cpp:131`) and threads the bool through.

The asset-dump entry point is the only caller that doesn't.

## Fix

Make the asset-dump path always emit `tree.xml` with the slot chain elided —
the dump cache is for inspection, not round-tripping, so the bidirectional
references add zero value and ~95% of the bytes on panel-heavy widgets.

Two-line change shape:

1. `WidgetXmlExporter.h` — add `bool bOmitSlotChain = false` to
   `BuildWidgetTreeXmlWithDiagnostic` (and the `BuildWidgetTreeXml`
   back-compat overload, default `false`).
2. `WidgetXmlExporter.cpp:444-446` — pass the flag into `BuildXmlString`.
3. `AssetDumpBuilder.cpp:174` and `AssetDumpHandler.cpp:1725` — pass
   `/*bOmitSlotChain=*/true`.

After the change, Slot is still emitted as a short property bag
(`Padding`, `HorizontalAlignment`, `VerticalAlignment`, `LayoutData`, etc.)
— only `Parent` and `Content` are dropped, and the XML element hierarchy
carries the structural relationship instead.

Regression coverage: add a `TestWidgetXmlHandlers.cpp` case that builds a
panel with N>=3 children and asserts the resulting XML contains zero
`_kind=cycle` and zero `_kind=max_depth` markers when the asset-dump path
is exercised.

## Non-duplicate scope notes

- `E-widget-export-xml-token-limit` (DONE) added the `omit_slot_chain` flag
  on the **live** `widget.export_xml` MCP call. It does not touch the
  asset-dump-side writer, which is what produces the on-disk
  `tree.xml` files this ticket targets.
- `E-asset-dump-oversized-fields` (DONE) added name-based `$omitted`
  placeholders for impulse responses, packed-actor instance arrays, and
  convex-hull geometry inside `properties.json` / `scs.json`. It explicitly
  scopes itself away from `tree.xml` ("widget hierarchy XML (legit deep
  widgets)") — the fix taxonomy there (property-name skip list with JSON
  placeholder) doesn't apply because the problem here is recursive Slot
  edges, not one oversized UPROPERTY value.
- `B-asset-dump-widget-tree-xml-silently-skipped` (DONE) and
  `B-asset-dump-widget-tree-skips-inherited-root` (DONE) handle the
  opposite failure mode (missing / placeholder tree.xml). Disjoint.

## History
- `#1-initial-repro` `OPEN` reporter — 344 widget `tree.xml` files in current sweep emit 16,395 `_kind=cycle` and 16,381 `_kind=max_depth` markers (~95 of each per file) because `UPanelSlot::Parent` / `UPanelSlot::Content` are walked recursively by `PropertyUtils::ExportObjectPropertyToJsonValueWithInheritance`. Top-5 sizes: 10.9 MB `W_HUD_TrainingStabilized_01`, 5.8 MB `W_TrackSelect`, 5.7 MB `W_ChangeAvatar`, 5.6 MB `W_AppMapEditor_ActionPanel`, 5.3 MB `W_MultiplayerUsersFrame`. Inspecting the worst file's head confirms each Slot emits `Parent={Slots={...all siblings...}}` with each sibling's own Parent/Content resolved as `_kind=max_depth`. Fix infrastructure (`bOmitSlotChain` + `SlotChainSkipSet{Parent, Content}`) already exists in `WidgetXmlExporter.cpp:296-301` from the `E-widget-export-xml-token-limit` rollout but is not threaded through `BuildWidgetTreeXmlWithDiagnostic` (the asset-dump entry point at `AssetDumpBuilder.cpp:174` / `AssetDumpHandler.cpp:1725`). Recommended fix: add `bool bOmitSlotChain` to `BuildWidgetTreeXmlWithDiagnostic`, pass `true` from both asset-dump call sites, regression test that resulting XML contains zero cycle/max_depth markers on a multi-child panel.
- `#2-thread-omit-slot-chain` `IN-REVIEW` developer — Added bOmitSlotChain parameter to BuildWidgetTreeXmlWithDiagnostic + BuildWidgetTreeXml; both asset-dump call sites (AssetDumpBuilder.cpp:174, AssetDumpHandler.cpp:1725) now pass true. Regression test at TestAssetDumpWidgetTreeXmlSlotChain.cpp asserts zero _kind=cycle and zero _kind=max_depth markers on a synthetic Overlay+3xTextBlock widget.
- `#3-skip-mcp-server-offline` `SKIP` tester — Cannot run live `asset.dump` to verify on-disk tree.xml because port 19880 is not listening (UnrealEditor.exe is running but the EditorAutomationRpcGateway HTTP server is not accepting connections; netstat shows the curl probe stuck in SYN_SENT). Code-side evidence is positive: AssetDumpBuilder.cpp:287 and AssetDumpHandler.cpp:1878 both pass `bOmitSlotChain=true`, WidgetXmlExporter.h:64-69 declares the parameter on both entry points, and TestAssetDumpWidgetTreeXmlSlotChain.cpp asserts the exact symptoms (zero `_kind=cycle`, zero `_kind=max_depth`, no `Slot.Parent`, no `Slot.Content`) on a 3-child Overlay. Re-verify after the gateway is reachable.
- `#4-verify-fix` `DONE` tester — Ran live `asset.dump` on both top-2 offenders. `W_HUD_TrainingStabilized_01/tree.xml` dropped 10.9 MB → 68.3 KB (159x smaller); `W_TrackSelect/tree.xml` dropped 5.8 MB → 41.3 KB (141x smaller). Grep for `_kind=cycle`, `_kind=max_depth`, `Slot.Parent`, `Slot.Content` in both regenerated files: zero matches. Fix is live end-to-end.
