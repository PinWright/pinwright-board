---
id: E-source-control-status-isunchanged-always-false-git
title: "source_control.status clean-state flags misleading under Git — every tracked file reads isCheckedOut:true, so derived isUnchanged is pinned false even for pristine files"
status: OPEN
severity: Medium
category: ergonomic
tags: [source-control, git, status, source-control-status-clean-state]
encounters: 1
lastSeen: 2026-07-02T10:16:46.4714927+03:00
---

# `source_control.status` cannot answer "is this file clean?" under the Git provider

`source_control.status` returns a fixed set of boolean flags per path
(`isCheckedOut`, `isAdded`, `isDeleted`, `isModified`, `isUnchanged`,
`canCheckOut`, `isUnknown`). Two of them — `isCheckedOut` and the PinWright-
derived `isUnchanged` — are **misleading for genuinely-clean files** when the
active provider is Git, which defeats the most common reason to call `status`:
"show me at a glance what's modified, what's new, and what's clean."

## What's wrong

For every **tracked, unmodified** file, the Git provider reports
`IsCheckedOut() == true` (git-without-locks treats all tracked files as
always-editable / "checked out"; the caller never checked anything out). That
value is passed straight through:

`Plugins/PinWright/Source/PinWright/Private/Handlers/SourceControl/SourceControlHandler.cpp:248`
```cpp
FileObj->SetBoolField(TEXT("isCheckedOut"), State->IsCheckedOut());
```

Then PinWright **derives** `isUnchanged` by folding `!IsCheckedOut()` into the
composite:

`Plugins/PinWright/Source/PinWright/Private/Handlers/SourceControl/SourceControlHandler.cpp:252`
```cpp
FileObj->SetBoolField(TEXT("isUnchanged"), !State->IsModified() && !State->IsAdded() && !State->IsDeleted() && !State->IsCheckedOut());
```

Because `IsCheckedOut()` is true for *every* tracked git file, `isUnchanged`
is **pinned to false for all of them — clean or dirty alike**. The one field
whose name promises "this file is unchanged from the depot" can never say
`true` under Git, so it is useless (and actively misleading) for its stated
purpose. A caller who reads `isUnchanged` to mean "clean" is lied to on every
git file; a caller who reads `isCheckedOut:true` may believe they hold a
pending checkout/lock they never took.

The only field that *does* correctly distinguish clean from dirty here is
`isModified` (false for the clean files) — but nothing documents that
`isUnchanged` is unreliable and `isModified` is the field to trust. This is
provider-inconsistent: under Perforce a clean unopened file is *not* checked
out, so `isUnchanged` computes to `true` correctly; the composite only breaks
for the lockless-Git model.

This is an ergonomic/readback defect, not a crash or a bad-input rejection:
the call succeeds and returns schema-valid JSON. The friction is that the
returned clean-state signal contradicts reality.

## What it should do

- Rework the `isUnchanged` derivation so it does not fold in `!IsCheckedOut()`
  (e.g. derive "clean" from `!IsModified() && !IsAdded() && !IsDeleted()`, or
  use the provider's own working-copy "unchanged" state), so a pristine tracked
  git file reports `isUnchanged:true`.
- And/or surface the provider's raw working-copy state string (e.g.
  `Unchanged` / `Modified` / `Added` / `CheckedOut`) so callers aren't forced
  to reverse-engineer clean-vs-dirty from booleans whose meaning shifts per
  provider.
- At minimum, document (wiki overlay) that under Git every tracked file reads
  `isCheckedOut:true` and `isUnchanged:false`, and that `isModified` is the
  reliable clean-signal.

## Verbatim repro (replayed at HEAD)

Ground truth (host git): all four assets are tracked and clean —
`git status --short` prints nothing; `git ls-files` lists them.

`source_control.get_provider` `{}` →
`{"providerName":"Git","isEnabled":true,"isAvailable":true}`

`source_control.status`
`{"assetPaths":["/Game/Characters/Echo/Animations/Idle","/Game/Characters/Echo/Animations/Jog_Fwd","/Game/Characters/Echo/Animations/Idle_Long","/Game/Characters/Echo/BP_EchoHair"]}`
→ each file (all clean/unmodified on disk):
```
{"path":"/Game/Characters/Echo/Animations/Idle", ...,
 "isCheckedOut":true,"isAdded":false,"isDeleted":false,
 "isModified":false,"isUnchanged":false,"canCheckOut":false,"isUnknown":false}
```
i.e. a provably-clean file reports `isCheckedOut:true` and `isUnchanged:false`.

Related confusion this caused downstream: because `status` shows everything as
`isCheckedOut:true`/`isUnchanged:false`, a caller who then runs
`source_control.mark_for_add` on already-tracked files gets `{success:true,
resultCode:1,count:2}` (a correct git no-op) yet sees the follow-up status
"unchanged" (`isCheckedOut:true`/`isAdded:false`), and cannot tell from the
booleans whether anything happened. (`mark_for_add` itself is behaving
correctly — the ambiguity is the `status` clean-state readback.)

severity rationale: impact=misleading-readback (a clean file reads as
checked-out and not-unchanged; the caller trusts a lie about clean vs. dirty) ×
reach=rare (source-control path) -> Medium.

## History
- `#1-initial-repro` `OPEN` reporter — Realism task ("source-control sanity pass on the Echo animation set: confirm provider, refresh status to see modified/new/clean, log, mark new files for add, revert an experiment"). Replay-confirmed at HEAD against the Git provider: all four Echo assets are tracked+clean per host git, yet `source_control.status` reports each `isCheckedOut:true`/`isUnchanged:false`. Root cause is PinWright's `isUnchanged` composite at `SourceControlHandler.cpp:252` folding in `!State->IsCheckedOut()`; the Git provider reports every tracked file checked-out (`SourceControlHandler.cpp:248` passes it through), so `isUnchanged` is pinned false for pristine files and the field cannot answer "is this clean?". `isModified` is the only reliable clean-signal but that's undocumented. Culprit reassigned from the attempt's suspected `mark_for_add` (which is a correct no-op on already-tracked files) to `source_control.status`.
