---
id: B-resolved-subject-move-drops-visibility-setter
title: "FResolvedSubject's hand-written move constructor and move assignment do not move VisibilitySetter, so a moved subject silently loses subject-coverage measurement"
status: OPEN
severity: Low
category: bug
tags: [render, capture, capture-subject, coverage, move-semantics, latent]
encounters: 1
lastSeen: 2026-10-02T01:55:00-05:00
---

# A moved FResolvedSubject loses its VisibilitySetter

## What happens

`FResolvedSubject(FResolvedSubject&&)` and `operator=(FResolvedSubject&&)` in
`Source/PinWright/Private/Handlers/Render/CaptureSubject.cpp` list every member by hand and omit
`VisibilitySetter` (declared in `CaptureSubject.h`). The moved-to subject has an unbound setter, so
`poseSet.coverageReferenceShots` and every `subjectCoverage` figure would be absent for a kind that
can in fact be hidden.

## Why it matters

Latent today: no call site moves an `FResolvedSubject` (grep for `MoveTemp(Resolved` finds none).
Severity Low. The hand-written member list is the trap: any field added to the struct must also be
added to both move operations, and nothing checks that.

## What should happen

Move `VisibilitySetter` in both operations (one line each), ideally with a small test that moves a
subject carrying every TFunction member and asserts each is still bound.

## History
- `#1-filed-during-pose-checkpoint` `OPEN` reporter — Found while adding `StateCheckpointer` / `StateCheckpointUnavailableReason` to `FResolvedSubject` for `F-pose-repeatability-checkpoint`; those two fields were added to both move operations, `VisibilitySetter` was left as found. Source-only; no runtime repro.
