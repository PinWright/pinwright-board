---
id: B-drive-click-changed-false-with-shape-diff
title: "drive.click returns outcome no_change_within_budget / changed:false while its own diff lists dozens of appeared and disappeared elements"
status: IN-REVIEW
severity: Medium
category: bug
tags: [drive, drive.click, settle, outcome, diff, inconsistent-result]
encounters: 1
lastSeen: 2026-09-29T16:25:00Z
---

# The outcome and the diff of one drive.click response disagree

`drive.click {handle:".../W_CommonTrackControls/BecomeSpectatorButton/SCommonButton", instance_name:"W_OverallUILayout", os_input:true, settle_budget_ms:500}`
on a PDS race-lobby client HUD returned:

```
{"outcome":"no_change_within_budget","changed":false,"settled":false,
 "diff":{"appeared_count":48, "disappeared_count":48, "appeared_sample":[".../W_RaceEndLeaderboardPlayer_C_504/..."], "disappeared_sample":[".../W_RaceEndLeaderboardPlayer_C_496/..."], ...}}
```

The HUD's opponents list recreates its row widgets every second, so element paths really did appear and
disappear during the settle window. That is a shape change, which the settle fingerprint is meant to
catch. The response still says `changed:false`, and `no_change_within_budget` suggests the click did
nothing. It did land: the server logged `Player B_DronePlayerController_C_3 set as SPECTATOR`. A caller
that trusts `outcome` concludes the click failed. Another response in the same session had
`changed:true` with a similar 24/24 diff, so the result depends on timing.

**Expected:** `changed` is true whenever the returned diff is non-empty, or the diff is omitted when
the settle logic judged the churn to be noise. The two fields should never contradict each other.

## History
- `#1-lobby-row-churn` `OPEN` reporter - UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`, PDS listen PIE with 3 clients. Seen on several os_input clicks on the client's Become Spectator / Become Pilot buttons. Verified against the server log instead. Cheap.
- `#2-diff-wins` `IN-REVIEW` developer - Root cause: the settle loop's per-tick fingerprint is shape-only (type + rect, handle-blind), so rows recreated in place under new handles never move it, while the response diff is keyed by handle (and value). New `FDriveActionCommon::WriteSettleResult` writes the shared `{outcome, changed, settled, condition_met, elapsed_ms, ticks, diff}` fields for every action verb: a non-empty diff forces `changed:true`, and a quiet `no_change_within_budget` with a non-empty diff becomes `settled_changed` / `settled:true` (the shape held the whole quiet budget). Empty diff keeps the quiet outcome. Also covers the value-only `drive.type` case the wiki used to document as `changed:false`. Game/editor (`RunAction`) and web (`StartWebSettle`) responses both route through it. Files: `Source/PinWright/Private/Handlers/Drive/DriveActionCommon.{h,cpp}`, `Handlers/Drive/DriveWebHandlers.cpp`, `Tests/Drive/TestDriveSettleDriver.cpp`, `docs/wiki-src/drive.md`. Test: `PinWright.drive.settledriver.RowChurnAgreesWithDiff` (fails if the reconciliation is removed). Not addressed: `changed:true` with an empty diff (UI moved and came back) is left as is and documented.
