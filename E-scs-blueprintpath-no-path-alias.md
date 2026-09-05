---
id: E-scs-blueprintpath-no-path-alias
title: "blueprint.scs.* require 'blueprintPath' with no 'path'/'assetPath' alias — missed by the canonical blueprint path-alias fix"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, scs, param-alias, blueprintpath, path, drift]
encounters: 3
lastSeen: 2026-09-05T00:00:00Z
---

# `blueprint.scs.*` require `blueprintPath` and reject the `path` spelling, outside the canonical blueprint alias fix

Sibling of the path/assetPath/blueprintPath drift family: the canonical
`E-blueprint-param-name-path-vs-assetpath` (DONE) wired dispatcher `FParamSpec`
aliases onto *resolver-backed* Blueprint path specs so `path`/`assetPath`/
`blueprintPath` are interchangeable. That fix migrated an enumerated set of
handler files; `SCSHandler.cpp` was in neither cluster, so the `blueprint.scs.*`
verbs were never aliased and still bind their slot directly.

`blueprint.scs.get` declares `RPC_PARAM_REQ("blueprintPath", ...)`
(SCSHandler.cpp:50) and reads via `Ctx.GetString(TEXT("blueprintPath"))` (:57) —
it does **not** route through `ResolveBlueprintPath`, so it carries no `path`/
`assetPath` alias. `blueprint.scs.add_component` (:82/:90) and
`blueprint.scs.remove_component` (:115/:119) follow the same direct-read pattern.
So an agent that just used `path`/`assetPath` on an aliased `blueprint.*` verb,
then probes the SCS tree, gets a hard `[MISSING_REQUIRED_PARAM] Missing required
parameter 'blueprintPath'` before the handler runs. CLAUDE.md's camelCase/
snake_case alias rule does not cover this — `path` and `blueprintPath` are
distinct names, not casing variants.

**Fix:** Apply the dispatcher `FParamSpec` alias from
`E-blueprint-param-name-path-vs-assetpath #4` to the `blueprint.scs.*`
`blueprintPath` specs (accept `path`/`assetPath`) and read body-side via the
multi-key getter, so the namespace stays uniformly aliased as new verbs land.

## History
- `#1-scs-get-rejects-path` `OPEN` reporter — Novel half of the rejected umbrella
  proposal `E-path-param-key-inconsistency` (cross-namespace restatement of the
  drift family, declined as a duplicate). Replay: `blueprint.scs.get
  {path:"/App/.../B_GateBase"}` → `[MISSING_REQUIRED_PARAM] Missing required
  parameter 'blueprintPath'`; retry with `{blueprintPath:...}` → success. One
  round-trip, accurate error, zero blocked progress. Source: SCSHandler.cpp:50
  (`RPC_PARAM_REQ("blueprintPath", ...)`), body :57 (`Ctx.GetString("blueprintPath")`),
  no `path` alias; add/remove verbs (:82/:115) read `blueprintPath` directly the
  same way. Not covered by the DONE `E-blueprint-param-name-path-vs-assetpath`,
  whose `#2`/`#4` migration list excludes `SCSHandler.cpp`. Severity Low (pure
  friction, self-correcting, matching the rest of the drift family).
- `#2-additional-assetpath-spelling` `OPEN` reporter — Re-observed on the
  `blueprint.scs.reparent_component` BP_SecuritySpotlight build (Actor BP: Pole
  root → LampHead → Spotlight reparented under LampHead + TriggerVolume; compile
  + save + read-back; outcome `clean`). Concrete evidence for the `assetPath`
  spelling the title names (the `#1` replay used `path`): the agent's first probe
  `blueprint.scs.get {assetPath:"/Game/BP_SecuritySpotlight"}` →
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'blueprintPath' (type:
  string)`; after one wiki-read of `blueprint.scs.get.md` it reissued with
  `{blueprintPath:...}` and succeeded. The wrong guess has clear cross-namespace
  provenance — the SAME run's `asset.save` keys on `assetPath`, so an agent
  fresh off an `asset.*` verb naturally reuses `assetPath` on `blueprint.scs.*`.
  One round-trip + one wiki-read, self-corrected, zero blocked progress —
  reconfirms the Low severity and the FParamSpec-alias fix. (CallAnalyzer flagged
  as `frustrating`/`guessed param format`, severity trivial.)
- `#3-siblings-accepted-assetpath-same-session` `OPEN` WEAPONS-critic — Third encounter, and the sharpest evidence yet for the "inconsistent alias sets inside one namespace" framing rather than "an agent guessed a param name". Measured during a WEAPONS critic review round 3 on `/Game/FPS/Weapons/BP/BP_Weapon_AR`: `blueprint.scs.get {assetPath:…}` → `[MISSING_REQUIRED_PARAM] Missing required parameter 'blueprintPath'`, while **three sibling verbs accepted `assetPath` in the same session on the same asset** — `blueprint.decompile`, `blueprint.graph.find_orphaned_nodes` and `asset.dump`. `#2` established the cross-namespace provenance of the wrong guess (an agent arriving from `asset.save`); this adds the within-`blueprint.*` case, which is stronger: the agent was not carrying a habit over from another namespace, it had just used `assetPath` successfully on `blueprint.decompile` and `blueprint.graph.*` one call earlier. Cost was again one round-trip and zero blocked progress, so **severity stays Low** — but the reason to fix it is no longer only friction: `blueprint.scs.get` is now the odd one out inside a namespace whose other members have all been aliased, which is exactly the drift the `FParamSpec` fix in `E-blueprint-param-name-path-vs-assetpath #4` was meant to end. Filed alongside a separate readback defect on the same verb the same round — `B-scs-get-inherited-override-rows-no-parent` (inherited-override rows carry no `parent`, and `WeaponRoot` reports `child_count: 1` with two children under it) — which is a different code path; a fixer opening `SCSHandler.cpp` for the alias should read that ticket too. No plugin source was opened for this entry.
