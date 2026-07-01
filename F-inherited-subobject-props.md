---
id: F-inherited-subobject-props
title: "`blueprint.scs.set_property` edits inherited SCS parent templates instead of child overrides"
status: DONE
severity: Medium
category: feature
tags: [blueprint, scs, inherited-components]
---

# `blueprint.scs.set_property` edits inherited SCS parent templates instead of child overrides

Inherited Blueprint SCS components can be found by `blueprint.scs.set_property`, but the inherited-SCS fallback must not write into the parent Blueprint's `USCS_Node::ComponentTemplate`.

Child Blueprint runtime instances read from the child-owned `UInheritableComponentHandler` override template. Editing only the parent template makes the RPC report success while fresh dumps and spawned child instances still show the old value.

**Fix:** For inherited Blueprint SCS nodes, resolve the parent `USCS_Node` by component name, build `FComponentKey(ParentNode)`, and edit the child Blueprint's existing or newly created `UInheritableComponentHandler` override template. Keep local SCS component templates and native CDO default-subobject fallback behavior unchanged.

## History
- `#1-inherited-subobject-fails` `OPEN` reporter — TrackUIComponent.RouterClass on B_PhotoInspection couldn't be set via MCP. Fixed via `mcp__editor_automation__.call path="python.execute" args={...}`.
- `#2-added-three-tier-lookup` `IN-REVIEW` developer — Added 3-tier fallback lookup in `SetSCSComponentProperty`: (1) local SCS (existing), (2) parent Blueprint SCS hierarchy via `ParentClass->ClassGeneratedBy`, (3) CDO default subobjects via `ForEachObjectWithOuter`. Response includes `source` field: `"local"`, `"inherited_scs"`, or `"default_subobject"`. Two tests: CDO subobject property set and local SCS regression.
- `#3-verified-cdo-lookup` `DONE` tester — Verified: scs_set_property on B_PhotoInspection with componentName:"TrackUI" (C++ default subobject on PhotoInspectionTrack). Response: source:"default_subobject", compiled:true, saved:true. 3-tier CDO lookup works.
- `#4-returned-geoscan-tags` `OPEN` tester — Returned: `blueprint.scs.set_property` on `/App/HELIOS/Drones/Atlas/B_Geoscan801` inherited prop components reported success with `source:"inherited_scs"`, `compiled:true`, and `saved:true`, but a fresh `asset.dump` and a temporary spawned instance still showed empty runtime `ComponentTags`. The edit hit the inherited parent template path, not the child Blueprint component template that runtime instances use. Workaround was `python.execute` with `SubobjectDataBlueprintFunctionLibrary.get_object_for_blueprint(...)` to mutate the child template directly.
- `#5-child-template-overrides` `IN-REVIEW` developer — `blueprint.scs.set_property` now resolves inherited Blueprint SCS components through the child Blueprint's `UInheritableComponentHandler` override template using `FComponentKey(ParentNode)` instead of mutating the parent `USCS_Node::ComponentTemplate`. Review follow-up delays child override creation until after read-only parent-template path validation and temporary-object JSON value validation; regression coverage now proves invalid inherited property paths/values do not create child overrides before the valid child-only `ComponentTags` override is applied.
- `#6-verify-child-override-persists` `DONE` tester — Verified: `blueprint.scs.set_property` on B_Geoscan801 with componentName "Drag: Fluid Audio" (inherited-only AudioComponent from HELIOS_BP, no prior override), propertyName "ComponentTags", value `["VerifyTag_F_inherited"]`. Response: source "inherited_scs", compiled true, saved true. Fresh `asset.dump` then showed a new `inherited-override` entry for "Drag: Fluid Audio" with `ComponentTags: ["VerifyTag_F_inherited"]`, while the original `inherited-scs` parent entry was unchanged — child UInheritableComponentHandler override is created and persisted as designed. Restored tags to `[]` afterward.
