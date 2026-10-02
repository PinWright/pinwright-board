---
id: F-graph-layout-comment-fit
title: "Graph layout ignores comment boxes: record comment membership before re-layout and re-fit each comment around its members afterwards; BPIR's comment-box emission path is dead code"
status: IN-REVIEW
severity: Medium
category: feature
tags: [layout, comments, blueprint, material, bpir, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Graph layout should keep comment boxes around their members

**Today:**
- No PinWright layout path accounts for comment boxes.
- In BPIR's post-compile layout, a pre-existing `UEdGraphNode_Comment` is just
  another obstacle, sized by the generic node estimator rather than its
  `NodeWidth` / `NodeHeight`. Its members are not tracked.
- Once a re-layout moves nodes, comments no longer enclose what they described.
- Material comments (`UMaterialExpressionComment`) are ignored by every material
  layout.
- BPIR never creates comment boxes. `FCodeNodeEmitter::CreateCommentBox` and
  `FinalizeCommentBoxes` (`Source/PinWright/Private/Compiler/CodeNodeEmitter.cpp:1092-1140`)
  have no callers. This is pre-existing dead code: wire it up or delete it.

**What it should do** (as a pass inside `F-graph-layout-core`):

1. Before layout, record each comment's members: in-scope nodes whose rects lie
   inside the comment rect. Nested comments form a tree.
2. Lay out without moving nodes into or out of a comment.
3. Re-fit each comment around its members' new bounds plus padding, including the
   title bar. Nested comments are fitted inner first.
4. Push packed trees apart so re-fitted comments do not overlap nodes outside
   them.
5. Report `commentsRefit[]` in the auto-layout verbs' response.

**Acceptance:** on a K2 fixture with one comment around a 3-node sub-chain and a
nested comment inside it:
- after a layout that moves those nodes, each comment encloses exactly its
  recorded members;
- no non-member node lies inside a comment rect;
- undo restores both comment rects.

## Severity justification

**Medium.** Comment boxes are how humans structure large graphs. A re-layout that
detaches them makes a hand-authored graph worse. Nothing is lost.

## History
- `#1-initial-spec` `OPEN` reporter — Gap analysis 2026-09-30: no layout path handles comments (existing comments are treated as estimator-sized obstacles; BPIR's CreateCommentBox/FinalizeCommentBoxes at CodeNodeEmitter.cpp:1092-1140 have no callers). Specify membership capture + re-fit as a F-graph-layout-core pass.
- `#2-comment-refit-implemented` `IN-REVIEW` developer — blocker F-graph-layout-core is DONE (blockedBy dropped). Comments are now a core pass (`Layout/PwGraphLayout.{h,cpp}`): `FLayoutComment {Key, Position, Size, TitleHeight, Members, Parent, bRefit}` on `FLayoutGraph::Comments`, `FSpacing::CommentPad` (16), `FArrangeReport::CommentsRefit`, and `RecordCommentMembers` (a node is a member when its rect lies inside the comment rect; a movable node at the origin is unplaced and joins none; a comment nests in the smallest comment containing it, equal rects by Key). (1) A comment with no movable member is a fixed obstacle (its rect joins the occupied set; movable nodes stay out). (2) A comment with a movable member reserves its frame when its first member is placed: X = the members' range (final before Y), height relative to that member = nested title bars + pads, or what the previous pass measured; non-members (nodes and disjoint comments) stay out of the reservation, members and nested comments ignore it, so nothing moves into or out of a comment. (3) After Y each frame is fitted around members + nested frames (pad, title bar above), inner first; a frame that outgrew its reservation widens it and Y re-runs (max 4 passes). (4) Commit writes the fitted rect. Adapters: `PwGraphLayoutEdGraph` (K2/anim/state machine) turns `UEdGraphNode_Comment`s into model comments (`CommentTitleHeight(FontSize)` = 1.4·FontSize+14, from the comment widget's paddings) and writes re-fitted rects through `Modify()` (floor/ceil to whole pixels, only when changed; `CommentsRefit` counts writes); `PwGraphLayoutMaterial` does the same for `GetEditorComments()` (`UMaterialExpressionComment` SizeX/SizeY). `material.authoring.auto_layout` (`Handlers/Material/MaterialAuthoringHandler.cpp`, the other agent's movedCount/moved[]/FScopedTransaction contract kept) now reports `commentsRefit[{nodeId, from{x,y,w,h}, to}]` read back from the comments and `sizeSource {measured, estimated}`; the transaction is kept when only comments changed. BPIR dead code deleted: `FCodeNodeEmitter::CreateCommentBox` / `FinalizeCommentBoxes` and the now-unused include/forward decl (`Compiler/CodeNodeEmitter.{h,cpp}`). Tests: `PinWright.layout.core.CommentRefitNested` (outer comment around Branch→B→C, nested around B, a data node feeding B from outside, a fixed node only the grown frame reaches; members enclosed, non-members out, both title bars, chain wires horizontal, re-recorded membership identical, second pass moves 0 / refits 0, shuffle-invariant), `PinWright.layout.core.CommentWithFixedMembersIsObstacle` (comment around a fixed node keeps its rect, chain avoids it; probe proves the chain would land inside without it), `PinWright.layout.blueprint.CommentsRefitAroundMovedMembers` (the acceptance K2 fixture: Event→Print→Branch→Print→Print, outer comment around the 3-node sub-chain, nested around the Branch, scattered start; after the move each comment encloses exactly its recorded members, no other node touches either rect, inner inside outer, second pass refits 0, one undo restores both comment rects and every position), `PinWright.material.authoring.auto_layout.CommentAroundPositionedNodesIsObstacle` (moved constant stays out of a comment around a positioned one, commentsRefit [] and sizeSource present). Offline harness: both core tests pass; removing the obstacle/reservation turns both red, disabling the reservation growth turns CommentRefitNested red. Docs: `docs/bpir-compiler-internals.md` (step 6, limitations), `docs/bpir-test-matrix.md` §2.6, wiki-src `material.authoring.md` auto_layout, CHANGELOG. Filter: `PinWright.layout+PinWright.material.authoring.auto_layout`. Known limit (documented): a non-member placed between two members before the frame is known can still end inside if 4 passes do not settle it. Every changed/new TU compile-checked with UBT -SingleFile (Linux, UE 5.8): all succeed. Not yet run in an editor (manager owns the build/test slot).
- `#3-fixture-clear-of-default-events` `IN-REVIEW` developer — Suite run: `layout.blueprint.CommentsRefitAroundMovedMembers` failed 'the outer comment records the 3-node sub-chain'. Root cause was the fixture, not the pass: a new Actor Blueprint's event graph carries three fixed default event nodes at x=0, y=0 / ~200 / ~400 (`FKismetEditorUtilities::AddDefaultEventNode`, Kismet2.cpp:549-710), and the outer comment (-240..1260, 200..1300) enclosed them, so membership correctly recorded more than 3 nodes. Fixture moved by (+4000, +4000) (`Tests/Layout/TestPwGraphLayoutBlueprintComments.cpp`); the assertion is unchanged, and a failing fixture check now names the recorded members (AddInfo before the return).
