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
`PropertyExport.cpp::ExpandInstancedSubobject` (`:290-318`; depth limit 3,
owner-walk cycle detection) eventually break the recursion, but only after
many redundant copies of the subtree have already been emitted.

> **Citation repoint.** This paragraph originally named
> `PropertyUtils::ExportObjectPropertyToJsonValueWithInheritance`. That symbol
> **does not exist and never existed** — zero hits over `Source/`, and
> `git log --all -S` finds it only in this board file's own text. It is not a
> rename, so it could not simply be re-pointed. The function that actually owns
> both markers is `ExpandInstancedSubobject`
> (`Source/PinWright/Private/Utils/PropertyExport.cpp:290-318`): `_kind=max_depth`
> emitted at `:299`, `_kind=cycle` at `:308`. The nearest name-match,
> `ExportPropertyToJsonValueWithInheritance` (`PropertyExport.cpp:1324`), carries
> no depth or cycle guard of its own and reaches them only via
> `ExportPropertyToJsonValue` (`:605`).

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
`Source/PinWright/Private/Utils/PropertyExport.cpp:238`, enforced at `:296`)
finally kicks in.

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

> **Pre-fix state; HEAD pointers.** The sentence above describes the defect as
> filed and is left as written. At HEAD (this ticket is DONE) both call sites
> pass `/*bOmitSlotChain=*/true`, and the two citations have drifted:
> `AssetDumpHandler.cpp:1725` → `Handlers/Asset/AssetDumpHandler.cpp:2777` (inside
> `BuildWidgetTreeAspect_Internal`, `:2771`); `AssetDumpBuilder.cpp:174` →
> `Utils/AssetDumpBuilder.cpp:301` — note the file moved out of `Handlers/Asset/`
> into `Utils/` and is now only 315 lines, so `:174` is a live but unrelated line.
> One factual correction to the sentence: the two call sites do **not** both go
> through `BuildWidgetTreeXmlWithDiagnostic`. `AssetDumpBuilder.cpp:301` calls the
> back-compat overload `BuildWidgetTreeXml` (`Handlers/UI/WidgetXmlExporter.h:71`);
> only `AssetDumpHandler.cpp:2777` calls the diagnostic form.

The plumbing to suppress this already exists. Per the fix shipped for
`E-widget-export-xml-token-limit`:

- `WidgetXmlExporter.cpp:296-301` defines
  `SlotChainSkipSet{TEXT("Parent"), TEXT("Content")}` and threads it through
  `CollectOverriddenAttributes` whenever `bOmitSlotChain == true`.
  *(HEAD: `Handlers/UI/WidgetXmlExporter.cpp:362-370`, the set itself at `:366`.)*
- `BuildXmlString` (`WidgetXmlExporter.cpp:258`) takes `bool bOmitSlotChain`.
  *(HEAD: `:318`.)*
- The MCP `widget.export_xml` handler exposes `omit_slot_chain` / `compact`
  flags (`WidgetXmlExportHandler.cpp:131`) and threads the bool through.
  *(HEAD: `Handlers/UI/WidgetXmlExportHandler.cpp:142`.)*

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
   *(HEAD: `Handlers/UI/WidgetXmlExporter.cpp:520`.)*
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
- `#5-repoint-citations-after-property-utils-split` `DONE` reporter — Citation maintenance only; **no behavioural claim in this ticket changes and its status is untouched**. `Utils/PropertyUtils.cpp` was split into `PropertyExport.cpp` / `PropertyImport.cpp` / `PropertyInspection.cpp` / `PropertyDiff.cpp` (`PropertyUtils.h` survives only as a deprecated umbrella forwarder), and the module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/`, so every `PropertyUtils.cpp` citation on this ticket was an unresolvable path a fixer could not open. Body citations repointed and verified line-by-line at plugin HEAD `ef8a1f1b`; **all of this ticket's `PropertyUtils` code landed in `PropertyExport.cpp`**, none in the other three. **One citation could NOT be resolved and was corrected rather than guessed:** `PropertyUtils::ExportObjectPropertyToJsonValueWithInheritance` **never existed** — zero hits over `Source/`, and `git log --all -S` on that name matches only this board file's own text, so it is not a rename. The behaviour it was credited with belongs to `ExpandInstancedSubobject` (`Utils/PropertyExport.cpp:290-318`), which emits `_kind=max_depth` at `:299` and `_kind=cycle` at `:308`; the nearest name-match, `ExportPropertyToJsonValueWithInheritance` (`:1324`), holds no guard of its own. History rows `#1`-`#4` are left verbatim per the append-only rule, so their citations map as follows: `#1`'s `ExportObjectPropertyToJsonValueWithInheritance` → `PropertyExport.cpp::ExpandInstancedSubobject` `:290-318` as above. Depth limit `PropertyUtils.cpp:56` → `PropertyExport.cpp:238` (`constexpr int32 InstancedSubobjectMaxDepth = 3;`), enforced `:296`. Neighbouring non-`PropertyUtils` citations re-verified in the same pass and annotated in the body: `WidgetXmlExporter.cpp:296-301` → `Handlers/UI/WidgetXmlExporter.cpp:362-370` (set at `:366`); `:258` → `:318`; `:444-446` → `:520`; `WidgetXmlExportHandler.cpp:131` → `Handlers/UI/WidgetXmlExportHandler.cpp:142`; `AssetDumpHandler.cpp:1725` → `Handlers/Asset/AssetDumpHandler.cpp:2777`; `AssetDumpBuilder.cpp:174` → `Utils/AssetDumpBuilder.cpp:301` — that file **moved out of `Handlers/Asset/` into `Utils/`** and is now 315 lines, so `:174` had become a live but unrelated line, the most misleading form of drift. One substantive correction recorded while re-deriving: the § *Root cause* claim that both asset-dump call sites go through `BuildWidgetTreeXmlWithDiagnostic` is wrong — `AssetDumpBuilder.cpp:301` calls the back-compat overload `BuildWidgetTreeXml` (`WidgetXmlExporter.h:71`); only `AssetDumpHandler.cpp:2777` calls the diagnostic form. `#3`'s `AssetDumpBuilder.cpp:287` / `AssetDumpHandler.cpp:1878` were already superseded by `#4` and are not repointed.
