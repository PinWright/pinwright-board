---
id: B-niagara-refused-edit-dirties-package
title: "Every error return inside set_module_input's transaction happens after Graph->Modify(), so a refused call dirties the package with no change"
status: OPEN
severity: Medium
category: bug
tags: [niagara, set_module_input, transaction, package-dirty, refused-write, side-effect]
encounters: 1
lastSeen: 2026-08-27
---

# A refusal still marks the asset dirty

In `niagara.set_module_input`, every error return inside the `FScopedTransaction` block happens
*after* `ModifyResolvedTarget(Target)` and `Target.Graph->Modify()`. So a call that is refused --
`INVALID_STACK`, `PARAMETER_TYPE_MISMATCH`, `UNSUPPORTED_INPUT_VALUE`, and now
`MODULE_INPUT_OVERRIDE_LINKED` -- leaves the package dirty having changed nothing.

The consequence is not cosmetic in a shared editor: a later legitimate save of that package persists
whatever else was pending, and a caller checking "is this asset dirty" to decide whether its own write
landed gets a false positive from somebody else's rejected call.

**Fix:** move the guards ahead of the transaction. That fixes all four refusal paths at once and is
the structural version rather than one `Modify()` audit per error return. It restructures the shared
handler prologue, which is why the agent that hit it did not do it -- two other agents were editing
that function at the time.

## History
- `#1-pre-existing-made-more-reachable` `OPEN` reporter -- Recorded by the agent fixing
  `B-niagara-literal-over-linked-override-pin`, whose new refusal joins the three existing ones on the
  same path. Pre-existing, not introduced by that fix. Source-level claim.
