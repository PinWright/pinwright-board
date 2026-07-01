---
id: E-ik-rig-family-wiki-advertises-compiled-out-workflow
title: "animation.authoring wiki advertises the IK Rig / IK Retargeter sub-workflow as live, but it hard-errors NOT_SUPPORTED — no module-prerequisite warning sends callers to read plugin C++"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [docs, animation, animation-authoring, ik-rig, ik-retargeter, retargeting, not-supported, discoverability, wiki]
---

# The wiki sells the IK Rig + IK Retargeter workflow as live; runtime hard-errors NOT_SUPPORTED

`docs/wiki-src/animation.authoring.md:14` lists the IK Rig / retargeter
family in the **Control rigs / IK** sub-workflow bullet as a live,
composable workflow:

> `create_ik_rig` + `add_ik_chain`; `create_ik_retargeter` +
> `set_retarget_chain_mapping`; `create_pose_library`.

and the namespace blurb (`:3`) advertises "control rigs / IK rigs /
retargeters." The per-method pages reinforce it — `create_ik_rig` reads
"Create a UIKRigDefinition asset (modern IK + retargeting source). Add
chains via `animation.authoring.add_ik_chain`, then pair two IK Rigs into
an IK Retargeter via `create_ik_retargeter` for cross-skeleton
retargeting workflows."

**Nowhere does any page state a module prerequisite or that the family
may be unavailable.** But in this plugin binary every verb in the family
hard-errors `[NOT_SUPPORTED] IK Rig module not available` at runtime
(compiled out behind `MCP_HAS_IKRIG` `__has_include` guards — see the
judge's `F-ik-rig-retargeter-family-not-compiled` for the build/code root
cause and fix). The wiki is the only thing a caller reads before acting,
and it gives no signal that the whole documented path is a dead end.

## Why this is a distinct PROCESS angle (not a re-file of the F- ticket)

The judge's `F-ik-rig-retargeter-family-not-compiled` is the **capability**
ticket: its action is "add `IKRig`/`IKRigEditor`/`IKRigDeveloper` to
`Build.cs` and implement the stub handlers." That is a build + C++ change
and may or may not ever ship.

This ticket is the **docs-overlay process fix that stands on its own
regardless of whether the feature ships**: today the namespace page
advertises a workflow that cannot run, with no caveat, which is exactly
what converted a well-specified task into wasted process. It is the same
shape and same namespace as the already-accepted precedent
`E-set-transition-rules-wiki-overstates-rule-authoring` (wiki over-promises
an `animation.authoring` capability the runtime does not deliver; judge
filed the outcome, auditor filed the docs angle). Until/unless the F-
ticket lands, the honest-caveat sentence is the cheap, independent win.

## Friction evidence (this task — animation.authoring, 11 calls, the cross-skeleton retargeting story)

The friction came entirely from trusting the wiki. The agent did the
right thing per the docs: read the family's wiki pages, confirmed both
source meshes exist, then tried to execute the documented workflow.

- **4 wiki-nav reads** up front to learn the documented chain
  (`create_ik_rig`, `add_ik_chain`, `create_ik_retargeter`,
  `set_retarget_chain_mapping` pages) — all advertising it as live.
- **3 deterministic `[NOT_SUPPORTED]` failures**: `create_ik_rig` for
  `IKR_Spider`, then for `IKR_DinoDragon`, then a **deliberate replay of
  `IKR_Spider`** purely to confirm the error was deterministic and not a
  transient — a retry the docs could have made unnecessary.
- A fall-back **read of the plugin C++**
  (`AnimationAuthoringHandler_AnimBlueprint.cpp` ~L3134-3173) to discover
  the `MCP_HAS_IKRIG` compile-time gate — the *only* place the prerequisite
  is discoverable.

Friction note verbatim: *"the wiki documents the full create_ik_rig ->
add_ik_chain -> create_ik_retargeter -> set_retarget_chain_mapping
workflow with no hint of a module prerequisite, yet the live handler
hard-errors NOT_SUPPORTED; had to fall back to reading the plugin C++ ...
to discover the whole IK family is gated behind compile-time MCP_HAS_IKRIG
... a real discoverability gap, and no MCP-reachable way to enable it."*

The over-advertisement is the worst kind of doc gap: it does not merely
omit a limitation, it positively presents a non-functional path as the
canonical one.

## Fix (wiki overlay — downstream wiki process, not this audit)

Edit `docs/wiki-src/animation.authoring.md`:

