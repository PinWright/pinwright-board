---
id: B-tests-addinfo-skip-not-marked
title: "128 AddInfo-then-return-true conditional skips across 43 test files, plus the 47 PINWRIGHT_SKIP_IF_*_FIXTURE* macro call sites, emit no PINWRIGHT_ASSERTIONS_SKIPPED marker — so check_suite_log classifies a run that measured nothing as COMPLETED_CLEAN"
status: OPEN
severity: High
category: bug
tags: [testing, automation, false-green, conditional-skip, skip-marker, addinfo, fixture-skip, verdict-tooling, check-suite-log, doc-correction]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# The `AddInfo` sibling of the warn-and-pass gap: same false-green, different emitter

`B-tests-warn-and-pass-without-skip-marker` (IN-REVIEW, High) converted the `AddWarning(...);
return true;` conditional skips to `PinWrightTestSkip::SkipAssertions` and shipped a CI gate,
`Content/Python/check_test_skips.py`, that flags new ones. That gate matches **`AddWarning`
only**. The `AddInfo` shape is the same defect with the same consequence and is entirely
outside it — the scanner's own docstring names it as "the largest remaining hole".

Mechanism, identical to the parent's: a test that takes a conditional-skip path returns
`true` without running its assertions. The automation framework has no skip state — the worker
derives the test's result from `GetErrorTotal()` alone — so the run reports it as a **pass**.
`PINWRIGHT_ASSERTIONS_SKIPPED:` is the plugin's own wire marker for that case
(`Source/PinWright/Private/Tests/TestSkipReporting.h`), counted by
`Content/Python/mcp_proxy.py::_count_assertion_skips` (`:816`) and used by
`check_suite_log.py` to refuse a `COMPLETED_CLEAN` verdict. None of the sites below emit it.

## Two claims from the source comment that do NOT hold — corrected here

`Content/Python/check_test_skips.py:49-53` currently reads:

> `AddInfo(...); return true;` is NOT flagged, and is the largest remaining hole: **227 sites
> in 70 files**, plus the `PINWRIGHT_SKIP_IF_*_FIXTURE*` macros in `Tests/TestUtils.h` and
> their 50 call sites. **`AddInfo` events never reach the automation log at all**, so those
> skips are worse than uncounted -- they are unreadable.

**The census is wrong.** Re-derived over the whole `Source/` tree (all eight modules;
`AddInfo` appears in zero non-test files):

- `AddInfo(...)` immediately followed by `return true;` — **128 occurrences in 43 files**, not
  227 in 70. Loosening the shape to allow one intervening statement, and restricting to
  `PinWright/Private/Tests`, reaches only 150 in 47. The stated figure does not reproduce
  under any adjacency shape tried.
- Total `AddInfo(` population for context: 423 across 114 files.
- Macro call sites: `PINWRIGHT_SKIP_IF_FIXTURE_MISSING` **43** +
  `PINWRIGHT_SKIP_IF_ALL_FIXTURES_MISSING` **4** = **47 across 18 files**. The "50" is
  accurate to within rounding.

This is the second census on this defect class to come in materially wrong —
`B-tests-warn-and-pass-without-skip-marker` `#1` filed 393 and the fixing agent had to
re-derive it. **Re-derive before acting on any number here too.**

**"`AddInfo` events never reach the automation log at all" is false.** The engine logs them.
`FAutomationControllerManager::ReportAutomationResult` walks `Results.GetEntries()` and emits
`EAutomationEventType::Info` as `UE_LOGF(LogAutomationController, Log, ...)`
(`C:\UE_5.8\Engine\Source\Developer\AutomationController\Private\AutomationControllerManager.cpp:1685-1693`).
Confirmed empirically against a retained run log —
`X:\src\unreal\unreal-fpv-new\Saved\Logs\Automation_PinWright_verify2.log` carries **36**
lines of the form:

```
LogAutomationController: FIXTURE-SKIP: /Game/Characters/Mannequins/Animations/ABP_Manny not present in this host project; test requires Lyra mannequin content.
```

So `FIXTURE-SKIP:` is **not** "a grep over a line that is never written" — the audit token
documented at `Docs/test-organization.md:260` works exactly as advertised. Correcting this
matters for the fix: the accurate framing is the one already recorded on
`TestSkipReporting.h:38-41` —

> `AddInfo` lands as `LogAutomationController: Display:`, textually identical to every other
> line of a passing run -- which is how this went unnoticed for 20 markers in `Tests/Render/`
> that no log-based checker could ever see.

i.e. the entry is *present and greppable by its own token*, but carries no severity and no
marker, so it is invisible to `check_suite_log`'s verdict and does not land in the report's
`succeededWithWarnings` bucket. That is a real defect. It is not a worse one than the parent's,
and it should not be worked as if the log evidence were missing.

