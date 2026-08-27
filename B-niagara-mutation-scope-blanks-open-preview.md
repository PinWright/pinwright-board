---
id: B-niagara-mutation-scope-blanks-open-preview
title: "BeginEmitterMutationScope kills system instances including the open toolkit's preview, and no response mentions it"
status: OPEN
severity: Medium
category: bug
tags: [niagara, BeginEmitterMutationScope, kill-system-instances, shared-editor, preview, undisclosed]
encounters: 1
lastSeen: 2026-08-28
---

# Every advanced-edit verb blanks somebody else's preview viewport

`BeginEmitterMutationScope` calls `KillSystemInstances`, which destroys **every** live
`FNiagaraSystemInstance` for the asset — including the one backing an open Niagara toolkit's preview
viewport.

So any `niagara.*` advanced-edit verb blanks the preview of anyone who has that asset open. The
preview does not come back on its own for a component that is not auto-activating.

Not a crash, and the kill itself is correct — it is what prevents the VectorVM assert
(`B-niagara-compile-while-live-component-vectorvm-assert`). The defect is that nothing in any response
says it happened, so from the other side the viewport just goes empty with no cause.

**Fix:** disclose it. `E-niagara-mutation-result-no-quiesced-count` already proposes a
`quiescedInstances` count on `MakeMutationResult`; this is the same field and the same argument, from
the shared-editor side rather than the response-honesty side. Worth resolving the two together.

## History
- `#1-found-while-classifying-the-guard` `OPEN` reporter — Found by the agent classifying Niagara
  mutators for the editor-open guard. Source reading, not reproduced.
