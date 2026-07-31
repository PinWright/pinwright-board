---
id: E-asset-dump-progress-notification
title: "Long asset dumps have no editor-visible progress or cancellation control"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [asset-dump, jobs, progress, editor-ui, cancel]
---

# Long asset dumps have no editor-visible progress or cancellation control

An MCP-triggered `asset.dump_folder` can run for many minutes while Unreal is
slow, but the editor exposes no visible indication that the dump is active.
Operators cannot see completed versus total assets, the current asset and phase,
or an estimated completion time without separately polling `system.job_status`.
There is also no editor-side cancellation control, even though the job registry
supports cancellation.

This makes a working dump look indistinguishable from an editor freeze and makes
the documented job-control path difficult to use when the MCP caller is no
longer visible.

**Fix:** show a nonmodal editor notification for interactive folder-dump jobs.
Include completed/total, percentage, estimated remaining time, current phase and
asset, and a Cancel button wired to the dump ticket's existing registry cancel
callback. Keep unattended runs UI-free and retain the machine-readable progress
fields for MCP callers.

## History
- `#1-no-visible-progress` `OPEN` reporter — A long `/App` dump made Unreal appear frozen because no editor UI showed completed/total assets, ETA, current work, or a cancellation control.
- `#2-editor-progress-notification` `IN-REVIEW` developer — Interactive MCP folder dumps now show a nonmodal Slate notification with completed/total, percent, ETA, phase, current asset, and a Cancel button wired to the job registry. Progress payloads expose the same counts and timing. The detail area is always two lines and middle-elides asset paths to 52 characters so the window height stays fixed; full paths remain in job status and `jobs.jsonl`.
