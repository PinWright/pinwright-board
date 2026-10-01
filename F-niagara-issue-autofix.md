---
id: F-niagara-issue-autofix
title: "niagara: enumerate + apply engine stack-issue fixes via RPC"
status: IN-REVIEW
severity: High
category: feature
tags: [niagara, diagnostics, autofix, stack-issues, gap-analysis-2026-09-30]
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
plugin), but that facade does not exist on UE 5.3-5.7, and the plugin advertises 5.3-5.8 (verified
target: UE 5.8 on Windows).
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
- `#3-attempt-failed` `OPEN` developer — Auto-fix attempt not published: the test loop and its abandon agent both died in an API-limit blip before the suite went green; the uncommitted implementation was discarded by the next baseline sweep (by invariant). The #2 implementation notes remain a valid design reference for the retry, including the diff-review escalation: after FixDelegate.Execute(), the re-enumeration path only calls RefreshChildren() — verify the issue list is genuinely refreshed post-apply.
- `#4-design-refresh-gap-analysis` `OPEN` reporter — Re-rated Medium -> High and design refreshed from the 2026-09-30 competitor gap analysis (pinwright.com/compare row "Stack issues list and apply fix": Epic yes, PinWright/ChiR24/Monolith partial, ue-mcp and VibeUE only wrap Epic's 5.8-only toolset). Why High: no workaround reaches engine one-click fixes, and the same surface is the only route to the engine's 27 `UNiagaraValidationRule`s (`NiagaraValidationRules.h`), which feed stack issues through `UNiagaraStackViewModel::UpdateStackWithValidationResults` (`NiagaraStackViewModel.cpp:92-115`) and `UNiagaraStackEntry::AddValidationIssue` (`NiagaraStackEntry.cpp:695-717`, Fixes -> Fix style, Links -> Link style). Design corrections against the Fix section: (1) do NOT use `PinWrightNiagara::AcquireSystemViewModel` — its cache is data-only (`NiagaraSystemViewModelCache.cpp:106`, `bIsForDataProcessingOnly=true`), and a data-only view model empties module issues (`NiagaraStackModuleItem.cpp:965-970`); build one throwaway FULL view model per phase with Epic's options (`NiagaraExternalSystemEditorUtilities.cpp:558-571`: no simulate, no auto-compile, no timeline edits, MessageLogGuid=AssetGuid), `bCompileForEdit=false` behind `UE_VERSION_NEWER_THAN_OR_EQUAL(5,6,0)` (member first appears 5.6 `NiagaraSystemViewModel.h:105`), never cached — a cached one keeps stale validation issues because `RefreshChildren` keeps external issues (`NiagaraStackEntry.cpp:884`) and rules only re-run at init or on a pending-refresh Tick. (2) Mirror Epic's walker (`NiagaraExternalSystemEditorUtilities.cpp:3358-3485` walk, `:3527-3565` match, `:3568-3677` apply) instead of calling it (5.8-only header): depth-first over `GetUnfilteredChildren` from the system and each emitter root, no extra `RefreshChildren`; record location (emitter, scriptUsage, module, renderer index, input path) and a display path. (3) Ids are the engine's content hashes (`NiagaraStackEntry.cpp:73,119`); 5.8 hashes LOCTEXT keys (`MakeStableIdentifierInput` :45) but 5.3-5.7 hash localized `FText::ToString()`, so publish `idStability: stable|session`. (4) Refuse `COMPILE_IN_PROGRESS` (Epic refuses at :3496/:3582), `ISSUE_NOT_FOUND`, `ISSUE_AMBIGUOUS` (more than one entry matches), and Link-style fixes (they open UI/assets). Execute the delegate before the view model dies (fix delegates capture it; copy the description first), inside one `FScopedTransaction`. (5) Post-fix: wait for the compile the fix started, then re-list with a NEW view model and derive `issueResolved` from that list — this answers the #3 RefreshChildren concern; blocked on `B-niagara-compile-wait-does-not-wait` because the current wait does not observe the compile. Competitors, for scope: ChiR24 lists short-description strings only (no ids, fixes or apply; `McpAutomationBridge_NiagaraAuthoringHandlersStackIssues.cpp:14-111`); Monolith `get_system_diagnostics` (`MonolithNiagaraActions.cpp:7816`) never touches stack issues. Added acceptance: a Link fix is refused `FIX_IS_LINK`; a call during a compile is refused; the round trip passes on 5.8 and 5.3. Effort M. Also corrected the stale "this host is 5.7" note in the Engine-API paragraph.
- `#5-two-verbs-over-full-view-model` `IN-REVIEW` developer — Implemented per `#4`. Blocker `B-niagara-compile-wait-does-not-wait` checked first: its bounded pump (`NiagaraCompileWait.cpp` `WaitForTargets`, `AdvanceOnGameThread` + `PollForCompilationComplete`, validated `timeoutSeconds`) is in this tree with a failure-direction behavioral test (`PinWright.niagara.CompileWait.BoundedPumpCompletesTransientSystem`) and `#15` measured a real 10.1 s wait landing, so `blockedBy` was removed. New verbs, registered unconditionally: `niagara.list_stack_issues(assetPath)` (read; restores the package dirty flag; reports `idStability` stable on 5.8 / session on 5.3-5.7 and a measured `compilePending`) and `niagara.apply_issue_fix(assetPath, issueId, fixId?, timeoutSeconds)` (re-enumerates, refuses `ISSUE_NOT_FOUND` / `ISSUE_AMBIGUOUS` / `FIX_NOT_FOUND` / `FIX_AMBIGUOUS` / `FIX_IS_LINK` / `COMPILE_IN_PROGRESS` before executing anything, runs the copied fix delegate inside one `FScopedTransaction` while the view model lives, then `NiagaraEdit::RequestNiagaraCompile(force:false)` + `WaitForSystemCompile`, then re-lists through a NEW full view model and derives `issueResolved` / `resolvedIssueIds` / `introducedIssueIds` from it; `issueResolved:null` when the wait expires; never saves, reports measured `packageDirty`). One FULL throwaway view model per phase with Epic's 5.8 options, `bCompileForEdit=false` behind `UE_VERSION_NEWER_THAN_OR_EQUAL(5,6,0)` (row added to `docs/engine-version-support.md`); walker mirrors Epic's DFS over `GetUnfilteredChildren` with emitter / scriptUsage / module / stackPath location. Systems only: a standalone emitter asset is refused `ASSET_WRONG_TYPE` (its issues are listed through an owning system). Files: `Handlers/Niagara/NiagaraStackIssueHandler.cpp` (both verbs), `Handlers/Niagara/NiagaraStackIssues.h` (`SelectStackIssueFix`), `Handlers/ErrorCodes.h` (6 codes), `Tests/Niagara/TestNiagaraStackIssues.cpp`, `docs/wiki-src/niagara.md`, `docs/engine-version-support.md`, `CHANGELOG.md`. Tests: `PinWright.niagara.stack_issues.FixSelection` (FIX_IS_LINK / FIX_AMBIGUOUS / FIX_NOT_FOUND on synthetic issues), `.ApplyFixResolvesIssue` (disable the stock fixture's `SolveForcesAndVelocity` -> new dependency issue with the engine "Enable module" fix -> apply by id -> provider re-enabled read from the graph, `issueResolved:true`, fresh list equals the pre-disable baseline), `.UnknownIdsAreRefused` (FIX_NOT_FOUND / ISSUE_NOT_FOUND, provider still disabled, issue still listed), `.RefusedWhileCompiling` (both verbs refuse COMPILE_IN_PROGRESS). Compile-checked with `-SingleFile` on 5.8 Linux; not built, not run, no live editor; 5.3-5.7 not compiled on this host (5.8 only).
