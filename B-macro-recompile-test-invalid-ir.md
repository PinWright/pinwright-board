---
id: B-macro-recompile-test-invalid-ir
title: "MacroRecompilePreservesCallerInstances fails on UE 5.7+: its failing-recompile IR calls Conv_IntToFloat, which no longer exists, so the compile fails for the wrong reason"
status: IN-REVIEW
severity: Medium
category: bug
tags: [bpir, compiler, tests, engine-version, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T09:25:51Z
---

# MacroRecompilePreservesCallerInstances asserts a caller-pin guard its IR never reaches

`PinWright.bpir.compiler.MacroRecompilePreservesCallerInstances`
(`Source/PinWright/Private/Tests/Bpir/TestCompilerMacroRecompilePreservesCallers.cpp`) recompiles
`TestMacro` a third time with `Value` retyped `float` -> `int` and asserts the compile fails, so
the caller-pin guard (`ReconstructSameBlueprintMacroCallers` ->
`ValidateMacroCallerSnapshotPreserved`, `BpirCompiler.cpp` ~1404: "changed type on existing caller
pin") refuses to mutate the existing caller, then checks the caller is restored intact.

That guard runs only after body emission succeeds (`BpirCompiler.cpp` ~3590-3630: an emission error
takes the `AccumulatedErrors` rollback first). The body called `Conv_IntToFloat`, which UE 5.7 and
5.8 removed (5.3-5.6 have both `Conv_IntToFloat` and `Conv_IntToDouble`; 5.7/5.8 only
`Conv_IntToDouble`, `KismetMathLibrary.h:3591`). On 5.7+ the compile therefore failed at emission:

```
LogBpirCompiler: Warning: EmitInstruction failed for opcode 1 at line 2
LogBpirValueResolver: Error: Value reference '%converted' instruction has no primary output pin
```

`TestFalse(bSuccess)` passed for the wrong reason, the caller-guard path was never exercised on
5.7+, and the resolver's `Error` line fails the test once it is attributed to it (host log
`Saved/Logs/pw_gapwave_groups3.log`, line 7113-7117).

The three `LogUObjectGlobals: Warning: Failed to find object 'Class Object'` lines at test start
come from the same IR: each of the three compiles declares `object<Object> Target`, and
`ResolveUClass("Object")` step 2 (`Utils/ClassUtils.cpp`) runs `LoadObject<UClass>` on the bare
short name, which can never load and warns every time. Any `object<ShortName>` spelling pays that
warning; that resolver noise is not fixed here.

**Fix:** body uses `Conv_IntToDouble` (present on 5.3-5.8); the three signatures use bare `object`,
which `ConvertTypeSpecToPinType` maps to the same `UObject` pin type without a class lookup; and the
failing recompile now also asserts that an error names "changed type on existing caller pin
'Value'", so an emission failure can no longer satisfy the test.

## History
- `#1-wrong-failure-reason` `OPEN` reporter — Surfaced by the gap-wave verification run (`pw_gapwave_groups3.log`): the failing-recompile step fails in emission on `Conv_IntToFloat` (absent on 5.7/5.8) instead of at the caller pin-type guard; `object<Object>` accounts for the three `Class Object` warnings.
- `#2-valid-ir-and-reason-assert` `IN-REVIEW` developer — Changed `TestCompilerMacroRecompilePreservesCallers.cpp`: `Conv_IntToFloat` -> `Conv_IntToDouble` (verified present in `C:\UE_5.3`..`C:\UE_5.8` `KismetMathLibrary.h`), `object<Object> Target` -> `object Target` in all three IRs, and added `TestTrue` that `Result.Errors` contains "changed type on existing caller pin 'Value'" (errors logged via `AddInfo` when it does not). No assertion removed. Needs a build and a run of this test on 5.8 and one pre-5.7 engine.
- `#3-guard-reached-default-lost` `IN-REVIEW` developer — Offscreen full suite (`Saved/Logs/pw_gapwave_full_offscreen2.log` line 20989) confirms the fixed IR now reaches the caller pin-type guard (the new "changed type on existing caller pin 'Value'" assertion passed). The next assertion fails: "Value default survives failed signature recompile" expected "5.0", got "". That is a real compiler defect the old emission failure hid, filed as `B-macro-refused-recompile-drops-defaults` and fixed there; this ticket's test change stays as is.
