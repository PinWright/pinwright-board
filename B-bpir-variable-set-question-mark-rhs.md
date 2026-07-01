---
id: B-bpir-variable-set-question-mark-rhs
title: "BPIR decompiler emits ambiguous ? RHS for variable-set/default assignments"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, variables]
---

# BPIR decompiler emits ambiguous ? RHS for variable-set/default assignments

The BPIR decompiler emits magic `?` right-hand sides for some variable-set/default assignments. That makes the output ambiguous for review and unsafe for round-tripping because the reader cannot tell whether the value means null, default, unresolved, omitted, or an unsupported pin value.

**Workaround:** Inspect the live graph node and pin details for each affected assignment.

**Fix:** Replace the magic `?` with an explicit BPIR null/default/unresolved form, or omit/reset semantics with structured diagnostics when the value cannot be serialized.

## History
- `#1-fresh-app-game-dump-audit` `OPEN` reporter — Fresh 2026-05-09 App/Game dumps contain 120 literal `= ?` assignments across 56 `bpir.txt` files. Examples: `/Game/Audio/_Script/Aud_Music_Trigger` emits `set MusicManager = ?`; `/App/App/UI/LobbyAndMenu/W_TutorialsSelectAcro` emits `set DA_Hovered = ?`, `set Result = ?`, and `set SelectedButton = ?`. Existing `B-bpir-optional-pin-no-default-question-mark` is DONE and scoped to named call-site optional pins; its history explicitly left `EmitVariableSet` untouched. Need an explicit BPIR null/default RHS form or omit/reset semantics so variable-set output is reviewable and round-trippable without a magic `?`.
- `#2-typed-default-literal-helper` `IN-REVIEW` developer — Extracted FormatPinDefaultLiteral helper on FBpirDecompiler that returns nullptr/""/false/0/0.0/None/struct-literal/<unresolved> based on PinCategory. Routed three former `return TEXT("?")` sites in ResolveInputValue (terminal Case 2 + two resolver-failure paths) through the helper. EmitVariableSet fallback in BpirTextEmitter.cpp now also routes through the typed-default path. Regression test FDecompilerVariableSetEmitsTypedDefaultNotQuestionMarkTest covers object/string/bool/int/name member vars.
- `#3-verify-fix` `DONE` tester — Verified: ran `blueprint.decompile` on `/App/App/UI/LobbyAndMenu/W_TutorialsSelectAcro` (a repro asset from #1). The three previously-broken assignments now emit typed defaults: `set DA_Hovered = nullptr`, `set Result = nullptr`, `set SelectedButton = nullptr`. No `= ?` substring appears anywhere in the returned BPIR.
