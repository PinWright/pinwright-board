---
id: B-error-code-registry-blind-to-variable-codes
title: "core.error_codes.AllEmittedCodesAreRegistered cannot see a code sent through a variable, so unregistered codes ship green"
status: IN-REVIEW
severity: Medium
category: bug
tags: [test-gap, error-codes, registry, AllEmittedCodesAreRegistered, guard-inefficacy]
encounters: 1
lastSeen: 2026-08-27
---

# The error-code registry guard has a blind spot, and several codes are already through it

`PinWright.core.error_codes.AllEmittedCodesAreRegistered` matches emission patterns anchored on a
string literal at the `SendError(` call site, or on a `...Code = TEXT(...)` assignment. A code
forwarded through a **variable** matches none of them and is invisible to the test.

The Niagara family does exactly that: `FNiagaraEditError::Make(TEXT("..."))` builds the error and
`SendNiagaraEditError` forwards `*Err.Code` to `Ctx.SendError`. Codes confirmed emitted and **not**
present in `Handlers/ErrorCodes.h`: `UNSUPPORTED_NODE_CLASS`, `INVALID_OP`, `PARAMETER_TYPE_MISMATCH`,
`DYNAMIC_INPUT_SET_FAILED`, and now `MODULE_INPUT_OVERRIDE_LINKED`.

The suite is green on all of them. That is the defect: the guard exists, is consulted, and does not
see this shape — so "the registry test passes" is not evidence that a verb's codes are registered.

Note this interacts with `RegistryAdoptingFilesUseConstantsOnly`, which grandfathers a file that
hand-spells raw literals and flips it to "adopting" on the first `ErrorCodes::ERR_` reference. Simply
registering these codes is therefore not a one-line change for the files that emit them; the two rules
have to be resolved together.

**Fix:** widen the emission scan to follow a code through a local, or (more robustly) key the check off
the wire rather than the source — collect every distinct code string the suite actually emits and
assert each is registered.

## History
- `#1-two-agents-found-it-independently` `OPEN` reporter — Found independently by the agent fixing
  `B-niagara-create-node-unfinalized-graph-node-creator-fatal` and the agent fixing
  `B-niagara-literal-over-linked-override-pin`, both of which checked whether their new code needed
  registering and discovered the existing ones were not. Source-level claim against
  `Tests/.../TestErrorCodeRegistry.cpp`'s three patterns and `Handlers/ErrorCodes.h`.
- `#2-widen-the-source-scan` `IN-REVIEW` developer — "Added a fourth emission pattern
  (an Err/Fail-named callee taking the code literal as its first argument) to
  `TestErrorCodeRegistry.cpp` so a code handed to an
  error-VALUE factory and forwarded to `SendError` through a variable is collected, kept it disjoint
  from the SendError patterns so the per-pattern liveness guard still means something, neutralized
  `ScanEmittedCodes`'s input so a commented-out call is no longer scored as an emission, and
  registered the 34 codes the widened scan exposed in `Handlers/ErrorCodes.h` — 32 Niagara plus
  ANALYSIS_FAILED / NO_TIMING_DATA from `Debug/TraceExportCore.cpp`; not 5 as the report estimated.
  Chose the source scan over the wire-side alternative: a runtime recorder only ever sees codes the
  suite happens to exercise, so its coverage would be a function of test coverage and an empty
  recording would look like a pass. New test
  `PinWright.core.error_codes.EmissionScanFollowsForwardedCodes`."
