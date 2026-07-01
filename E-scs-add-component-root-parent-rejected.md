---
id: E-scs-add-component-root-parent-rejected
title: "blueprint.scs.add_component rejects parentComponentName='RootComponent' (resolver only matches SCS nodes, not the native root)"
status: OPEN
severity: Low
category: ergonomic
tags: [scs, blueprint, add_component, root, discovery, error-diagnostics]
---

# `blueprint.scs.add_component` rejects `parentComponentName: "RootComponent"`

Passing the obvious `parentComponentName: "RootComponent"` (the natural first
guess for "attach under the root") fails with `SCS_ERROR Parent component not
found: RootComponent`. `FSCSHandlers::AddSCSComponent` resolves the parent name
by walking **only** `SCS->GetAllNodes()` and matching `GetVariableName()`
(`PinWright_SCSHandlers.cpp:668-687`, error emitted at `:683`); the actor's
native/inherited `RootComponent` is a CDO template, not an SCS node, so it is
never a match. The non-success result is wrapped with the default `SCS_ERROR`
code by `SendSCSResult` (`SCSHandler.cpp:42`). The error names neither the valid
SCS node options nor the omit-to-attach-at-root path, so the natural first guess
errors with a non-diagnostic message.

Distinct from the IN-REVIEW sibling `E-scs-add-component-root-promotion-undocumented`,
which documents what UE does on the *omit* path (promote first scene component,
reparent the rest). This ticket is the inverse: explicitly *naming* the root is
rejected, not silently reparented.

**Workaround:** omit `parentComponentName` (or pass empty) — that path attaches
under the native root correctly (confirmed by `blueprint.scs.get`). The param
doc (`SCSHandler.cpp:85`) does already say "Existing SCS node name … omit … to
attach as a child of the root," so a doc-reading caller can avoid it.
**Fix:** resolve `"RootComponent"` (and the actual native root component name) to
the attach-at-root path instead of erroring; or make the error name the valid
parent options so the recovery is obvious.

## History
- `#1-initial-repro` `OPEN` reporter — Verified against source: `AddSCSComponent` resolves `parentComponentName` only over `SCS->GetAllNodes()` (`PinWright_SCSHandlers.cpp:668-687`); the native root is not an SCS node, so `parentComponentName:"RootComponent"` returns `SCS_ERROR Parent component not found: RootComponent` (wrap `SCSHandler.cpp:42`). Session evidence: across ~32 gate migrations, `add_component {…, parentComponentName:"RootComponent"}` errored; omitting it parented under the root correctly. Filed E-/Low (not B-): the omit path works and the param doc hints at it, so this is a discoverability/error-diagnostics gap, not a functional defect. Distinct from `E-scs-add-component-root-promotion-undocumented` (omit-path promotion docs).
