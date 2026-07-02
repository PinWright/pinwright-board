---
id: E-scs-add-component-not-idempotent
title: "blueprint.scs.add_component hard-errors on a duplicate name while sibling add verbs (add_variable/set_default/ensure_exists) are idempotent — breaks re-runnable provisioning scripts"
status: OPEN
severity: Low
category: ergonomic
tags: [add-verb-not-idempotent, scs, blueprint, add_component, idempotency, re-runnable]
encounters: 1
lastSeen: 2026-07-02T13:50:48.4836772+03:00
---

# `blueprint.scs.add_component` is not idempotent, unlike its sibling `blueprint.add*` verbs

The Blueprint authoring verbs disagree on what "add a thing that already
exists" means, which breaks the common "safe to re-run" provisioning-script
pattern:

- `blueprint.add_variable` on an existing variable → **success**, keyed with a
  note: `{"success":true, "note":"Variable already exists; no changes applied."}`
  (`Handlers/Blueprint/BlueprintPropertyHandler.cpp:138-140`).
- `blueprint.set_default` and `blueprint.ensure_exists` are likewise idempotent
  (re-invoke is a clean no-op / `created:false`).
- `blueprint.scs.add_component` on an existing component name → **hard error**
  `[SCS_ERROR] Component with name '<name>' already exists` (`is_error:true`).
  The duplicate-name check returns `success:false` with that message
  (`PinWright_SCSHandlers.cpp:694-698`), which `SendSCSResult` wraps into an
  `SCS_ERROR` (`SCSHandler.cpp:41-46`).

So an agent writing an idempotent "create-or-reuse" provisioning script can
freely re-call `add_variable`/`set_default`/`ensure_exists`, but must special-case
`scs.add_component`: probe `blueprint.scs.get` first and guard the add, or the
second run of the script aborts on the first component that already exists. The
inconsistency is the ergonomic defect — the tool's refusal itself is correct
(it is not a silent false-success and it does not corrupt anything), but callers
have no way to know one add verb throws where the others quietly succeed, and the
error is a hard `is_error` rather than a `changed:false`-style idempotent result.

**Severity rationale: impact=ergonomic-consistency/discoverability (correct refusal,
easy one-probe workaround) × reach=reasonable-but-not-every-session (re-runnable
provisioning scripts) -> Low.** Matches how the sibling
`E-scs-add-component-root-parent-rejected` (also a correct-but-awkward
`scs.add_component` diagnostics gap) was rated.

**Workaround:** before calling `blueprint.scs.add_component`, call
`blueprint.scs.get` and skip the add if a node of that name already exists
(the attempt agent did exactly this to make its script re-runnable).

**Fix (either):** (a) make `blueprint.scs.add_component` idempotent to match the
sibling add verbs — when a node of that `componentName` (same/compatible class)
already exists, return `success:true` with a note like
`"Component already exists; no changes applied."` and a `changed:false`/
`created:false` signal, instead of `success:false`; or (b) if a hard error is
intentional for a name collision, document on the method page that this verb —
unlike `add_variable`/`ensure_exists` — is NOT idempotent and must be guarded by
a prior `scs.get` for re-runnable scripts. Note the related handler
`BlueprintComponentHandler.cpp:414-419` already takes the graceful route for its
own component add (`success:true` + `warning:"Component already exists"`), so the
idempotent shape is already precedented in the codebase.

## Verbatim repro (live, replay-confirmed via `mcp__pinwright__call`)

Blueprint: `/Game/Pickups/BP_HealthPotion` (Actor), with an existing SCS node
`PotionMesh` (StaticMeshComponent) — confirmed via `blueprint.scs.get`
(`count:1`, `PotionMesh`).

1. `blueprint.scs.add_component` `{blueprintPath:"/Game/Pickups/BP_HealthPotion", componentClass:"StaticMeshComponent", componentName:"PotionMesh"}` (node already exists) →
   **`[SCS_ERROR] Component with name 'PotionMesh' already exists`** (`is_error:true`).
2. Contrast — `blueprint.add_variable` re-adding an existing variable →
   `{"success":true, "note":"Variable already exists; no changes applied."}` (no error).

The asymmetry (step 1 hard-errors, step 2 idempotent-succeeds) is the quotable
demonstration. Distinct from `E-scs-add-component-root-parent-rejected`
(parentComponentName="RootComponent" resolution) and
`E-scs-add-component-root-promotion-undocumented` (root-promotion docs) — this
ticket is about the duplicate-**name** re-add path being non-idempotent relative
to the sibling `add_variable`/`set_default`/`ensure_exists` verbs.

## History
- `#1-initial-repro` `OPEN` reporter — Seed `blueprint.ensure_exists`, idempotent collectible-pickup provisioning task on `/Game/Pickups/BP_HealthPotion`. All calls `ok:true` except the intentional idempotency-test re-add of `PotionMesh`, which returned `[SCS_ERROR] Component with name 'PotionMesh' already exists` (`is_error:true`). Replay-confirmed live against `mcp__pinwright__call`: the exact re-add reproduces the SCS_ERROR, while re-adding an existing variable via `blueprint.add_variable` returns `{"success":true,"note":"Variable already exists; no changes applied."}`. Verified against source: the duplicate-name guard hard-fails at `PinWright_SCSHandlers.cpp:694-698` (wrapped to SCS_ERROR at `SCSHandler.cpp:41-46`), whereas the idempotent variable path is `BlueprintPropertyHandler.cpp:138-140`; the graceful precedent already exists in `BlueprintComponentHandler.cpp:414-419` (`success:true` + `warning:"Component already exists"`). Filed E-/Low: correct refusal, easy `scs.get`-probe workaround, but a cross-verb idempotency inconsistency that breaks re-runnable provisioning scripts and forces callers to special-case this one add verb.
