---
id: E-drive-settle-changed-false-on-value-edit
title: "drive action verbs report outcome=no_change_within_budget / changed:false after a SUCCESSFUL text edit — the settle change-detector's fingerprint is value-blind, contradicting drive.md's 'changed is honest / no observable UI change' guarantee"
status: OPEN
severity: Medium
category: ergonomic
tags: [drive, settle, drive.type, settle-change-detector-value-blind, no-change-misreport]
encounters: 1
lastSeen: 2026-07-04T15:47:49.8351918+03:00
---

# drive settle reports "no change" after a successful text edit (value-blind fingerprint)

## What's wrong
Every `drive.*` action verb settles by polling a UI **fingerprint** and reporting
`changed` / `outcome` from whether that fingerprint diverged from the pre-action
baseline. The fingerprint (`FDriveChangeDetector::Compute`) hashes only each visible
element's `Type` + whole-pixel rect, and its own header states content is
**intentionally** excluded. An editable input's live typed `value` is therefore NOT
part of the fingerprint.

Consequence: `drive.type "Aria Vance"` into a fixed-geometry `EditableTextBox`
changes only that element's `value` — its `Type`, rect, and visibility are unchanged
— so the baseline and post-action fingerprints are byte-identical. `bChangedSinceBaseline`
is never true, and the settle loop reports `outcome:"no_change_within_budget"`,
`changed:false` even though the edit fully succeeded (the same session's
`drive.observe` surfaces `value=="Aria Vance"` and `drive.expect text_equals` returns
`met:true`).

This directly contradicts the wiki's explicit honesty guarantee. `drive.md`
"Timing and settle" (line 29) promises:

> The response's `changed` boolean is honest: `changed:false` with outcome
> `no_change_within_budget` means the action produced no observable UI change, not a
> faked success.

But typing text IS an observable UI change — it is what appears on screen and what
`drive.observe` reports as the element's `value`. So on the single most common
`drive.type` path (typing into a text field), `changed:false` /
`no_change_within_budget` is a systematic misreport: an agent that trusts `changed`
(as the wiki instructs) reads it as "my type didn't land" and may retry or abort a
type that actually worked.

The same value-blindness affects the structural `diff` reported alongside: the
per-handle `changed` set is computed by `SignatureEquals` (Type + bVisible + rounded
rect only), so `changed_count` is also 0 for a pure text edit even though the tool
already carries the new `value` per element (added by the earlier
`B-drive-editable-text-reads-hint-not-value` fix). The tool observes the value
change but its change-detection ignores it.

This is a FAMILY-level issue in the shared settle driver, not a single verb. All
action verbs route through `FDriveActionCommon` -> `FDriveSettleDriver` and inherit
the value-blind signal; any action whose only effect is an editable value change
(no layout/visibility change) misreports. Affected methods: `drive.type`,
`drive.key`, `drive.click`, `drive.scroll`, `drive.drag`, `drive.hover`. The
demonstrated and most-common case is `drive.type` (and `drive.key`) doing text entry.

## What it should do
Either make `changed` honest for value edits, or stop the wiki claiming it covers
"any observable UI change":
- Preferred: fold each element's `value` into the settle change signal — e.g. have
  `FDriveChangeDetector::Compute` include `value` in the digest, and/or
  `SignatureEquals` compare `Value` — so a successful `drive.type` yields
  `outcome:"settled_changed"` / `changed:true` and a `changed` diff entry for the
  edited handle. The value is already tracked per element (`FDriveElement.Value`), so
  the data is on hand.
- At minimum: correct `drive.md` (line 29) to say `changed`/`no_change_within_budget`
  reflect only a **structural/shape** change (element type/geometry/visibility), NOT
  content/value edits, and note that a successful text entry legitimately reports
  `no_change_within_budget` — so an agent verifies typed text via `drive.observe`
  `value` / `drive.expect text_equals` rather than trusting `changed`.

## Verbatim repro
Task: type a name into a live PIE HUD `EditableTextBox` and confirm it captured the
text.
1. `mcp__pinwright__call` method=`drive.type`
   args=`{surface:"game", handle:"<NameField EditableTextBox handle>", text:"Aria Vance"}`
   -> success, but settle result reports `outcome:"no_change_within_budget"`,
   `changed:false` (the typed text nonetheless landed).
