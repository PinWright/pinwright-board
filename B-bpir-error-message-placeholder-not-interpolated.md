---
id: B-bpir-error-message-placeholder-not-interpolated
title: "`compile_bpir` private-variable error message contains literal `{VariableName}` placeholder"
status: DONE
severity: Low
category: bug
tags: [bpir, error-messages, formatting, placeholder]
---

# `compile_bpir` private-variable error message contains literal `{VariableName}` placeholder

When BPIR `set %target.SomeVar = value` references a `BlueprintPrivate` variable on an external object, the compile error message contains the literal text `{VariableName}` instead of the actual offending variable name. The trailing `Set <VarName>` half does carry the name correctly, so the formatter is only half-broken.

## Repro (observed this session)

After adding `Replay` and `ParentScreen` vars to `W_RenameReplay` via `blueprint_add_variable` (which defaults to `BlueprintPrivate=true` — see `E-add-variable-type-format` #4), then attempting BPIR on `W_MyReplayListItem` to set them on a freshly-spawned popup:

```
set %n3.AsWRenameReplay.Replay = $Replay
set %n3.AsWRenameReplay.ParentScreen = $ParentScreen
```

Errors returned (verbatim):

```
{VariableName} is private and not accessible in this context.  Set ParentScreen
{VariableName} is private and not accessible in this context.  Set Replay
{VariableName} is private and not accessible in this context.  Set ParentScreen
{VariableName} is private and not accessible in this context.  Set Replay
```

Note: the trailing `Set <Name>` part *does* substitute the variable name correctly. Only the leading sentence's `{VariableName}` fails to interpolate. Looks like a printf-style format string was passed where a brace-placeholder format string was expected, or vice versa — the `{VariableName}` token was never substituted.

## Impact

Low — the error is still actionable because the trailing half includes the var name. But the literal `{VariableName}` makes the message look like a half-shipped feature, and would mislead anyone grepping logs for the actual var name in the error sentence's expected position.

**Workaround:** read the trailing `Set X` half of each error line.

**Proposal:** find the source format string (likely in a UE `LOCTEXT`/`FText::Format` or `FString::Printf` call in the BPIR compiler's variable-resolution path), confirm the substitution is wired and the placeholder argument is being supplied. If the format string uses `{VariableName}` brace style, ensure the `FFormatNamedArguments` map has a `"VariableName"` entry; if printf-style, the format specifier and arg should match.

## History
- `#1-initial-repro` `OPEN` reporter — Hit during `W_MyReplayListItem` rename-button wiring. Four identical errors with literal `{VariableName}` in the leading sentence; trailing `Set Replay` / `Set ParentScreen` correctly named. Resolved the underlying private-access problem via `blueprint_set_variable_settings` + repo file edit, but the formatting bug remains.
- `#2-repair-engine-broken-placeholder` `IN-REVIEW` developer — Root cause is in UE engine source (`K2Node_VariableSet.cpp:450` calls `LOCTEXT(...).ToString()` directly without `FText::Format(..., Args)`, so `{VariableName}` never substitutes — sibling line 446 for `NotWritable` does it correctly). Since the plugin can't patch engine source, added a narrow repair pass in `BlueprintHandlerUtils.cpp::CompileBlueprintWithDiagnostics` that detects the exact broken pattern (`{VariableName} is private and not accessible in this context.  Set <id>`) and splices the identifier from the trailing `Set <id>` suffix into the leading sentence. Regression test `FBpirPrivateVariableErrorMessageInterpolationTest` directly exercises the repair helper with the known engine-produced string.
- `#3-verified-interpolation` `DONE` tester — Added a private (BlueprintPrivate=true) variable `MyPrivate` to `/Game/App/UI/Test/W_McpVerifyTemp`, then on `/Game/App/UI/Test/W_McpVerifyOverride` ran `compile_bpir` body `%p = call GetOwningPlayer(Target: self); %item = call K2Node_CreateWidget(Class: /Game/App/UI/Test/W_McpVerifyTemp.W_McpVerifyTemp_C, OwningPlayer: %p); set %item.MyPrivate = 7`. Returned `compiled: false, status: "Error"` with `errors: [{message: "MyPrivate is private and not accessible in this context.  Set MyPrivate"}]`. Variable name correctly interpolated in the leading sentence — pre-fix would have been `"{VariableName} is private..."` literal. Repair pass active.
