---
id: B-rpc-design-says-property-set-bare-posteditchange
title: "rpc-design.md §5d still says property.set and container.* fire a bare PostEditChange(); they emit a named event since B-property-set-container-empty-change-event"
status: OPEN
severity: Low
category: bug
tags: [docs, rpc-design, property, container, notification]
encounters: 1
rice: [1, 2, 1, 1]
priority: 17
---

# rpc-design.md claims property.set still fires an empty change event

`docs/rpc-design.md` (the "A bare `PostEditChange()` is not the notification" bullet, line ~214) ends: "that is what `property.set` and the whole `container.*` family still do (`UtilityPropertyHandler.cpp:1129` and siblings)". That has been false since `B-property-set-container-empty-change-event` (DONE): every reflected mutator in `Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp` routes through `NotifyReflectedPropertyChanged` (`:392`), which emits a named non-chain event via `PinWright::NotifyPropertyChanged`; the file's own comment at `:356` says it "replaced a bare RootObject->PostEditChange()". `grep -n "PostEditChange()"` in that file finds only comments. The cited `:1129` points at unrelated code.

A contributor reading the design doc is told the plugin's flagship property writer is the bad example, and may "fix" it again or copy the doc's framing into new verbs.

**Fix:** drop the trailing clause (or rewrite it in the past tense, naming `NotifyReflectedPropertyChanged` as the fix and the ticket that landed it). Docs-only.

## History
- `#1-stale-design-doc-clause` `OPEN` reviewer — Found while reviewing `B-property-set-wiki-construction-rerun` (whose developer edited the adjacent "Notify without PreEditChange" bullet). Verified at plugin 7230b41d: `git show HEAD:Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp | grep -n "PostEditChange()"` returns only the comment lines `:348` and `:1234`; the clause in `docs/rpc-design.md` is unchanged.
