---
id: E-rpc-cull-151-record
title: "RECORD: RPC-removal cull of 151 methods (55 stubs + 12 session + 84 superseded) from the PinWright plugin"
status: DONE
severity: Medium
category: ergonomic
tags: [rpc-cull, removal, record, stub, superseded, session, surface-area]
---

# RECORD: 151-method RPC cull

This is an **append-only record/log entry**, not a fix ticket. It documents the
bulk removal of 151 RPC methods from the PinWright plugin
(`Plugins/PinWright`), executed by the RPC-removal plan agents. It is filed
directly at `DONE` because it records completed removal work; it is outside the
OPEN -> IN-REVIEW -> DONE fix lifecycle (there is nothing to verify-then-close —
the removal is the fact being recorded). It exists so that anyone later asking
"what happened to `X`?" or "why did the tool surface shrink?" has one authoritative
inventory with per-method replacements.

## Counts

**55 stubs + 12 session + 84 superseded = 151 methods removed.**

- **55 stubs** — fake-success or NOT_IMPLEMENTED tombstones: handlers that
  reported `success:true` (or a guided NOT_IMPLEMENTED) while doing nothing, or
  worse, mutating the wrong state. Source audit manifest: the stub audit
  (67 namespaces, 55 flagged).
- **12 session** — game-runtime online-session / split-screen / LAN / voice-chat
  echo & voice methods that never belonged in an editor automation plugin.
- **84 superseded** — real but outdated methods (mostly inherited from the
  original Unreal_mcp import) each fully covered by a newer PinWright method.
  Source audit manifest: the supersession audit (66 namespaces, 84 flagged).

## Dependency drop

Removing the 12 session methods let the plugin drop three online/voice modules
from both `PinWright.uplugin` and `PinWright.Build.cs`:
**`OnlineSubsystem`, `OnlineSubsystemUtils`, `VoiceChat`**. These were pulled in
solely by the culled session/voice handlers; no remaining handler references them.

## Removed methods

### Stubs (55)

Grouped by namespace (count in parentheses):

- **ai** (4): `ai.configure_bt_node`, `ai.set_perception_team`, `ai.set_focus`, `ai.clear_focus`
- **animation** (1): `animation.authoring.set_axis_settings` — reimplementation tracked in [`F-anim-set-axis-settings-proper`](F-anim-set-axis-settings-proper.md)
- **blueprint** (1): `blueprint.probe_subobject_handle`
- **debug** (1): `debug.spawn_category`
- **editor** (3): `editor.set_immersive_mode`, `editor.set_fixed_delta_time`, `editor.set_editor_mode` — reimplementations tracked in [`F-editor-set-immersive-mode-proper`](F-editor-set-immersive-mode-proper.md), [`F-editor-set-fixed-delta-time-proper`](F-editor-set-fixed-delta-time-proper.md), [`F-editor-set-editor-mode-proper`](F-editor-set-editor-mode-proper.md)
- **environment** (2): `environment.build.export_snapshot`, `environment.build.import_snapshot`
- **gas** (7): `gas.set_attribute_clamping`, `gas.set_ability_targeting`, `gas.add_ability_task`, `gas.configure_cue_trigger`, `gas.set_cue_effects`, `gas.add_ability`, `gas.grant_ability`
- **geometry** (2): `geometry.set_lod_screen_sizes` — reimplementation tracked in [`F-geometry-lod-screen-sizes-proper`](F-geometry-lod-screen-sizes-proper.md); `geometry.triangulate`
- **input** (4): `input.set_input_trigger`, `input.set_input_modifier`, `input.enable_input_mapping`, `input.disable_input_action`
- **interaction** (4): `interaction.configure_trigger_events`, `interaction.configure_interaction_trace_on_actor`, `interaction.configure_interaction_widget_on_actor`, `interaction.create_interaction_component_on_actor`
- **level** (3): `level.set_world_settings`, `level.set_lighting`, `level.structure.configure_level_bounds`
- **log** (2): `log.subscribe`, `log.unsubscribe`
- **material** (2): `material.authoring.set_material_parameter`, `material.authoring.set_cast_shadows`
- **misc** (2): `misc.set_viewport_resolution`, `misc.create_bookmark`
- **networking** (2): `networking.configure_net_serialization`, `networking.configure_net_driver`
- **physics** (1): `physics.configure_vehicle` — real typed vehicle authoring already exists as `vehicle.*` (see [`F-vehicle-chaos-typed`](F-vehicle-chaos-typed.md))
- **pipeline** (1): `pipeline.list_categories`
- **skeleton** (3): `skeleton.preview_physics`, `skeleton.mirror_weights`, `skeleton.import_morph_targets`
- **spline** (2): `spline.configure_mesh_spacing`, `spline.configure_mesh_randomization`
- **system** (4): `system.inspect.get_project_settings`, `system.inspect.get_editor_settings`, `system.inspect.get_performance_stats`, `system.inspect.get_memory_stats`
- **texture** (4): `texture.import_texture`, `texture.create_cube_texture`, `texture.create_volume_texture`, `texture.create_texture_array`