2. `mcp__pinwright__call` method=`drive.observe` (game surface)
   -> the same NameField element carries `value:"Aria Vance"` — i.e. the tool sees the
   value change it just said didn't happen.
3. `mcp__pinwright__call` method=`drive.expect`
   args=`{condition:{type:"text_equals", target:"NameField", expected_text:"Aria Vance"}}`
   -> `met:true` — independently confirms the edit succeeded.

Attempt's own friction note, verbatim: "drive.type reported outcome
'no_change_within_budget'/changed:false even though the field correctly captured the
text ... the settle change-detector didn't register the value change, which is
momentarily confusing but not a failure."

(Live MCP re-issue was blocked this iteration — the host editor was down between
iterations, connection refused — so this repro is confirmed from the guilty source
chain below plus the attempt's structured call-log, which is deterministic: a
value-only edit cannot move the fingerprint.)

## Guilty source (ground truth, read verbatim)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveSettleDriver.cpp:50,64,66`
  — the change signal is derived purely from the value-blind fingerprint:
  `Baseline = FDriveChangeDetector::Compute(Initial);` (50)
  `const FDriveFingerprint Fingerprint = FDriveChangeDetector::Compute(Current);` (64)
  `const bool bChangedSinceBaseline = (Fingerprint != Baseline);` (66)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveFingerprint.h:12-16`
  — the fingerprint intentionally excludes content:
  "Two fingerprints compare equal when the visible element set has the same types, the
  same whole-pixel rects, and the same visible count. Handles, labels, focus, and
  enabled state are intentionally NOT part of the fingerprint - it answers 'did the UI
  change shape?', not 'did addressing or content change?'." (`value` likewise never
  enters `Compute`.)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveFingerprint.cpp:90-96`
  — `Compute` digests only `Element->Type` + rounded rect per visible element; `Value`
  is not read. `SignatureEquals` (`:63-66`) compares only `Type`, `bVisible`, and
  `RoundRect`, so the per-handle `Diff` (`:109-157`) also never flags a value-only edit.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveSettleDecision.cpp:82-85`
  — with `bHasChanged` never set, once the quiet budget elapses it returns
  `EDriveSettleOutcome::NoChangeWithinBudget` with `bChanged=false`.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveActionCommon.cpp:53`
  maps that enum to the wire string `no_change_within_budget`; `:273-274` write it as
  `outcome` and `changed` into every action verb's response.
- Contradicted doc: `Saved/PinWright/wiki/drive.md` "Timing and settle" (line 29) —
  the "changed boolean is honest ... no observable UI change" guarantee quoted above.

severity rationale: impact=misleading result field on a normal path (changed/outcome
report "no change" after a successful edit, contradicting the documented honesty
guarantee) but the operation works and the value round-trips via observe/expect, so no
data is lost and the failure is a safe-direction false-negative with a documented
workaround × reach=drive.type text entry is an every-session verb -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Filed: the shared drive settle change-detector
  is value-blind. `FDriveChangeDetector::Compute` hashes only Type + rect + visibility
  (`DriveFingerprint.h:12-16`, `DriveFingerprint.cpp:90-96`), and
  `DriveSettleDriver.cpp:66` derives the whole change signal from it, so a successful
  `drive.type` into a fixed-geometry `EditableTextBox` (value changes, geometry does
  not) yields `outcome:"no_change_within_budget"` / `changed:false` even though
  `drive.observe` surfaces the new `value` and `drive.expect text_equals` returns
  `met:true`. That contradicts `drive.md:29` ("changed is honest ... no observable UI
  change"). Distinct root cause from `B-drive-editable-text-reads-hint-not-value` /
  `E-drive-expect-editable-text-docs` (those were the hint-vs-value read-back in
  `DriveEditorChrome.cpp`/`DriveConditionEval.cpp`, now fixed — which is why observe/
  expect correctly show the value here). Family-level: affects every action verb via
  `FDriveSettleDriver`; demonstrated on `drive.type`. Live MCP re-issue blocked this
  iteration (editor down, connection refused); confirmed from the guilty source chain
  + the attempt's structured call-log.
