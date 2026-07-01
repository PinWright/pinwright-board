---
id: F-niagara-generic-particle-support
title: "Generic Niagara inspection, editing, and particle asset dumps"
status: DONE
severity: High
category: feature
tags: [niagara, particles, asset-dump, cascade]
---

# Generic Niagara inspection, editing, and particle asset dumps

The particle-system MCP surface was misleading: specialized Niagara helper
methods exposed only a small set of hard-coded operations, some handlers
returned success without real asset mutation, and `asset.dump` did not export
Niagara graph, stack, renderer, parameter, or Cascade data.

Replace the specialized surface with generic read-first Niagara tooling,
one-operation edit RPCs, complete Niagara asset dump aspects, and read-only
Cascade dumps. Remove the old helper namespaces outright instead of keeping
compatibility aliases.

## History
- `#1-generic-particle-request` `OPEN` reporter — User requested a total rework of particle-system MCP support: remove overly specialized Niagara helper methods, add generic view/edit coverage for editor-exposed data, include full Niagara graph/stack data in `asset.dump`/`asset.dump_folder`, support Cascade dump-only output, and add strong tests.
- `#2-niagara-cascade-rework` `IN-REVIEW` developer — Added Niagara dump builder and `asset.dump` aspect files (`niagara_system.json`, `niagara_emitters.json`, `niagara_parameters.json`, `niagara_stack.json`, `niagara_graphs.json`, `niagara_compile.json`), `niagara.inspect`, `niagara.graph.get`, `niagara.validate`, `niagara.add_emitter`, and generic one-operation edit RPCs for properties, parameters, renderers, stack/modules, and graph pins. Removed all public `niagara.authoring.*` handlers and removed `effect.*` Niagara asset-authoring helpers while keeping runtime preview helpers. Added read-only Cascade `cascade.json` dump support. Added/strengthened Niagara, Cascade, edit, absence, and documentation coverage. Static review passed; build/tests/Unreal were not run by developer per project rule.
- `#3-verified-niagara-dumps` `DONE` tester — Verified: `niagara.inspect` on `/Game/UltraDynamicSky/Particles/Puddle_Ripple` returned `assetKind:"NiagaraSystem"` plus system/emitter/parameter data; `asset.dump` for the same Niagara system wrote `niagara_system.json`, `niagara_emitters.json`, `niagara_parameters.json`, `niagara_stack.json`, `niagara_graphs.json`, and `niagara_compile.json`; `asset.dump` for Cascade `/Game/Fx/Smoke` wrote `cascade.json`; `niagara.authoring` is not registered while new generic methods such as `niagara.set_property` and `niagara.add_module` are registered.