## What is actually broken

1. **47 macro call sites in 18 files** skip through `PINWRIGHT_SKIP_IF_FIXTURE_MISSING` /
   `PINWRIGHT_SKIP_IF_ALL_FIXTURES_MISSING` (`Tests/TestUtils.h:462-503`). Both macro bodies
   are `AddInfo(FString::Printf(TEXT("FIXTURE-SKIP: ...")))` + `return true;` and neither emits
   the marker. **Two macro-body edits make all 47 honest at once** — this is the cheapest
   half of the fix by a wide margin and should land first.
2. **128 hand-written `AddInfo(...); return true;` sites in 43 files** do the same thing
   inline. Concentrated in `Tests/Render/` (~60), `Tests/Assets/` CRIR (~40) and
   `Tests/EditorOps/`.

Consequence: on a host missing Lyra mannequin content — which is every host except the PDS
fork — a large block of tests reports green having measured nothing, and
`check_suite_log.py` classifies the run `COMPLETED_CLEAN` because it counts only
`PINWRIGHT_ASSERTIONS_SKIPPED`. That is the exact verdict-tooling failure the marker exists to
prevent.

## Severity

**High**, equal to the parent `B-tests-warn-and-pass-without-skip-marker`, not above it.
Impact class is silent false-success on the suite verdict: the caller trusts a
`COMPLETED_CLEAN` that is a lie and ships on it. Reach bump applied — this is every suite run
on every host. Declined Critical: nothing crashes and no asset data is lost.

Rated **equal to** and not above the parent deliberately. The "worse than the AddWarning case"
argument rested on `AddInfo` being unlogged, which is refuted above; with that gone the two are
the same defect with different emitters and comparable populations (parent 393 sites / 137
files; this 128 + 47 sites / ~59 files).

**Fix:** route both macro bodies through `PinWrightTestSkip::SkipAssertions` — keeping the
`FIXTURE-SKIP:` token in the message, since `Docs/test-organization.md:260` names it as the
audit token and it demonstrably works — then sweep the 128 inline sites the way the parent
swept its 393. Widen `check_test_skips.py` to `AddInfo` **after** the sweep, not before; the
scanner's own note is right that a gate born red gets an allowlist or gets switched off. Fix
the two wrong claims in that file's docstring (`:49-53`) in the same commit — a source comment
asserting the audit token is never written will send the next reader looking for a
log-plumbing bug that does not exist.

## History
- `#1-addinfo-hole-with-corrected-census` `OPEN` reporter — Source-only census plus one archived-log measurement; **no suite was run and no plugin source was modified** (the tree is mid-verification on another wave). Counts re-derived over all of `Source/` with a multiline regex: `AddInfo\([^;]*\);\s*\n\s*return true;` → **128 in 43 files** (the filed figure of 227 in 70 does not reproduce; a loosened one-intervening-statement shape over `PinWright/Private/Tests` reaches only 150 in 47). Total `AddInfo(` population 423 across 114 files, all inside `*/Tests/` directories. Macro call sites `43 + 4 = 47` across 18 files, against the filed "50". **Refuted the load-bearing claim** that "`AddInfo` events never reach the automation log at all" (`Content/Python/check_test_skips.py:51-52`) on two independent grounds: engine code path — `FAutomationControllerManager::ReportAutomationResult` logs `EAutomationEventType::Info` entries as `UE_LOGF(LogAutomationController, Log, ...)` at `AutomationControllerManager.cpp:1685-1693` (engine source read at `C:\UE_5.8\Engine\Source`, NOT the `C:\Program Files\Epic Games\UE_5.8` path the outer repo's CLAUDE.md names — see `E-claude-md-engine-source-path-wrong`); and empirically — `Saved/Logs/Automation_PinWright_verify2.log` contains 36 `LogAutomationController: FIXTURE-SKIP: ...` lines from a real run. The accurate mechanism is the one already stated on `TestSkipReporting.h:38-41`: the entry is logged and greppable by its own token, but carries no severity, so `check_suite_log.py` (which counts only `PINWRIGHT_ASSERTIONS_SKIPPED`) cannot see it and the run classifies COMPLETED_CLEAN. Confirmed the gate does not cover this: `check_test_skips.py:99` is `_WARN_RE = re.compile(r"\bAddWarning\s*\(")` and `:291` skips any file with no `AddWarning` in it. Confirmed `SkipAssertions` is already adopted at 646 sites, so the idiom is established and this is adoption, not design. Severity High and explicitly held level with the parent `B-tests-warn-and-pass-without-skip-marker` rather than raised above it, because the "worse" argument depended on the refuted log claim.
