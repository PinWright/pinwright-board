---
id: E-scs-blueprintpath-no-path-alias
title: "blueprint.scs.* require 'blueprintPath' with no 'path'/'assetPath' alias — missed by the canonical blueprint path-alias fix"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, scs, param-alias, blueprintpath, path, drift]
encounters: 2
lastSeen: 2026-07-02T08:30:24.2495380+03:00
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