1. Line 14 (the **Control rigs / IK** bullet): mark the IK Rig /
   retargeter sub-family (`create_ik_rig`, `add_ik_chain`,
   `create_ik_retargeter`, `set_retarget_chain_mapping`) as
   **conditionally compiled** — it returns `[NOT_SUPPORTED] IK Rig module
   not available` in builds where the IKRig editor modules are not on the
   `Build.cs` dependency list. State plainly that callers should not
   assume the family is live; cross-reference
   `F-ik-rig-retargeter-family-not-compiled` (the build/code enablement +
   stub implementation work).
2. Line 3 (namespace blurb): soften "control rigs / IK rigs / retargeters"
   so it does not imply the IK Rig/retargeter path is always available.
3. Optionally surface the same caveat on the four per-method overlay pages
   so a caller who lands directly on `create_ik_rig` sees the prerequisite
   without having to read the namespace page or the plugin source.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `animation.authoring.create_ik_retargeter` cross-skeleton retargeting
  task (11 calls; judge filed `F-ik-rig-retargeter-family-not-compiled`
  for the capability/build root cause). PROCESS finding distinct from the
  judge's feature ticket: `docs/wiki-src/animation.authoring.md:14` (+ the
  `:3` blurb and the `create_ik_rig`/`add_ik_chain`/`create_ik_retargeter`/
  `set_retarget_chain_mapping` per-method pages) advertise the IK Rig / IK
  Retargeter sub-workflow as a live, composable path with **no
  module-prerequisite warning**, yet every verb hard-errors
  `[NOT_SUPPORTED] IK Rig module not available` at runtime. The
  over-advertisement is what cost process: 4 up-front wiki-nav reads of the
  documented chain, then 3 deterministic `[NOT_SUPPORTED]` failures
  (including a deliberate `create_ik_rig` replay just to confirm
  determinism), then a fall-back read of
  `AnimationAuthoringHandler_AnimBlueprint.cpp` (~L3134-3173) — the only
  place the `MCP_HAS_IKRIG` compile-time gate is discoverable. Fix is
  wiki-overlay-only and stands independent of whether the F- capability
  ticket ever lands: caveat line 14 (and the per-method pages) that the IK
  Rig/retargeter family is conditionally compiled and may return
  `NOT_SUPPORTED`, cross-referencing `F-ik-rig-retargeter-family-not-compiled`.
  Deduped: distinct from that judge feature ticket (build/code + stub
  implementation, `category: feature`) — this is the docs-process angle,
  same shape as the accepted precedent
  `E-set-transition-rules-wiki-overstates-rule-authoring` in the same
  namespace. No existing `E-`/docs ticket covers the IK family wiki page.
  Severity Medium — the page actively misdirects to a dead-end path on a
  core, well-specified retargeting task.
- `#2-already-fixed` `IN-REVIEW` developer — Already resolved in current
  source by the sibling capability fix `F-ik-rig-retargeter-family-not-compiled`
  (commit `5762be2`, confirmed ancestor of plugin HEAD in this baseline). That
  GO commit landed BOTH halves the E-ticket worried about: (1) the CAPABILITY —
  `EditorAutomationRpcGateway.Build.cs:141-142` now
  `TryAddConditionalModule(... "IKRig")` / `"IKRigEditor"`, and the master guard
  `AnimationAuthoringHandler_AnimBlueprint.cpp:86` probes
  `#if __has_include("Rig/IKRigDefinition.h")` (UE 5.7 ships that header under
  `Plugins/Animation/IKRig`), so `MCP_HAS_IKRIG==1` and the handlers run their
  live bodies instead of the `SendError("NOT_SUPPORTED", "IK Rig module not
  available")` fall-throughs at `:3170`/`:3247` — the documented workflow is no
  longer a dead end in the shipped build; (2) the DOCS caveat that is Fix item 3
  — the SAME commit added the per-method overlay section
  `### animation.authoring.create_ik_rig / add_ik_chain / create_ik_retargeter /
  set_retarget_chain_mapping` at `docs/wiki-src/animation.authoring.md:153-191`,
  whose `:157` `**Module prerequisite.**` paragraph states the family returns
  `[NOT_SUPPORTED]` if built without `IKRig`/`IKRigEditor` and that "the shipped
  build links them." So the ticket's central premise (every verb hard-errors
  NOT_SUPPORTED / wiki sells a dead end / no module-prerequisite warning) is now
  false in current source, and the honest-caveat win it sought is already
  written. The only residual is cosmetic (line 14 sub-workflow bullet, line 3
  namespace blurb) and is no longer load-bearing — the path it lists is live; the
  build-config nuance for custom rebuilds is already captured at `:157`. No code,
  no compile — flipping OPEN -> IN-REVIEW for a tester to verify the family
  compiles in and the caveat renders on the per-method pages against the current
  binary.