Five stubs removed a **legitimately-wanted capability** (the impl was fake, not the
idea). Proper reimplementations are filed as OPEN feature tickets:
[`F-geometry-lod-screen-sizes-proper`](F-geometry-lod-screen-sizes-proper.md),
[`F-anim-set-axis-settings-proper`](F-anim-set-axis-settings-proper.md),
[`F-editor-set-fixed-delta-time-proper`](F-editor-set-fixed-delta-time-proper.md),
[`F-editor-set-immersive-mode-proper`](F-editor-set-immersive-mode-proper.md),
[`F-editor-set-editor-mode-proper`](F-editor-set-editor-mode-proper.md).

### Session (12)

All in the `session` namespace — game-runtime online/split-screen/LAN/voice:

`configure_local_session_settings`, `configure_session_interface`,
`configure_split_screen`, `set_split_screen_type`, `configure_lan_play`,
`join_lan_server`, `enable_voice_chat`, `configure_voice_settings`,
`set_voice_channel`, `mute_player`, `set_voice_attenuation`,
`configure_push_to_talk`.

The `session` namespace **keeps**: `host_lan_server`, `add_local_player`,
`remove_local_player`, `get_sessions_info`.

### Superseded (84)

Grouped by namespace (count in parentheses); `method -> replacement`:

**actor** (2)
- `actor.call_function` -> `object.call_function`
- `actor.get_metadata` -> `actor.get`

**ai** (9)
- `ai.create_behavior_tree` -> `behavior_tree.create`
- `ai.configure_sight_config` -> `ai.set_ai_perception`
- `ai.configure_hearing_config` -> `ai.set_ai_perception`
- `ai.configure_damage_sense_config` -> `ai.set_ai_perception`
- `ai.add_ai_perception_component` -> `blueprint.scs.add_component` (or `ai.set_ai_perception`, which find-or-creates the same node)
- `ai.setup_perception` -> `ai.set_ai_perception`
- `ai.create_ai_controller` -> `blueprint.create`
- `ai.create_nav_link_proxy` -> `blueprint.create`
- `ai.get_blackboard_value` -> `ai.get_ai_info` (blackboardPath branch) and `property.get`

**animation** (2)
- `animation.create_anim_blueprint` -> `animation.create_animation_bp`
- `animation.setup_ik` -> `animation.authoring.create_control_rig` (asset creation); `animation.authoring.create_ik_rig` + `animation.authoring.add_ik_chain` (IK authoring); `controlrig.compile_crir` (rig graph content)

**asset** (10)
- `asset.create_material` -> `material.authoring.create_material`
- `asset.create_material_instance` -> `material.authoring.create_material_instance` (plus `material.authoring.set_material_instance_parent` for MIC-of-MIC parents)
- `asset.add_material_parameter` -> `material.authoring.add_scalar_parameter` / `add_vector_parameter` / `add_static_switch_parameter` / `add_texture_sample`
- `asset.add_material_node` -> `material.graph.add_expression` (with 'properties'), `material.authoring.add_material_node`, `material.graph.create_nodes` (batch)
- `asset.connect_material_pins` -> `material.graph.connect_nodes`
- `asset.remove_material_node` -> `material.graph.remove_node` (also `material.authoring.remove_material_node`)
- `asset.break_material_connections` -> `material.graph.break_connections`
- `asset.get_material_node_details` -> `material.graph.get_node_details` (also `material.authoring.get_material_node_details`)
- `asset.generate_report` -> `asset.list` (and `asset.search_assets` for multi-class filters)
- `asset.get_source_control_state` -> `source_control.status`

**audio** (1)
- `audio.authoring.create_submix_effect` -> `audio.authoring.create_sound_submix`

**blueprint** (4)
- `blueprint.insert_code_at_node` -> `blueprint.insert_bpir_at_node`
- `blueprint.undo_last_compile` -> `blueprint.undo_last_bpir`
- `blueprint.add_node` -> `blueprint.graph.create_node` (bulk authoring: `blueprint.compile_bpir`)
- `blueprint.connect_pins` -> `blueprint.graph.connect_pins`

