---
id: B-import-xml-add-drops-sibling-roots
title: "`widget.import_xml` mode `add` with two top-level sibling elements imports only the first and still reports success"
status: IN-REVIEW
severity: High
category: bug
tags: [widget, import-xml, add-mode, silent-drop, xml-parse, silent-false-success]
encounters: 1
lastSeen: 2026-09-23T19:40:00Z
---

# The second sibling vanishes without a warning

`widget.import_xml {widgetPath: /App/App/UI/LobbyAndMenu/Popups/W_AppSchoolNameLogin, targetName: "Ov_ErrorOk",
mode: "add", xml: "<Image name=\"img_ErrorOkBG\" .../><CommonTextBlock name=\"tb_ErrorOk\" .../>"}` returned
`success: true`. Afterwards `widget.export_xml` of `SB_ErrorOk` showed `Ov_ErrorOk` with
`[ErrorOkButton, img_ErrorOkBG]` only: `tb_ErrorOk` was never created, and the response did not mention it.
A second call adding just `<CommonTextBlock name="tb_ErrorOk" .../>` worked.

`add` mode is the natural way to append several children to one panel, and the markup is two well-formed
elements; the parser evidently takes the first root node and ignores the rest.

**Workaround:** one `import_xml add` call per sibling, or wrap siblings in a throwaway panel.

**Fix (proposed):** in add mode, iterate every top-level element (parse the payload wrapped in a synthetic
root); otherwise refuse with `INVALID_ARGUMENT: multiple top-level elements` so the caller knows. Report
`widgetsCreated` per element either way.

## History
- `#1-second-sibling-dropped` `OPEN` reporter - Filed from a UMG pass on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Adding `img_ErrorOkBG` + `tb_ErrorOk` under `Ov_ErrorOk` in one `add` call created only the Image; the missing label showed up as a blank OK button in the PIE screenshot. Needed a second import call.
- `#2-parse-inside-synthetic-root` `IN-REVIEW` developer - Root cause: `FXmlFile::CreateRootNode` returns as soon as the first top-level element closes, so every later sibling was never parsed and the handler never saw it. `Handlers/UI/WidgetXmlImportHandler.cpp` now parses the payload wrapped in a synthetic `<PinWrightImportRoot>` (wrapper tags on their own lines so FXmlFile's line-based `<?xml`/`<!DOCTYPE` cull still applies) and treats its children as the top-level elements. `add` validates all of them first (duplicate names across siblings included), then builds each in document order under the target; rollback now removes every child the import added. `replace` (full or targeted) with more than one top-level element is refused with `INVALID_ARGUMENT` instead of dropping the rest. The response adds `rootWidgets` (names of the top-level widgets built); `widget_count` counts all of them. Baseline on unmodified code (live editor, scratch WBP): `add` of `<Image name="SiblingA"/><TextBlock name="SiblingB"/>` returned `success:true, widget_count:1` and export showed no `SiblingB`. After the fix the same call returned `widget_count:2, rootWidgets:["SiblingA","SiblingB"]` and export showed both, in order; a two-element `replace` returned `INVALID_ARGUMENT`. Tests: `PinWright.widget.import_xml.AddImportsEveryTopLevelSibling`, `PinWright.widget.import_xml.ReplaceRejectsMultipleTopLevelElements` (new file `Tests/WidgetXml/TestWidgetXmlImportPlacement.cpp`). Scoped run `PinWright.widget.import_xml+PinWright.widget.export_xml.RoundTrip+PinWright.widget.bind_event.XmlImportedButtonSiblingBinding+PinWright.widget.required_names.widget_import_xml`: 14/14, `check_suite_log` COMPLETED_CLEAN. Wiki: `docs/wiki-src/widget.md` `### widget.import_xml` Modes updated. Plugin commit `a1e98be5`.
