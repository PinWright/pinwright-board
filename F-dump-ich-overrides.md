---
id: F-dump-ich-overrides
title: "scs.json ignores InheritableComponentHandler overrides on child BPs"
status: DONE
severity: Medium
category: feature
tags: [scs, dump, ich, inherited-override, asset-dump]
---

# scs.json ignores InheritableComponentHandler overrides on child BPs

When a child Blueprint overrides a property on a component inherited from a parent Blueprint, that override is stored in `Blueprint->InheritableComponentHandler` (`UInheritableComponentHandler`), not in SCS. The dump never iterates `ICH->Records`, so child-BP overrides on inherited components are silently dropped from `scs.json`.

For example: parent BP sets `StaticMesh = SM_Fuselage`; child BP overrides it to `SM_Fuselage_Variant`. The child's `scs.json` will show no entry for that component at all, making it appear the override does not exist.

**Fix:** Call `Blueprint->GetInheritableComponentHandler(false)`; if non-null, iterate `Records`. For each record with a valid `ComponentTemplate`, emit a JSON entry with `source: "inherited-override"`. The component name comes from `Record.ComponentKey.GetSCSVariableName()`. The diff base is the parent template (located via the parent BP's SCS `FindSCSNode`), falling back to the class CDO if the parent template is not found.

## History
- `#1-feature-request` `OPEN` reporter — ICH override records on child BPs are never written to scs.json; parent-set property overrides are invisible to consumers.
- `#2-iterate-ich-records` `IN-REVIEW` developer — Added ICH iteration in `GetBlueprintSCS` (EditorAutomationRpcGateway_SCSHandlers.cpp). After native components are emitted, the function calls `Blueprint->GetInheritableComponentHandler(false)`, iterates `Records`, and for each valid record emits a JSON entry with `source: "inherited-override"`. Component name from `Record.ComponentKey.GetSCSVariableName()`. Diff base resolved via parent BP SCS `FindSCSNode` walk, falling back to class CDO. Shared `AddOverriddenProperties` helper used.
- `#3-fix-ue56-ich-private-records` `IN-REVIEW` developer — Compile error fix: `ICH->Records` is private in UE 5.6; switched to public `CreateRecordIterator()`. Drop-in replacement, same semantics.
- `#4-agent-verified-pass` `IN-REVIEW` tester — PASS. Test: `asset.dump` on `/App/HELIOS/Drones/Icarus/Parent/Icarus_ParentBP`. scs.json shows 31 entries with `source: "inherited-override"` covering inherited components from the BP parent chain (Camera HUB, Motors:* set, FPV Camera:*, Drag, Trail, ArrowComponents, AudioComponents, etc.). Properties block sparse — only values that differ from the parent template are listed (e.g. `CurrentAperture: 5` on the cinecam, `bAbsoluteScale: true` on several SceneComponents).
- `#5-reverified-this-pass` `DONE` tester — Re-verified during /mcp-review on a fresh dump: scs.json for Icarus_ParentBP contains 31 entries with `source:"inherited-override"` (matches #4 count exactly). Confirms ICH iteration via `CreateRecordIterator()` is producing the documented output. Marking DONE.
