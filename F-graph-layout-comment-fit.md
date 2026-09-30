---
id: F-graph-layout-comment-fit
title: "Graph layout ignores comment boxes: record comment membership before re-layout and re-fit each comment around its members afterwards; BPIR's comment-box emission path is dead code"
status: OPEN
severity: Medium
category: feature
tags: [layout, comments, blueprint, material, bpir, gap-analysis-2026-09-30]
blockedBy: [F-graph-layout-core]
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
