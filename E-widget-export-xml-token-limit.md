---
id: E-widget-export-xml-token-limit
title: "`widget.export_xml` output overflows MCP tool-result token limit on real HUDs"
status: DONE
severity: Low
category: ergonomic
tags: [widget, export-xml, token-limit, slot-serialization, ergonomics]
---

# `widget.export_xml` output overflows MCP tool-result token limit on real HUDs

`widget.export_xml` on a typical CommonUI HUD with ~30 sub-widgets returns
an XML payload large enough to overflow the MCP tool-result token limit.
Observed live on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_WaitingLobby`:
71,877-character response, persisted to a
`mcp-editor-automation-call-{ts}.txt` fallback file. The agent then has
to Grep through the fallback file to find any string of interest — the
return-value pipeline is bypassed entirely.

`include_defaults` is already `false` by default, so the size is not
from explicitly-requested defaults. The bulk per widget is the
`Slot="{...}"` attribute, which currently carries three classes of
content:

1. **The widget's own slot data** (Padding, alignment, anchors) —
   useful. Generally <200 chars per widget.
2. **`Parent={Slots={...}}` chain** — for every widget, the serializer
   dumps every sibling slot in the parent's panel. On a Canvas/Overlay
   with 10 children, the per-child output includes the other 9 slots
   too. O(N^2) blow-up per panel level.
3. **`Content={...}` block** — dumps the widget's full UPROPERTY
   default state inside the slot attribute, including signal lines
   that carry no information for inspection workflows:
   `OnVisibilityChanged={_kind=FMulticastInlineDelegateProperty,value=()}`,
   `WidgetTree=`, `bIsVolatile=false`, `bIsEnabled=true`, etc.

Empirically the Slot attribute is 70–90% of each widget's serialized
size. Most of that is redundant (sibling chain) or default-valued
(unbound delegates, untouched bools).

**Impact:** Any inspection of a non-trivial HUD via `widget.export_xml`
requires the fallback-file → Grep workaround. The structured XML the
tool produces is unusable as a direct response. Asset-dump cache helps
when fresh, but live verification after edits cannot lean on it.

**Workaround:** Read the cached `tree.xml` from
`.editor-automation/asset-dumps/...` if available; otherwise consume the
persisted `mcp-editor-automation-call-{ts}.txt` via Grep.

**Fix:** Add an `omit_slot_chain` (or `compact: true`) param that elides
the `Parent={Slots=...}` recursive sibling dump and drops UPROPERTY
values that match the slot class's CDO defaults. This is the highest
signal-to-noise change — sibling-chain elision alone should cut output
~5-10× on panel-heavy HUDs without removing any per-widget info the
caller actually asked for, and CDO-equality filtering inside `Content`
strips the `OnVisibilityChanged=()`-style noise lines. The flag stays
off by default so existing callers see no change; inspection-style
agents opt in. Variants considered but not recommended now:

- *Make `include_defaults=false` aggressively drop CDO-equal slot
  UPROPERTYs unconditionally.* Cleaner per-call ergonomics but
  changes behavior of an already-shipped flag — risks breaking
  consumers that depend on the current verbose shape.
- *Add a `widget_names_only` summary mode (tree shape, class+name+children, no props).*
  Useful but `widget.describe` with low `max_depth` already covers
  "what's in this widget" queries; would duplicate that surface.

## History
- `#1-initial-repro` `OPEN` reporter — `widget.export_xml` on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_WaitingLobby` returned 71,877 chars, overflowed MCP tool-result token limit, response persisted to fallback file; agent had to Grep the file to extract any data. Slot attribute (Parent sibling chain + Content UPROPERTY default dump) is 70-90% of per-widget size. Proposes `omit_slot_chain` / `compact` flag that drops sibling chain and CDO-equal property values; existing `E-widget-describe-slot-truncation` (DONE) covers `widget.describe`, not export_xml — no overlap. Existing `B-widget-export-xml-*` entries are XML well-formedness fixes, not size.
- `#2-omit-slot-chain-flag` `IN-REVIEW` developer — Promoted `CollectOverriddenAttributes` to `WidgetXmlExporter::` namespace with optional `PropertyNameSkipSet`; added `bOmitSlotChain` flag to `BuildXmlString`; new `omit_slot_chain` / `compact` params in `widget.export_xml` (off by default). Regression: `EditorAutomationRpcGateway.widget.export_xml.OmitSlotChainDropsSkipSetProperties`.
- `#3-compact-still-emits-slot-chain` `OPEN` tester — Returned: `compact=true` still emitted `Slot` attributes containing `Parent={Slots=...}` sibling chains and `Content={...}` blocks instead of eliding them. Test: `widget.export_xml` with `{"assetPath":"/App/App/UI/LobbyAndMenu/HUD/W_HUD_WaitingLobby","compact":true}`.
- `#4-compact-skips-raw-slot` `IN-REVIEW` developer — `WidgetXmlExporter.cpp` now skips the raw widget `Slot` property through `CollectOverriddenAttributes(..., PropertyNameSkipSet)` when `compact` / `omit_slot_chain` is active, while preserving dedicated per-slot layout attributes; added regression `EditorAutomationRpcGateway.widget.export_xml.CompactOmitsRawWidgetSlotChain`.
- `#5-verify-fix` `DONE` tester — Verified: `widget.export_xml` on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_WaitingLobby` with `compact=true` returns inline XML at 2,321 bytes (down from the originally-reported 71,877; default mode now overflows at 340,474 chars to a fallback file). Grep against the compact XML for `Parent={Slots`, `Content={`, and `Slot="{` returned zero matches; per-widget layout still emitted via `Slot.LayoutData=...`. Default behavior unchanged (overflow + fallback file), so existing callers see no regression.
