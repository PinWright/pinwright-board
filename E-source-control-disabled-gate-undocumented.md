---
id: E-source-control-disabled-gate-undocumented
title: "SC-disabled precondition gate undocumented in BOTH the source_control AND asset wiki overlays (SC-off → every SC verb errors)"
status: OPEN
severity: Low
category: ergonomic
tags: [source-control, docs]
encounters: 2
lastSeen: 2026-06-23T10:08:23Z
---

# `source_control` wiki overlay doesn't document the provider precondition gate

The `source_control` namespace has a hard precondition that is invisible until
you trip it: when no provider is connected — `source_control.get_provider`
returns `{providerName:"None", isEnabled:false, isAvailable:false}` — **every
other verb** (`status`, `log`, `revert`, `mark_for_add`) fails with
`[SOURCE_CONTROL_DISABLED] Source control is not enabled`, and there is no
in-MCP way to bring a provider up (the capability gap itself is tracked
separately in `F-source-control-namespace-followup`).

The wiki overlay (`docs/wiki-src/source_control.md`) says nothing about this
gate. It currently reads as a flat list of usable verbs:

> Query and mutate the editor's source-control provider state — read the active
> provider, list file status, view log/history, revert local changes, and mark
> new files for add.

A caller reading that has no way to know the namespace is **inert in a
disabled-SC session** until it has already issued the doomed calls.

## What's awkward

This is a docs/process gap, not a verb bug — the disabled errors are clean and
correct. The friction is that the precondition isn't surfaced where a caller
plans the task, so the caller burns calls discovering a dead end instead of
gating on `get_provider.isEnabled` once and stopping. The signal already exists
in the `get_provider` payload (`isEnabled` / `isAvailable`); it just isn't
documented as the gate to check first.

## What it should do

Add a short precondition note to the `docs/wiki-src/source_control.md` overlay:

- **Check `get_provider` first.** If `isEnabled` is `false` (or `providerName`
  is `"None"`), the whole namespace is unusable in this session: `status`,
  `log`, `revert`, and `mark_for_add` will all return
  `[SOURCE_CONTROL_DISABLED]`.
- There is currently **no in-MCP verb to connect/enable a provider**; the
  provider must already be brought up in the editor UI (cross-link
  `F-source-control-namespace-followup` for the planned `connect` verb).

This lets a caller short-circuit after one `get_provider` call instead of
probing `status`/`log` and re-confirming the provider.

The **same gate also fronts the `asset.*` source-control verbs**
(`asset.get_source_control_state`, `asset.source_control_checkout`,
`asset.source_control_submit`), which live in the `asset` namespace, not
`source_control` — and the `docs/wiki-src/asset.md` overlay has **zero**
mention of source control or the precondition. A content/asset-hygiene caller
naturally reads `asset.md` (they're working with assets), so the note is
needed there too: state that all three `asset.*` SC verbs require an enabled
provider and tell the caller to gate on `source_control.get_provider.isEnabled`
once before issuing any of them. Both overlays (`source_control.md` and
`asset.md`) need the same one-line precondition note.

## Evidence (this task: "revert M_Button_Emissive to its committed baseline")

The task's friction note: *"I hit a hard capability/config gap: no provider is
configured in the editor and the source_control namespace exposes no
connect/enable method ... there is no in-MCP way to bring up git and complete
the revert round-trip."* Discovery itself was reported smooth — the gap is that
nothing warned about the disabled-provider gate before the calls were made.

Call-log shows the wasted steps caused by the undocumented gate: after the
first execute `get_provider {}` already returned `None`/disabled, the agent
still issued `source_control.status` (→ `[SOURCE_CONTROL_DISABLED]`),
`source_control.log` (→ `[SOURCE_CONTROL_DISABLED]`), and a **third**
`get_provider` ("re-confirm") — i.e. 3 redundant/doomed SC calls past the point
the disabled state was already known. A documented "gate on `get_provider` first"
note collapses that to one call.

## History
- `#1-initial-audit` `OPEN` reporter — Process/docs friction distinct from the capability gap in `F-source-control-namespace-followup` (which proposes adding `connect`). Here: the `docs/wiki-src/source_control.md` overlay never states that `get_provider.isEnabled=false` gates the entire namespace (`status`/`log`/`revert`/`mark_for_add` all → `[SOURCE_CONTROL_DISABLED]`), so a caller burns calls discovering the dead end. Evidence from this task's call-log: 1 execute `get_provider {}` → None/disabled, then doomed `status` + `log` + a 3rd "re-confirm" `get_provider` = 3 redundant SC calls a documented precondition note would have prevented.
- `#2-asset-namespace-overlay-evidence` `OPEN` reporter — Broadening scope: the SAME undocumented gate also fronts the `asset.*` SC verbs, and the `docs/wiki-src/asset.md` overlay mentions source control **zero** times (`grep -i source.control asset.md` → no hits). New replay evidence from a DemoRoom content-hygiene task (check SC state → checkout → set/get metadata → submit changelist → re-check state on 4 materials): the caller naturally drove SC through the `asset` namespace, not `source_control`. Call-log shows **8 doomed SC calls** all blocked by the disabled provider — 4× `asset.get_source_control_state` (pre-checkout) → `[SC_DISABLED]`, then `source_control.get_provider` (which already exposed `isEnabled=false`), then `asset.source_control_checkout` → `[SOURCE_CONTROL_DISABLED]`, `asset.source_control_submit` (load-bearing) → `[SOURCE_CONTROL_DISABLED]`, and 2× `asset.get_source_control_state` (post-submit re-check) → `[SC_DISABLED]`. A one-line "gate on `get_provider.isEnabled` first; these 3 `asset.*` SC verbs are inert in a disabled-SC session" note in `asset.md` (mirroring the `source_control.md` note) collapses all of that to a single guarded check. (The inconsistent `[SC_DISABLED]` vs `[SOURCE_CONTROL_DISABLED]` spelling on the same gate is the error-vocabulary axis, already tracked in `E-error-code-vocabulary-registry` `#4`.) Named overlay to improve: `docs/wiki-src/asset.md` (add SC precondition for the asset SC verbs); `docs/wiki-src/source_control.md` already named in `#1`.

