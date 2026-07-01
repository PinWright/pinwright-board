---
id: E-bpir-param-target-collision
title: "BPIR's `Target:` caller-object convention collides with UFUNCTION parameters literally named `Target`"
status: DONE
severity: Low
category: ergonomic
tags: [bpir, call-syntax, pin-resolution, docs]
---

# BPIR's `Target:` caller-object convention collides with UFUNCTION parameters literally named `Target`

BPIR uses `Target:` as the conventional argument for the caller-object pin on member-function calls (`call Func(Target: %obj, Param: value)`). When a UFUNCTION declares a parameter literally named `Target` — common in this codebase for enum-dispatching APIs — `Target:` resolves to the caller-object pin, not the parameter. The error at least surfaces the right pin list:

```
Available pins: self, Target, InitialTransform
```

...but the correct syntax (`self: %obj` for the caller + `Target: $value` for the parameter) is not documented in `bpir-language-reference.md`.

**Repro (observed this session, `/App/App/UI/ReplayEditor/W_AppReplayEditor`):**

`UReplayEditorManualPlacementWidget::Begin(EReplayManualPlacementTarget Target, const FTransform& InitialTransform)` — the first param is literally `Target`.

First attempt (collides):
```
call Begin(Target: %typed, Target_1: $EnumTarget, InitialTransform: $T)
```
→ `Could not find target pin 'Target_1' on node 'Begin'. Did you mean 'Target'? Available pins: self, Target, InitialTransform`

Fix (undocumented, discovered by reading the pin-list hint):
```
call Begin(self: %typed, Target: $EnumTarget, InitialTransform: $T)
```

**Impact:** Low — the `Available pins:` hint (see `E-bpir-createwidget-pin-hint`) already reveals the right names. But without docs, authors still guess `Target_1`, `Target2`, or similar invalid disambiguators first. The `self:` alias isn't surfaced anywhere in the language reference; without that keyword the collision is otherwise unresolvable short of renaming the parameter in C++.

**Workaround:** Use `self:` for the caller-object when a UFUNCTION has a parameter named `Target`.

**Proposal:**
1. Add a short section to `bpir-language-reference.md` (next to the `call` syntax) describing the `self:` alias for the caller-object pin and documenting the collision rule: "when a function's parameter shares a name with a BPIR convention (`Target`, `Self`, etc.), the parameter takes precedence and the caller-object must be passed via `self:`".
2. Consider aliasing the caller-object pin as `Self:` too for consistency with BPIR's `self` literal; would make `call Func(Self: %obj, Target: $param)` read more naturally than `self:` (lowercase looks like a keyword, not an argument).

## History
- `#1-target-convention-collision` `OPEN` reporter — Hit during replay-editor root wiring when composing the `ShowManualPlacement` custom event body. The enum param `EReplayManualPlacementTarget Target` on `UReplayEditorManualPlacementWidget::Begin` collided with `Target:` convention; first attempt used `Target_1:` as a guess at disambiguation. The `Available pins: self, Target, InitialTransform` hint surfaced the right names; `self:` worked but isn't documented. Lost ~1 minute iterating. Low severity — the hint keeps it from being a dead-end.
- `#2-docs-section-added` `IN-REVIEW` developer — Added new subsection "Caller-Object Pin and the `Target:` Convention Collision" to `docs/bpir-language-reference.md` Section 2.1. Documents the `Target:` alias, the collision rule when UFUNCTIONs have a parameter literally named `Target`, and the `self:` (lowercase) escape hatch with a concrete example. Docs-only; per mcp-sprint rules for ergonomic-with-no-observable-behavior tasks, no regression test is required.
- `#3-verified-docs-section-present` `DONE` tester — Verified: grepped `docs/bpir-language-reference.md` for "Caller-Object Pin"/"Target.*Collision"/"`self:`". Section present at line 208 titled "Caller-Object Pin and the `Target:` Convention Collision"; content documents the `self:` lowercase escape hatch (line 214), a right/wrong example pair (line 223), and a standardization note on lowercase (line 228). Docs-only fix is in place.
