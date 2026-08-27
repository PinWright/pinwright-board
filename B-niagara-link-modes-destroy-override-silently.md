---
id: B-niagara-link-modes-destroy-override-silently
title: "set_module_input's link and dynamicInput value modes destroy a pre-existing override with no opt-in and nothing in the response saying what they displaced"
status: OPEN
severity: High
category: bug
tags: [niagara, set_module_input, dynamic-input, linked-parameter, override-pin, silent-mutation, no-opt-in]
encounters: 1
lastSeen: 2026-08-27
---

# The literal path now refuses to clobber a link; the other two paths still clobber silently

`B-niagara-literal-over-linked-override-pin` is fixed: writing a bare literal onto an override pin that
already has an inbound link is refused with `MODULE_INPUT_OVERRIDE_LINKED`, and replacing it is an
explicit `breakExistingLink: true` opt-in whose response carries
`replacedOverride: {valueMode, source}` naming what was displaced.

The `{link: ...}` and `{dynamicInput: ...}` value modes of the same verb still do the old thing: they
call `ClearModuleInputOverride` unconditionally, deleting whatever was on the pin -- a dynamic-input
chain an artist built, or another agent's parameter binding -- with no opt-in and nothing in the
response reporting it.

The inconsistency is the sharp edge. A caller who has learned that this verb refuses to clobber will
reasonably assume that holds for all three value modes. It does not.

**Fix:** the same `replacedOverride` report on both paths at minimum, so the mutation is disclosed.
Whether they should *also* gate behind `breakExistingLink` is a compatibility call: the literal path
could gate because writing a literal over a link was never a working operation, whereas assigning a
new dynamic input over an old one is plausibly intended most of the time.

## History
- `#1-family-inconsistency-after-the-literal-fix` `OPEN` reporter -- Raised by the agent that fixed
  `B-niagara-literal-over-linked-override-pin`, which scoped itself to the literal branch because the
  other two sit in code other agents were editing at the time. Source-level claim against the two
  `ClearModuleInputOverride` call sites.