**character** (5)
- `character.set_walk_speed` -> `character.configure_movement_speeds` (walkSpeed param); also `property.set`
- `character.set_ground_friction` -> `character.configure_movement_speeds` (groundFriction param); also `property.set`
- `character.set_braking_deceleration` -> `character.configure_movement_speeds` (deceleration param); also `property.set`
- `character.set_jump_height` -> `character.configure_jump` (jumpHeight param); also `property.set`
- `character.set_gravity_scale` -> `character.configure_jump` (gravityScale param); also `property.set`

**editor** (2)
- `editor.show_stats` -> `performance.show_stats` (plus `performance.show_fps` for the FPS half; `editor.console_command` as the generic path)
- `editor.hide_stats` -> `editor.console_command` (command='Stat None'); equivalently `performance.show_stats` (category='none')

**effect** (6)
- `effect.create_volumetric_fog` -> `effect.spawn_niagara` (one-call strict superset); `niagara.spawn_actor`
- `effect.create_impact_effect` -> `effect.spawn_niagara`; `niagara.spawn_actor`
- `effect.create_environment_effect` -> `effect.spawn_niagara`; `niagara.spawn_actor`
- `effect.create_particle_trail` -> `effect.spawn_niagara`; `niagara.spawn_actor`
- `effect.create_niagara_ribbon` -> `niagara.create_ribbon` (purpose-built ribbon spawn); `effect.spawn_niagara` (generic superset)
- `effect.create_dynamic_light` -> `lighting.spawn_light` (primary); `actor.spawn` + `actor.set_component_properties` (PIE-world spawn case)

**environment** (4)
- `environment.control.console_command` -> `editor.console_command` (editor-world scope) and `system.console_command` (process/GEngine scope)
- `environment.build.delete` -> `actor.delete`
- `environment.build.bake_lightmap` -> `level.build_lighting`
- `environment.build.create_fog_volume` -> `actor.spawn`

**game_framework** (4)
- `game_framework.set_default_pawn_class` -> `blueprint.set_default` (alternatively `property.set` + `asset.save`)
- `game_framework.set_player_controller_class` -> `blueprint.set_default` (alternatively `property.set` + `asset.save`)
- `game_framework.set_game_state_class` -> `blueprint.set_default` (alternatively `property.set` + `asset.save`)
- `game_framework.set_player_state_class` -> `blueprint.set_default` (alternatively `property.set` + `asset.save`)

**gas** (1)
- `gas.add_ability_system_component` -> `blueprint.scs.add_component`

**geometry** (4)
- `geometry.convert_to_nanite` -> `geometry.convert_to_static_mesh` + `render.nanite_rebuild_mesh` (or `asset.nanite_rebuild_mesh` for finer settings)
- `geometry.generate_lods` -> `asset.generate_lods` (assetPath arm; batch-capable), plus `geometry.convert_to_static_mesh` + `asset.generate_lods` for the actorName arm
- `geometry.quadrangulate` -> `geometry.remesh_uniform`
- `geometry.remesh_voxel` -> `geometry.remesh_uniform` + `geometry.fill_holes`

**level** (5)
- `level.import` -> `level.duplicate` (also `asset.duplicate`)
- `level.build_level_lighting` -> `level.build_lighting`
- `level.add_to_world` -> `level.add_sublevel`
- `level.spawn_light` -> `lighting.spawn_light` + `lighting.spawn_sky_light` (`actor.spawn` gives exact parity)
- `level.structure.create_sublevel` -> `level.add_sublevel` (with `level.create` / `level.structure.create_level` to author the sublevel asset first)

**material** (3)
- `material.authoring.disconnect_nodes` -> `material.graph.break_connections`
- `material.authoring.add_material_node` -> `material.graph.add_node` (and `material.graph.add_expression` for property-initialized adds)
- `material.authoring.remove_material_node` -> `material.graph.remove_node`

**niagara** (3)
- `niagara.graph.add_module` -> `niagara.add_module`
- `niagara.graph.set_parameter` -> `niagara.set_parameter`
- `niagara.save` -> `asset.save`

**sequencer** (6)
- `sequencer.duplicate` -> `asset.duplicate`
- `sequencer.rename` -> `asset.rename`
- `sequencer.delete` -> `asset.delete`
- `sequencer.list` -> `asset.list`, `asset.search`
- `sequencer.get_metadata` -> `asset.get`
- `sequencer.open` -> `editor.open_asset`

