---
id: B-property-component-parent-default
title: "property.reset uses component class default for inherited component templates"
status: DONE
severity: Medium
category: bug
tags: [property, reset, inherited-components, parent-template]
---

# property.reset uses component class default for inherited component templates

`property.list` and `property.reset` on child Blueprint component templates can use the bare component class CDO as the default instead of the nearest parent Blueprint component template. For inherited component overrides, this does not match the Blueprint editor reset arrow.

Session repro: `/App/HELIOS/Drones/Atlas/B_Geoscan801.B_Geoscan801_C:Motors: FR Move_GEN_VARIABLE` is an inherited prop-plane component whose parent `HELIOS_BP` template has `ComponentTags=["Prop_FR_Spin"]`. `property.list` reported `ComponentTags.value=["Prop_FR_Spin"]` after manual repair, but `defaultValue=[]` and `isOverridden=true`. A `property.reset` on that field would therefore clear the prop-discovery tag instead of restoring the parent `HELIOS_BP` value.

Impact: callers cannot safely use MCP to perform the same reset as the Blueprint details-panel arrow for inherited component properties. In the Geoscan prop case, this blocks restoring `ComponentTags` inheritance and forces a Python fallback.

**Workaround:** Use `python.execute` with `SubobjectDataBlueprintFunctionLibrary.get_object_for_blueprint(...)` to target the child Blueprint component template, then copy the intended parent-template value explicitly.

**Proposal:** For UObject paths that resolve to Blueprint-generated component templates, resolve the nearest parent component template by component variable name and use it as the default/reset base. Fall back to the class CDO only when no parent template exists. Response metadata should distinguish `defaultSource:"parent_template"` from `defaultSource:"class_cdo"` so callers know what reset means.

## History
- `#1-initial-geoscan-repro` `OPEN` reporter — During Geoscan prop repair, `property.reset` existed but could not reproduce the Blueprint reset arrow for inherited component `ComponentTags`: the live property default was `[]` from `StaticMeshComponent` rather than the parent `HELIOS_BP` template tags. Resetting would have removed the tags needed by `UDronePropellerComponent`; Python was required to set the child template explicitly.
- `#2-parent-template-default-source` `IN-REVIEW` developer — `UtilityPropertyHandler.cpp` now resolves inherited Blueprint component template defaults from the nearest parent template for `property.reset`, `property.get`, and `property.list`, emits `defaultSource`, and `TestPropertyComponentParentDefault.cpp` adds `FPropertyInheritedComponentParentTemplateDefaultTest` covering parent-tag list/reset behavior.
- `#3-verify-parent-template-default` `DONE` tester — Verified: `property.list` on `/App/HELIOS/Drones/Atlas/B_Geoscan801.B_Geoscan801_C:Motors: FR Move_GEN_VARIABLE` returns `ComponentTags` with `defaultSource:"parent_template"`, `defaultValue:["Prop_FR_Spin"]` (matches parent HELIOS_BP template, not empty class CDO), `value:["Prop_FR_Spin"]`, `isOverridden:false`, `hasDefaultValue:true` — exactly the parent-template inheritance behavior described in the proposal.
