---
id: B-source-recompile-state-loss
title: "Source rebuilds silently discard asset state that the incoming source cannot express"
status: IN-REVIEW
severity: Critical
category: bug
tags: [pwsource, pwskel, pwanim, pwmodel, recompile, data-loss]
---

# Source rebuilds silently discard asset state outside the incoming source

PinWright's source compilers generate one asset from one source file, but a later typed
RPC mutation can add asset state that the source does not describe. Recompiling the same
source, or taking over the asset from a different stamped source, can then replace or reset
that state and report success. A same-source provenance stamp provides a generated-state
baseline; a takeover must instead compare the live asset directly with the incoming source.

The measured skeleton case included a preview mesh, non-default per-bone translation
retargeting, and animation curve metadata. A source file with none of those fields rebuilt
successfully and silently lost them. The same failure class applied to model and animation
source compilers whenever an independent mutator changed a generated asset.

**Impact:** a normal successful recompile can irreversibly destroy authored asset data.

**Fix:** record the generated semantic state in provenance, compare baseline/current/desired
state before any destructive rebuild, and reject unmanaged changes by default with
`PWSRC_RECOMPILE_UNMANAGED_STATE`. A takeover without `overwrite=true` is refused and names
the state that would be lost. An explicit overwrite permits the takeover but changes that
same diagnostic to a warning. Never carry unspecified live state forward.
The skeleton source format now owns preview mesh, per-bone translation retargeting, and
animation curve metadata. Unsupported state remains guarded and can only be discarded by
an explicit overwrite.

## History
- `#1-silent-recompile-loss` `OPEN` reporter — Reproduced a generated skeleton carrying a preview mesh, per-bone retargeting, and animation curve metadata that its source could not express; recompiling silently reset the state and returned success. Source inspection found no overwrite, clobber, preservation, out-of-band, or unmanaged-state diagnostic across the source compiler family.
- `#2-guard-source-recompile` `IN-REVIEW` developer — Added a shared three-way provenance guard to the skeleton, animation, static-model, and skeletal-model rebuild paths. Added `preview_mesh`, per-bone `retarget`, and `curve` metadata to the skeleton source format. Added failure-direction tests that require the exact diagnostic on unmanaged live state, require a clean recompile when source describes it, and require explicit overwrite to warn before clearing unsupported state. Updated source-format and wiki documentation.
- `#3-scoped-verification` `IN-REVIEW` developer — Full editor build completed with `Result: Succeeded`. Scoped automation for Model, infra, Skeleton, Animation, and the shared source contract completed 621/621 tests successfully with zero failures or skips and both queue-drain and test-exit markers. Ticket remains `IN-REVIEW` for independent tester disposition.
- `#4-takeover-gap-reproduced` `OPEN` reporter — Reproduced that all four guarded rebuild paths ran `PWSRC_RECOMPILE_UNMANAGED_STATE` only for a matching source stamp. Skeleton, animation, static-model, and skeletal-model takeovers therefore skipped the guard entirely; `overwrite=true` could silently discard live state, while a refusal did not name what would be lost.
- `#5-guard-takeover-paths` `IN-REVIEW` developer — Changed the shared guard and all four callers to check same-source rebuilds and takeovers. Takeovers compare current state directly with desired state, no-overwrite refusals include the named losses, and overwrite-permitted takeovers emit `PWSRC_RECOMPILE_UNMANAGED_STATE` as a warning while succeeding. Added failure-direction coverage for skeleton, animation, static-model, and skeletal-model paths. Full build: `Result: Succeeded`; scoped affected groups: 37/37 succeeded, zero failed, zero skipped. Ticket remains `IN-REVIEW` for independent tester disposition.
