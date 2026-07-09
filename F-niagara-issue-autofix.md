---
id: F-niagara-issue-autofix
title: "niagara: enumerate + apply engine stack-issue fixes via RPC"
status: IN-REVIEW
severity: Medium
category: feature
tags: [niagara, diagnostics, autofix, stack-issues]
---

# niagara: enumerate + apply engine stack-issue fixes via RPC

The Niagara editor stack attaches engine-provided one-click fixes to its entries (missing
required module, deprecated usage, reset-to-sane-value, etc.), but there was no RPC to enumerate
or apply them (grep `apply_issue_fix|autofix|auto_fix|GetStackIssues` over `Source/` = zero).
Agents had to hand-author each repair with a specific edit verb.

**These are the engine stack issues, NOT `niagara.validate`'s issues.** `niagara.validate`'s
`issues` are PinWright-synthesized: they come from the compile log
(`NiagaraInspectHandler.cpp` `AddCompileIssues`, wired from `NiagaraDumpBuilder::BuildCompileJson`)
plus a hand-authored disabled-emitter scan (`AddDisabledEmitterIssues`). None of those carry an
engine fix delegate. The applicable one-click fixes live on a different surface — the stack view
model's entries: `UNiagaraStackEntry::GetIssues()` -> `FStackIssue` -> `FStackIssueFix`
(`GetUniqueIdentifier`/`GetDescription`/`GetFixDelegate`/`GetStyle`), reached by walking the stack
view models the `FNiagaraSystemViewModel` owns (`GetSystemStackViewModel()` +
`GetEmitterHandleViewModels()[i]->GetEmitterStackViewModel()`). So this is a new stack-issue
enumeration/apply path, not an extension of `niagara.validate`.

Engine-API note: UE 5.8 wraps the same mechanism behind
`UNiagaraExternalEditUtilities::GetStackIssues`/`ApplyStackIssueFix` (the `NiagaraToolsets`
plugin), but that facade does not exist on UE 5.3-5.7 (the supported range; this host is 5.7).
The low-level `UNiagaraStackEntry` stack-issue API it wraps has been present and
`NIAGARAEDITOR_API`-exported since ~5.0, so drive that directly and register unconditionally (no
`SendUnsupportedEngineVersion` needed — the surface builds on every supported engine).

Related documentation-side tickets (do not duplicate):
`E-niagara-validate-compile-state-uninitialized-undocumented` (OPEN),
`E-niagara-validate-strict-empty-system-undocumented` (IN-REVIEW).

Fix:
- Add `niagara.list_stack_issues(assetPath|systemPath)` (read): acquire the system view model via
  `PinWrightNiagara::AcquireSystemViewModel`, walk the system + enabled-emitter stack view models
  (`RefreshChildren` then recurse `GetUnfilteredChildren`, collecting `GetIssues()`), and return
  each issue with a stable `issueId`, severity, descriptions, and a `fixes` array
  (`fixId`/`description`/`style`/`applicable`; Link-style fixes are non-programmatic).
- Add `niagara.apply_issue_fix(assetPath|systemPath, issueId, fixId?)` (mutate): re-enumerate,
  match `issueId` (and `fixId`, or the issue's single applicable fix), execute the fix delegate
  inside an `FScopedTransaction`, then re-enumerate and report `issueResolved` +
  `remainingIssueCount`. Read and mutate are two separate verbs (not one verb toggled by a flag).

Acceptance: on a Niagara System with a fixable stack issue, list_stack_issues -> apply_issue_fix
(by issueId) -> list_stack_issues shows that issue gone.

## History
- `#1-no-fix-application` `OPEN` reporter — validate reports issues but engine-proposed fixes cannot be applied via RPC (grep verified). Epic 5.8 GetStackIssues/ApplyStackIssueFix loop is the parity target; add issue ids + apply_issue_fix.
- `#2-reword-and-implement` `IN-REVIEW` developer — REWORDED then implemented. Two load-bearing mis-statements corrected against source: (1) the cited `UNiagaraExternalEditUtilities::ApplyStackIssueFix` is UE 5.8-only (absent from C:\UE_5.7 NiagaraEditor; `NiagaraToolsets` plugin not present on 5.7) — used the 5.7-available low-level `UNiagaraStackEntry::GetIssues()`/`FStackIssue::GetFixes()` API instead (all `NIAGARAEDITOR_API`-exported); (2) the "extend niagara.validate output" scope was wrong — validate's issues are PinWright-synthesized compile-log/disabled-emitter JSON (`NiagaraInspectHandler.cpp` AddCompileIssues/AddDisabledEmitterIssues:130-196), not engine stack issues, so this is a new SVM stack-issue path. Implemented as two verbs sharing an enumerator: `niagara.list_stack_issues` (read) + `niagara.apply_issue_fix` (mutate), registered unconditionally. Files: Handlers/Niagara/NiagaraApplyIssueFixHandler.cpp (both verbs), Handlers/Niagara/NiagaraStackIssueHelpers.h (shared stack walk + JSON), Tests/Niagara/TestNiagaraApplyIssueFix.cpp (adopted red registration test PinWright.niagara.apply_issue_fix.Registration + behavioral list/apply tests over a SimpleExplosion-derived fixture).
