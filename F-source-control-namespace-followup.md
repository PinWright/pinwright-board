---
id: F-source-control-namespace-followup
title: "source_control.connect — no in-MCP way to bring up / attach a provider"
status: IN-REVIEW
severity: Medium
category: feature
tags: [source-control, perforce, git, connect, revert]
---

# `source_control.connect` — bring up a source-control provider from the MCP

This is the sibling follow-up ticket that `F-source-control-namespace` (`DONE`)
explicitly promised but never got filed. That ticket shipped a narrowed 5-RPC
subset (`get_provider`, `status`, `log`, `revert`, `mark_for_add`) and
**deferred** the originally-proposed `connect`, `diff`, and `get_changelist`
verbs "to a sibling ticket TODO F-source-control-namespace-followup" (see
`F-source-control-namespace` history `#2-implemented-narrowed-namespace`). No
file by that name exists on the board, so the deferral was never tracked.

## Scope (narrowed)

Of the three deferred verbs, only **`connect`** addresses a real, un-worked-around
capability gap; it is the prerequisite that makes the entire `source_control.*`
namespace reachable from a remote/headless/CI caller. The other two are dropped
from this ticket:

- **`diff`** — largely overlaps the already-shipped `source_control.log`, which
  fetches full per-revision history via `FUpdateStatus::SetUpdateHistory(true)`.
  Marginal incremental value; not pursued here.
- **`get_changelist`** — the Perforce-centric changelist concept the parent
  ticket itself flagged as messy and deliberately deferred. Speculative until a
  concrete caller needs it; file a fresh ticket then.

This ticket therefore ships exactly one verb: `source_control.connect`.

## What's wrong / why it matters

Without `source_control.connect`, there is **no in-MCP way to enable or attach
a source-control provider**. In any editor session where SC isn't already
connected — e.g. a freshly-launched project, CI, or a headless host —
`get_provider` returns `{providerName:"None", isEnabled:false,
isAvailable:false}` and every mutating/query verb (`status`, `log`, `revert`)
fails cleanly with `[SOURCE_CONTROL_DISABLED]`. The namespace can only operate
when a human has already brought up the provider in the editor UI, which a
remote/automation caller cannot do. The shipped verbs return correct errors —
this is a capability gap, not a bug in those verbs.

Concretely, a reasonable "discard my local edit and restore this asset to its
committed baseline" task cannot be completed end-to-end through the MCP: the
`source_control.revert` round-trip is unreachable because nothing can first
turn SC on.

## What it should do

- `source_control.connect(providerName?, settings?)` — switch the active
  provider to `providerName` (one of the registered provider names) and force a
  connection/login attempt, so a caller can bring SC up. When `providerName` is
  omitted, re-attempt connection/login on the **already-configured** provider
  (useful when SC is set but `isAvailable` is false). On an **unknown**
  provider name, return a clear, actionable `INVALID_PROVIDER` error that lists
  the available provider names — and **never** call `ISourceControlModule::SetProvider`
  with an unregistered name (that engine API asserts → crashes the editor).
  Response surfaces `{providerName, isEnabled, isAvailable, switched, availableProviders}`.

**Fix:** New handler `source_control.connect` in `SourceControlHandler.cpp`,
routing through `ISourceControlModule::Get()`: gate `SetProvider(FName)` behind
a `GetProviderNames()` membership check (SetProvider asserts on an unknown name),
then `GetProvider().Init(/*bForceConnection=*/true)` + `Login()`, and report the
resulting enabled/available flags. No `RequireEnabledProvider` gate — this verb
is the bring-up affordance and must run in a disabled session.

## Verbatim repro (current shipped behavior)

- `source_control.get_provider` `{}` →
  `{"providerName":"None","isEnabled":false,"isAvailable":false}`
- `source_control.status` `{"assetPaths":["/Game/Global/Materials/M_Button_Emissive"]}` →
  error `[SOURCE_CONTROL_DISABLED] Source control is not enabled`
- `source_control.revert` `{"assetPaths":["/Game/Global/Materials/M_Button_Emissive"]}` →
  error `[SOURCE_CONTROL_DISABLED] Source control is not enabled`

The `source_control` namespace page lists only `get_provider`, `log`,
`mark_for_add`, `revert`, `status` — there is no `connect`/`enable`/
`set_provider` verb, and no sibling RPC in any other namespace connects a
provider.

## History
- `#1-initial-repro` `OPEN` reporter — Filing the never-created follow-up that `F-source-control-namespace` `#2` promised. Real task ("revert `/Game/Global/Materials/M_Button_Emissive` to its committed baseline") is blocked: provider is `None`/disabled and there is no in-MCP `connect` to bring SC up, so the `source_control.revert` round-trip is unreachable. Replay-confirmed get_provider→None/disabled and status/revert→`[SOURCE_CONTROL_DISABLED]`. The shipped verbs error correctly; this is the deferred-capability gap, not a bug in them.
- `#2-narrowed-to-connect` `OPEN` developer — Reworded down to the one high-value verb. The deferred bundle was over-scoped: `diff` overlaps the shipped `source_control.log` (both walk per-revision history via `FUpdateStatus::SetUpdateHistory(true)`), and `get_changelist` is the Perforce-centric concept the parent deliberately deferred — both dropped (file fresh tickets if a concrete caller appears). Only `source_control.connect` fixes the real, un-worked-around blocker (no in-MCP way to bring up a provider for the `revert` round-trip). Title/scope/Fix updated; severity Medium and category feature unchanged.
- `#3-implemented-connect` `IN-REVIEW` developer — Added `source_control.connect` to `Source/EditorAutomationRpcGateway/Private/Handlers/SourceControl/SourceControlHandler.cpp`. Optional `providerName` (omitted = re-attempt the current provider) + optional `settings` object. Validates `providerName` against `ISourceControlModule::Get().GetProviderNames()` and returns `INVALID_PROVIDER` (listing `availableProviders`) on miss — guarding against `SetProvider(FName)` which asserts/crashes on an unregistered name. On a valid name calls `SetProvider`, then `GetProvider().Init(true)` + `Login()`, and reports `{providerName, isEnabled, isAvailable, switched, availableProviders}`. Not gated by `RequireEnabledProvider` (this is the bring-up affordance, must run in a disabled session). Settings keys are forwarded into `FSourceControlInitSettings` (currently informational/echoed; not yet routed through a per-provider CreateProvider path). Regression test `FSourceControlConnectInvalidProviderTest` in `Source/EditorAutomationRpcGateway/Private/Tests/SourceControl/TestSourceControlHandlers.cpp` invokes the production handler with `providerName:"NoSuchProvider"` and asserts the `INVALID_PROVIDER` error + an `availableProviders` array — it would fail (or crash) if the membership guard before `SetProvider` were removed. Did not compile/run (later phase).
