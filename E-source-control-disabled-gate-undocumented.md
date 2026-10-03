---
id: E-source-control-disabled-gate-undocumented
title: "source_control.md never says that a disabled provider gates status/log/revert/mark_for_add or that source_control.connect brings one up"
status: OPEN
severity: Low
category: ergonomic
tags: [source-control, docs]
encounters: 2
lastSeen: 2026-06-23T10:08:23Z
rice: [1, 1, 1, 1]
priority: 8
---

# `source_control` wiki overlay doesn't document the provider precondition gate

When no provider is enabled, `source_control.get_provider` returns `{providerName:"None",
isEnabled:false, isAvailable:false}` and every other read/write verb (`status`, `log`, `revert`,
`mark_for_add`) fails with `[SOURCE_CONTROL_DISABLED] Source control is not enabled`
(`RequireEnabledProvider`, `Source/PinWright/Private/Handlers/SourceControl/SourceControlHandler.cpp:81-91`).
`source_control.connect` (`:132`) is the verb that brings a provider up, and it deliberately skips
that gate (`:126-131`).

`docs/wiki-src/source_control.md` mentions neither the gate nor `connect`. The overlay opens with a
flat list of usable verbs, so a caller learns the gate by issuing the doomed calls. Two recorded
tasks spent 3 and 8 redundant SC calls this way after `get_provider` had already returned
`isEnabled:false`.

The `asset.*` SC verbs share the gate: `asset.source_control_checkout` and
`asset.source_control_submit` fail every row with `SOURCE_CONTROL_DISABLED` at preflight
(`Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp:358-369`, `:491-501`). `docs/wiki-src/asset.md:103`
documents this for checkout; the submit section (`:105-107`) does not.

**Fix:** add a short precondition note near the top of `docs/wiki-src/source_control.md`: call
`get_provider` first; if `isEnabled` is false, call `source_control.connect` (optionally with
`providerName`) before any other verb in the namespace, or stop. Add the same disabled-SC preflight
sentence to the `asset.source_control_submit` section of `asset.md`.

**Acceptance:** the generated `source_control` page states the `isEnabled` gate, lists the verbs it
blocks, and names `source_control.connect` as the way out. The `asset.source_control_submit` section
says that disabled source control is a preflight failure.

Related: `F-source-control-namespace-followup` (added `connect`), `E-error-code-vocabulary-registry`.

## History
- `#1-initial-audit` `OPEN` reporter — Process/docs friction distinct from the capability gap in `F-source-control-namespace-followup` (which proposes adding `connect`). Here: the `docs/wiki-src/source_control.md` overlay never states that `get_provider.isEnabled=false` gates the entire namespace (`status`/`log`/`revert`/`mark_for_add` all → `[SOURCE_CONTROL_DISABLED]`), so a caller burns calls discovering the dead end. Evidence from this task's call-log: 1 execute `get_provider {}` → None/disabled, then doomed `status` + `log` + a 3rd "re-confirm" `get_provider` = 3 redundant SC calls a documented precondition note would have prevented.
- `#2-asset-namespace-overlay-evidence` `OPEN` reporter — Broadening scope: the SAME undocumented gate also fronts the `asset.*` SC verbs, and the `docs/wiki-src/asset.md` overlay mentions source control **zero** times (`grep -i source.control asset.md` → no hits). New replay evidence from a DemoRoom content-hygiene task (check SC state → checkout → set/get metadata → submit changelist → re-check state on 4 materials): the caller naturally drove SC through the `asset` namespace, not `source_control`. Call-log shows **8 doomed SC calls** all blocked by the disabled provider — 4× `asset.get_source_control_state` (pre-checkout) → `[SC_DISABLED]`, then `source_control.get_provider` (which already exposed `isEnabled=false`), then `asset.source_control_checkout` → `[SOURCE_CONTROL_DISABLED]`, `asset.source_control_submit` (load-bearing) → `[SOURCE_CONTROL_DISABLED]`, and 2× `asset.get_source_control_state` (post-submit re-check) → `[SC_DISABLED]`. A one-line "gate on `get_provider.isEnabled` first; these 3 `asset.*` SC verbs are inert in a disabled-SC session" note in `asset.md` (mirroring the `source_control.md` note) collapses all of that to a single guarded check. (The inconsistent `[SC_DISABLED]` vs `[SOURCE_CONTROL_DISABLED]` spelling on the same gate is the error-vocabulary axis, already tracked in `E-error-code-vocabulary-registry` `#4`.) Named overlay to improve: `docs/wiki-src/asset.md` (add SC precondition for the asset SC verbs); `docs/wiki-src/source_control.md` already named in `#1`.
- `#3-rephrased` `OPEN` developer — Dropped the claim that no in-MCP verb brings a provider up: `source_control.connect` exists and skips the gate (`SourceControlHandler.cpp:126-132`). Dropped the `asset.get_source_control_state` evidence because that verb no longer exists. `asset.md:103` now documents disabled SC as a checkout preflight failure, so the `asset.md` ask is narrowed to the submit section (`:105-107`; the same gate is at `AssetWorkflowHandler.cpp:491-501`). Main ask: a `get_provider` then `connect` precondition note in `source_control.md`, which still mentions neither. Added Acceptance. Severity unchanged (Low).
