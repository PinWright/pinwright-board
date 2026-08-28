---
id: B-verbs-read-undeclared-parameters
title: "66 verbs read parameters their RPC_PARAMS never declares, so the dispatcher rejects those keys before the handler runs"
status: IN-REVIEW
severity: High
category: bug
tags: [dispatcher, unknown-params, undeclared-parameter, param-spec, unreachable-code, sweep]
encounters: 1
lastSeen: 2026-08-28
---

# A mechanized sweep found 66 dead parameters across 36 verbs

`B-niagara-set-parameter-emitter-scope-unreachable` was one instance of a class. A verb whose handler
reads a key its `RPC_PARAMS` block does not declare is unreachable through that key: the dispatcher
rejects the payload with `UNKNOWN_PARAMS` before the handler body runs. The handler code is correct and
dead.

Found by parsing all 1,208 `REGISTER_RPC_HANDLER` bodies, extracting the literal keys passed to
`Ctx.Get*` / `Ctx.Require*`, and diffing against the accepted-name set (Name + Aliases + TypedAliases).

**66 pairs across 36 verbs, high confidence — read directly in the handler body, each site eyeballed.**
The exact list is the baseline in `Tests/Infra/TestDeclaredParamCoverage.cpp`; the clusters:

| verbs | undeclared keys |
|---|---|
| `behavior_tree.{add_node,connect_nodes,remove_node,break_connections,set_node_properties}` | `behaviorTreePath`, `path` — the whole namespace's alternate path spellings |
| `sequencer.set_playhead` | `sequencePath`, `sequence_path`, `openIfNeeded`, `open_if_needed`, `force_update`, `update_method` |
| `camera.animation_shots` | `force_update`, `hide_editor_sprites`, `openIfNeeded`, `open_if_needed`, `restore_playhead`, `update_method` |
| `skeleton.*` (8 verbs) | `meshPath`, `skeletonPath`, `boneName`, `bone_name`, `nameFilter`, `name_filter`, `parentBoneName`, `newParentBone`, `verbose` |
| `audio.synth.{patch,variations,export}` | `candidate_id`, `id`, `rootSeed`, `vary_seed` |
| ~20 singles | incl. `actor.set_collision:actor_name`, `actor.describe:componentsMode/field/include_components`, `editor.open_level:path`, `data_table.create:rowStruct/savePath`, `niagara.graph.get:emitterName/scriptType`, `animation.{play_montage,add_notify}:assetPath` |

**A further 88 pairs across 35 verbs are read inside a shared helper taking `FHandlerContext&` — medium
confidence**, verified by grep at the definition site rather than by call-graph proof: `drive.*` read
`browser_index` through `*Web`/`RunAction`/`ParseRootSelector`; the 7 `game_framework.configure_*` read
`blueprintPath`/`name`/`path` through `FCommonParams::Extract`; 4 `audio.authoring.*` through
`BuildMetaSoundLiteralFromParams`; 6 `chooser.*`; and others.

**One confirmed false positive, so nobody chases it:** `widget.describe` / `widget.export_xml` calling
`FWidgetGeometryResolver::ParseRequest` (which takes no `FHandlerContext&`) bare-name-resolved to the
unrelated `ParseRequest` in `LocalizationHandler.cpp`.

**Not all of these are necessarily defects.** Some may be deliberate — a key read opportunistically from
the raw payload, or a spelling nobody intends to support. Each needs a decision: declare it, or delete
the read. What is not defensible is the current state, where the handler reads a key no caller can send.

`PinWright.infra.declared_params.HandlersOnlyReadDeclaredParams` now fails on any pair outside this
66-entry baseline and warns when a baseline entry stops reproducing, so the list can only shrink.

**What is left.** 60 of the 66 are decided and the baseline is down to 6, all of them
`sequencer.set_playhead` (`sequencePath`, `sequence_path`, `openIfNeeded`, `open_if_needed`,
`force_update`, `update_method`), deliberately untouched here because `Sequencer/SequenceHandler.cpp`
was owned by a concurrent workstream. The ~88 medium-confidence helper-read pairs are still out of
scope and still unmeasured by the test (they are the KNOWN LIMITS section's first bullet, not baseline
entries). Two adjacent instances the scanner cannot see, found while fixing these and left alone:
`actor.set_collision` reads `collision_enabled` straight off `GetRawPayload()` and does not declare
it, and `behavior_tree.attach_decorator` / `attach_service` reach the same
`{assetPath, behaviorTreePath, path}` list through `HandleAttachBTSubNode` while declaring only
`assetPath`.

## History
- `#1-mechanized-sweep` `OPEN` reporter — Swept while fixing `B-test-invokehandler-bypasses-param-gate`,
  which explains why the class survived: `TestUtils.h`'s `InvokeHandler` skips the dispatcher's
  parameter gate, so tests written that way cannot observe it. Verified by porting the C++ scanner to
  Python and running it: 1,208 registrations recovered, exactly 66 pairs, zero extras.
- `#2-namespace-sweep-resolved` `IN-REVIEW` developer — "Resolved 60 of 66 pairs across 21 files;
  baseline shrunk from 66 to 6"
