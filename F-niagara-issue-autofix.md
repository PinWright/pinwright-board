---
id: F-niagara-issue-autofix
title: "niagara: apply-fix RPC for validate/stack issues"
status: OPEN
severity: Medium
category: feature
tags: [niagara, diagnostics, autofix, parity-ue58]
---

# niagara: apply-fix RPC for validate/stack issues

`niagara.validate` returns structured issues and `niagara.compile` compiles, but there is no way to APPLY the fixes the Niagara stack itself proposes (grep `apply_issue_fix|autofix|auto_fix` in `Handlers\Niagara\` = zero). Many stack issues carry engine-provided one-click fixes (missing required module, deprecated usage, reset to sane value); agents must currently hand-author each repair.

UE 5.8 parity evidence: NiagaraToolsets diagnostics loop `GetStackIssues` -> `ApplyStackIssueFix` (+ async `GetSystemCompileState` polling), `C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\NiagaraToolsets\` (grew 49 -> 61 tools in 5.8.0), over `UNiagaraExternalEditUtilities`. This validate-fix-recompile loop is the best agent ergonomic in Epic's whole toolset family.

Related documentation-side tickets (do not duplicate): `E-niagara-validate-compile-state-uninitialized-undocumented` (OPEN), `E-niagara-validate-strict-empty-system-undocumented` (IN-REVIEW).

Proposed scope:
- Extend `niagara.validate` output with stable issue ids + available fix descriptors per issue.
- `niagara.apply_issue_fix(asset, issueId, fixIndex)` applying the engine-registered fix, returning the re-validated issue list.

Acceptance: on a system with a known fixable stack issue, validate -> apply fix -> re-validate shows the issue gone and compile passes.

## History
- `#1-no-fix-application` `OPEN` reporter — validate reports issues but engine-proposed fixes cannot be applied via RPC (grep verified). Epic 5.8 GetStackIssues/ApplyStackIssueFix loop is the parity target; add issue ids + apply_issue_fix.
