---
id: E-gas-set-effect-tags-drops-unregistered
title: "gas set_effect_tags / set_ability_tags silently drop unregistered tags (ok:true, no signal)"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [gas, set_effect_tags, set_ability_tags, gameplay-tags, silent-noop]
---

# GAS tag-write verbs report a clean success while dropping unregistered tags

`gas.set_effect_tags` accepts a `grantedTags` array and reports which tags it
applied via `tagsAdded`. When a requested tag is **not yet registered** in the
project tag registry, the call still returns `ok:true` / `is_error:false`, but
the tag is silently omitted: `tagsAdded:[]`. There is no error, no warning, and
no field indicating that a requested tag was rejected for being unregistered —
the response is indistinguishable from a no-op success.

The same silent-drop is a **shared root cause across the GAS tag-write family**,
not unique to `set_effect_tags`. Every site resolves tags with
`FGameplayTag::RequestGameplayTag(FName(*TagStr), /*ErrorIfNotFound=*/false)`
(the `GetOrRequestTag` helper) and then gates the mutation on `if (Tag.IsValid())`
with no else-branch, so unregistered tags evaporate. The sibling
`gas.set_ability_tags` is worse: its `cancelAbilitiesWithTags` and
`blockAbilitiesWithTags` containers are not even reflected in `tagsAdded`, so an
unregistered cancel/block tag is invisible even to the
"diff input against tagsAdded" heuristic.

The caller's only signal that something went wrong is to notice that the input
array had N tags but `tagsAdded` came back shorter (here, empty) — and for the
cancel/block containers there is no signal at all. To recover, the tag must
first be declared via `gameplay_tags.add` (now available), after which the
identical call succeeds. Nothing in the first response points at "register the
tag first."

This is a misleading-success ergonomic gap: a write RPC that drops part of its
input must not report unqualified success.

**Fix:** Adopt the established validate-before-mutate convention already landed
for the identical defect class in `ai.configure_slot_behavior`
(`B-configure-slot-behavior-ignores-behavior-and-tags`): pre-resolve every
requested tag against the registry, collect the unresolved ones into a
`droppedTags` list, and if any are dropped **reject the whole call with
`INVALID_PARAMS`** carrying a `droppedTags` array and a message steering the
caller at `gameplay_tags.add` — no partial write. Apply this across the GAS
tag-write family: `gas.set_effect_tags` (`grantedTags`) and `gas.set_ability_tags`
(`abilityTags` + `cancelAbilitiesWithTags` + `blockAbilitiesWithTags`). On the
all-valid path, behavior is unchanged (tags applied, `tagsAdded` returned). The
single-tag cue/asset sites (`add_tag_to_asset` at `INVALID_TAG`, the cue-tag
assignments) are out of scope: `add_tag_to_asset` already rejects unregistered
tags, and the cue assignments are not array-shaped grants.

## Evidence (this task — focus `gas.add_effect_execution_calculation`)

Friction note (verbatim): "set_effect_tags silently returns tagsAdded:[] for
an unregistered tag (no error); had to register via gameplay_tags.add first."

Call-log shows the exact two-step recovery:

- `gas.set_effect_tags {grantedTags:["Effect.Damage.Fire"]}` →
  `tagsAdded:[]` — `ok:true, is_error:false` (tag unregistered, silently dropped)
- `gameplay_tags.add {tag:"Effect.Damage.Fire"}` → ok
- `gas.set_effect_tags {grantedTags:["Effect.Damage.Fire"]}` →
  `tagsAdded:["Effect.Damage.Fire"]` — ok

Cost: one wasted `set_effect_tags` whose clean success hid the drop, plus a
register round-trip the response gave no hint was needed.

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of the `gas.add_effect_execution_calculation` task. `gas.set_effect_tags` returned `ok:true` with `tagsAdded:[]` when given an unregistered tag `Effect.Damage.Fire`, silently dropping it with no error/warning/skipped field; the author only recovered by registering via `gameplay_tags.add` and re-running (then `tagsAdded:["Effect.Damage.Fire"]`). Misleading-success: a partial-write RPC reports unqualified success. Distinct from `F-gameplay-tags-namespace` (which added the *registry* surface, not error reporting on the GAS consumer) and from the `E-gas-info-skips-asc-owner-actor` readback gap. Proposes erroring (`TAG_NOT_REGISTERED`) or surfacing a `tagsSkipped`/`unregisteredTags` field, or optional auto-register, so the drop is visible.
- `#2-reword` `OPEN` developer — Reworded to match reality and the established fix convention. (a) Scope widened from `set_effect_tags` alone to the **GAS tag-write family**: source review confirms the identical `GetOrRequestTag` + `if (Tag.IsValid())` silent-drop also lives in `gas.set_ability_tags` (`abilityTags` at GASHandler.cpp:671, plus `cancelAbilitiesWithTags`:695 / `blockAbilitiesWithTags`:708 which don't even populate `tagsAdded`, so they drop invisibly). (b) Replaced the three divergent proposed remedies (new `TAG_NOT_REGISTERED` code / `tagsSkipped` field / auto-register) with the single `INVALID_PARAMS` + `droppedTags` validate-before-mutate convention already shipped for the sibling `ai.configure_slot_behavior` (`B-configure-slot-behavior-ignores-behavior-and-tags`, AIHandler.cpp:1879-1916), for cross-handler consistency. The single-tag cue/`add_tag_to_asset` sites are explicitly out of scope (the latter already rejects with `INVALID_TAG`).
- `#3-evidence-now-rejects-but-wiki-silent` `IN-REVIEW` reporter — Cross-task evidence + a distinct **docs** angle. A later REALISM task (action-RPG GAS setup: PlayerAttributeSet + GE_Burning/GE_Heal under /Game/RPG/GAS) confirms the shipped validate-before-mutate fix is now live: `gas.set_effect_tags {grantedTags:["Status.Burning"]}` on a fresh project returned `[INVALID_PARAMS] 1 granted tag(s) are not registered and would be dropped; register them first (e.g. via gameplay_tags.add) then retry. See droppedTags.` — the runtime now rejects+steers correctly (no more silent `tagsAdded:[]`), so the core ergonomic defect is resolved. **Remaining process friction is purely discoverability**: the prereq is still not flagged anywhere on the `gas.set_effect_tags` wiki page, so callers only learn the register-first requirement by hitting the error mid-task. Author friction note (verbatim): "gas.set_effect_tags rejects unregistered tags (had to gameplay_tags.add Status.Burning first) — sensible but not flagged on the gas.set_effect_tags wiki page." Recovery cost: one wasted `set_effect_tags`, one `gameplay_tags.add`, one retry (3 calls for a 1-call intent). Downstream wiki fix (not mine to apply): in `docs/wiki-src/gas.md`, on the `gas.set_effect_tags` (and `gas.set_ability_tags`) section, add a one-line prerequisite note — granted/ability/cancel/block tags must already be registered via `gameplay_tags.add`, else the whole call is rejected with `INVALID_PARAMS` + `droppedTags`.