**skeleton** (1)
- `skeleton.set_physics_constraint` -> `skeleton.add_physics_constraint` (create) + `skeleton.configure_constraint_limits` (update)

**spline** (1)
- `spline.create_spline_mesh_component` -> `blueprint.scs.add_component` (+ `blueprint.scs.set_property` for a non-default forwardAxis)

**system** (3)
- `system.inspect.get_world_settings` -> `editor.status` (plus `level.get_info` for per-level metadata)
- `system.inspect.get_scene_stats` -> `system.inspect.list_actor_classes` (census) / `system.inspect.list_objects` (totalMatches)
- `system.call_subsystem` -> `object.call_function` (path discovery via `system.inspect.list_subsystems`)

**ui** (4)
- `ui.play_in_editor` -> `editor.play`
- `ui.stop_play` -> `editor.stop`
- `ui.save_all` -> `editor.save_all`
- `ui.simulate_input` -> `editor.simulate_input` (raw injection); `drive.key` (element-focused / modifier presses)

**volume** (2)
- `volume.remove_volume` -> `actor.delete`
- `volume.create_trigger_box` -> `actor.spawn`

**widget** (2)
- `widget.set_widget_parent_class` -> `blueprint.reparent`
- `widget.create_style` -> `blueprint.add_variable`

## Superseded prior tickets (regression tests removed with the method)

Several stub methods had previously been patched to "fail loud" (NOT_IMPLEMENTED)
by their own board tickets, with dedicated regression tests pinning that behavior.
The cull removed the methods outright, so those fix tickets are moot and their
regression tests were deleted alongside the handlers. Each has been annotated in
place (append-only):

- [`B-probe-subobject-handle-ignores-class`](B-probe-subobject-handle-ignores-class.md) — `blueprint.probe_subobject_handle` (stub); tests `BogusClassErrors` / `NonComponentClassErrors` in `TestBlueprintHandlers.cpp` removed.
- [`B-export-snapshot-empty-stub`](B-export-snapshot-empty-stub.md) — `environment.build.export_snapshot` / `import_snapshot` (stubs); `TestEnvironmentSnapshotStubsNotImplemented.cpp` + related cases removed.
- [`B-widget-apply-style-silent-noop`](B-widget-apply-style-silent-noop.md) — `widget.apply_style` NOT_IMPLEMENTED tombstone (coupled with the `widget.create_style` -> `blueprint.add_variable` supersession); `TestWidgetApplyStyleNotImplemented.cpp` removed.
- [`B-spawn-category-silent-noop-fake-existsafter`](B-spawn-category-silent-noop-fake-existsafter.md) — `debug.spawn_category` (stub); `NoFakeExistsAfterFailsLoud` / `BogusNameFailsLoud` in `TestDebugHandlers.cpp` removed.
- [`F-vehicle-chaos-typed`](F-vehicle-chaos-typed.md) — the physics.configure_vehicle rejected-candidates entry: `physics.configure_vehicle` (stub) removed; its 4 tests in `TestPhysicsHandlers.cpp` removed. Real typed vehicle authoring lives on in `vehicle.*`. (The plugin-source `docs/rpc-hard-removal-rejected-candidates.md:143` had previously rejected removing it; that rejection assumed the `CreateVehicle` exec commands worked — they do not exist — so it was void.)

## History
- `#1-cull-record` `DONE` reporter — Record entry (filed directly at DONE; a record, not a fix-lifecycle ticket). Documents the RPC-removal cull of 151 methods: 55 stubs (fake-success / NOT_IMPLEMENTED tombstones), 12 `session.*` game-runtime online/split-screen/LAN/voice methods, and 84 superseded legacy methods each covered by a newer PinWright method. Complete per-namespace inventory with replacements pasted above, parsed from the two audit manifests (stub audit: 67 namespaces / 55 flagged; supersession audit: 66 namespaces / 84 flagged). Dependency drop noted: OnlineSubsystem, OnlineSubsystemUtils, VoiceChat removed from PinWright.uplugin + Build.cs. Five legitimately-wanted capabilities whose impl was fake were re-filed as OPEN reimplementation tickets (F-geometry-lod-screen-sizes-proper, F-anim-set-axis-settings-proper, F-editor-set-fixed-delta-time-proper, F-editor-set-immersive-mode-proper, F-editor-set-editor-mode-proper). Five superseded prior tickets (B-probe-subobject-handle-ignores-class, B-export-snapshot-empty-stub, B-widget-apply-style-silent-noop, B-spawn-category-silent-noop-fake-existsafter, F-vehicle-chaos-typed) annotated in place that their regression tests were removed with their methods.
