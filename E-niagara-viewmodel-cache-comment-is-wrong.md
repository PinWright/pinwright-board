---
id: E-niagara-viewmodel-cache-comment-is-wrong
title: "NiagaraSystemViewModelCache.h claims it reuses an open editor's view model; its own .cpp says it cannot"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, docs, comment, NiagaraSystemViewModelCache, misleading]
encounters: 1
lastSeen: 2026-08-28
---

# A header comment that contradicts its implementation

`NiagaraSystemViewModelCache.h`'s doc comment states that it reuses an editor tab's live
`FNiagaraSystemViewModel` when one is present.

Its own `.cpp` says the opposite, and explains why: `TNiagaraViewModelManager::GetExistingViewModelForObject`
is not exported, so the cache **always** allocates a second `FNiagaraSystemViewModel` over the same
`UNiagaraSystem`.

The consequence of trusting the header is a wrong mental model at exactly the wrong moment: a reader
concludes that edits made through the cache are shared with the open toolkit, which is the assumption
behind several of the shared-editor defects in this batch.

Comment-only fix.

## History
- `#1-found-while-classifying-the-guard` `OPEN` reporter — Found by the agent classifying Niagara
  mutators for the editor-open guard, which read both the header and the implementation.
