---
id: B-source-recompile-state-loss
title: "Source recompiles silently discard typed-RPC asset edits that the source cannot express"
status: IN-REVIEW
severity: Critical
category: bug
tags: [pwsource, pwskel, pwanim, pwmodel, recompile, data-loss]
---

# Source recompiles silently discard asset state outside the source

PinWright's source compilers generate one asset from one source file, but a later typed
RPC mutation can add asset state that the source does not describe. Recompiling the same
source then replaces or resets that state and reports success. The provenance stamp proves
that the target came from the source, but previously stored no semantic generated-state
baseline and no compiler checked live state against it before rebuilding.

The measured skeleton case included a preview mesh, non-default per-bone translation
retargeting, and animation curve metadata. A source file with none of those fields rebuilt
successfully and silently lost them. The same failure class applied to model and animation
source compilers whenever an independent mutator changed a generated asset.

**Impact:** a normal successful recompile can irreversibly destroy authored asset data.

**Fix:** record the generated semantic state in provenance, compare baseline/current/desired
state before any destructive rebuild, and reject unmanaged changes by default with
`PWSRC_RECOMPILE_UNMANAGED_STATE`. An explicit `overwrite=true` acknowledges the loss and
changes that same diagnostic to a warning. Never carry unspecified live state forward.
The skeleton source format now owns preview mesh, per-bone translation retargeting, and
animation curve metadata. Unsupported state remains guarded and can only be discarded by
an explicit overwrite.

## History
- `#1-silent-recompile-loss` `OPEN` reporter — Reproduced a generated skeleton carrying a preview mesh, per-bone retargeting, and animation curve metadata that its source could not express; recompiling silently reset the state and returned success. Source inspection found no overwrite, clobber, preservation, out-of-band, or unmanaged-state diagnostic across the source compiler family.
- `#2-guard-source-recompile` `IN-REVIEW` developer — Added a shared three-way provenance guard to the skeleton, animation, static-model, and skeletal-model rebuild paths. Added `preview_mesh`, per-bone `retarget`, and `curve` metadata to the skeleton source format. Added failure-direction tests that require the exact diagnostic on unmanaged live state, require a clean recompile when source describes it, and require explicit overwrite to warn before clearing unsupported state. Updated source-format and wiki documentation.
