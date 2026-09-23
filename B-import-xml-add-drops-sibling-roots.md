---
id: B-import-xml-add-drops-sibling-roots
title: "`widget.import_xml` mode `add` with two top-level sibling elements imports only the first and still reports success"
status: OPEN
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
