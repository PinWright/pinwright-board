---
id: F-source-control-namespace
title: "First-class `source_control.*` namespace (provider status, log, diff, mark-for-add, revert)"
status: DONE
severity: Low
category: feature
tags: [source-control, perforce, git, subversion]
---

# First-class `source_control.*` namespace (provider status, log, diff, mark-for-add, revert)

Current SCM surface is three handlers under `asset.*`:
`asset.get_source_control_state`, `asset.source_control_checkout`,
`asset.source_control_submit`. They cover only checkout/submit/state on a
single asset and have no concept of a provider, history, or diff. There is no
way for a remote caller to ask "which SCM is active?", "is it connected?",
"what changed in this file three revisions ago?", or "discard my local
changes".

**Proposal:** Add a dedicated `source_control.*` namespace wired to
`ISourceControlModule::Get().GetProvider()` so it works uniformly across the
shipped providers (Perforce, Git, Subversion, PlasticSCM via third-party
plugins). Suggested methods:

- `source_control.get_provider()` — returns active provider name, connected flag, workspace/root info.
- `source_control.connect(providerName, settings)` — switches/initialises a provider (`FSourceControlInitSettings`).
- `source_control.status(paths)` — batched `FUpdateStatus` over a list of asset/file paths.
- `source_control.log(asset, maxRevisions)` — `FGetFileHistory` results: revision, date, user, description, action.
- `source_control.diff(asset, revision?)` — fetch a previous revision's file via `ISourceControlRevision::Get()`; return path on disk (or contents for text files).
- `source_control.mark_for_add(asset)` — `FMarkForAdd`.
- `source_control.revert(asset)` — `FRevert`.
- `source_control.get_changelist(name?)` — describe pending changelist contents (Perforce) / staged set (Git).

The existing three `asset.source_control_*` handlers stay as thin aliases so
no caller breaks. Discovery and the wiki overlay surface the new namespace as
the recommended entry point.

**Fix:** Implement as a new `Handlers/SourceControl/` subdirectory, one
handler per verb following the standard `REGISTER_RPC_HANDLER` pattern.
Operations route through `ISourceControlModule` so behaviour is consistent
across providers without per-provider branching in our code.

## History
- `#1-proposed-namespace` `OPEN` reporter — Audit found SCM coverage is asset-scoped checkout/submit/state only; no provider introspection, history, diff, or revert. Proposed full `source_control.*` namespace backed by `ISourceControlModule`.
- `#2-implemented-narrowed-namespace` `IN-REVIEW` developer — Implemented narrowed 5-RPC subset of source_control.* in new `Handlers/SourceControl/SourceControlHandler.cpp`: get_provider, status, log, revert, mark_for_add. All route through ISourceControlModule::Get().GetProvider() with Execute() over FUpdateStatus/FRevert/FMarkForAdd operations (history fetched via FUpdateStatus::SetUpdateHistory(true); there is no separate FGetFileHistory in UE 5.x). Asset paths converted to filenames via FPackageName::TryConvertLongPackageNameToFilename + GetAssetPackageExtension. Existing asset.source_control_* handlers kept as-is for back-compat. The deferred verbs (connect, diff, get_changelist) have provider-specific quirks (init settings JSON plumbing, file-fetch + path semantics for binary uassets, Perforce-centric changelist concept) and are deferred to a sibling ticket TODO F-source-control-namespace-followup. Tests in `Tests/SourceControl/TestSourceControlHandlers.cpp` cover GetProvider shape + StatusDisabled counterfactual. No Build.cs change — SourceControl already on PrivateDependencyModuleNames.
- `#3-verify-fix` `DONE` tester — Verified: `source_control?` discovery returns all 5 promised methods (get_provider, status, log, revert, mark_for_add); `source_control.get_provider` executes cleanly and returns `{providerName:"None", isEnabled:false, isAvailable:false}` in this disabled-SCM editor session, matching the implementation contract.
