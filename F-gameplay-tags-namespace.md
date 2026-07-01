---
id: F-gameplay-tags-namespace
title: "No project-level gameplay tag authoring namespace"
status: DONE
severity: High
category: feature
tags: [gas, gameplay-tags, authoring, discovery]
---

# No project-level gameplay tag authoring namespace

The plugin has rich GAS asset-authoring coverage (`gas.create_gameplay_ability`,
`gas.set_ability_tags`, `gas.set_effect_tags`, `gas.add_tag_to_asset`, …) but
**no RPC surface for the underlying project-level tag registry itself**. Tags
are only consumable through GAS asset wrappers; they cannot be created,
removed, or listed as a first-class concept. Every non-trivial GAS workflow
eventually needs to write new tag names into the project's tag INI sources,
and today that step has to be done by hand in the editor before any of the
existing `set_*_tags` RPCs can reference the new tag.

The registry slice should expose four INI-backed methods:

```
gameplay_tags.add(tag, source?, comment?)
gameplay_tags.remove(tag, source?)
gameplay_tags.list(prefix?, source?, includeImplicitParents?, limit?, includeTotal?)
gameplay_tags.add_source(source, kind="ini", searchPath?)
```

**Use cases blocked:**

1. Standing up a fresh GAS feature — caller needs to declare
   `Ability.Attack.Melee`, `State.Stunned`, `Cooldown.Dash` in the project
   tag registry before any `gas.set_ability_tags` / `gas.set_effect_tags`
   call can resolve them. Today the agent has to instruct the user to add
   tags in the Project Settings UI, breaking automation.
2. Tag hygiene / audit — listing every tag currently registered in the
   project, grouped by source INI / DataTable, is invisible to the agent.
   No way to detect duplicates, dead tags, or tags only referenced from a
   single GAS asset.
3. Custom tag sources — projects routinely split tags across multiple INI
   files (`GameplayTags.ini`, `DefaultGameplayTags.ini`, plus per-feature
   `<Feature>Tags.ini`) or DataTable sources. Adding a new source is a
   one-time bootstrap step that has no RPC today.

**Current workarounds:**

- Have the user click through Project Settings → GameplayTags → Add New
  Gameplay Tag manually before each automation run.
- `editor.console_command "GameplayTags.DumpTagList"` — output goes to the
  log, not the RPC response, requiring a log-read round-trip; no
  authoring counterpart.
- `python.execute` walking `unreal.GameplayTagsManager` (where the Python
  binding even exists) — bypasses the typed RPC surface entirely.

**Fix:** Add a first-class `gameplay_tags` namespace covering registry CRUD,
source management, and listing. Mutating INI operations should use
`IGameplayTagsEditorModule` where its public API is source-precise
(`AddNewGameplayTagToINI`, single-source `DeleteTagFromINI`, and
`AddNewGameplayTagSource`); multi-source `remove(tag, source)` targets the
requested public `FGameplayTagSource` list because the editor-module delete
method accepts only a tag node. Listing should use `UGameplayTagsManager` and
`GetTagEditorData` so each row returns the tag name, source, dev comment, and
whether the tag is explicit rather than an implicit parent.

```
gameplay_tags.add(
    tag: string,                  // dotted name, e.g. "Ability.Attack.Melee"
    source?: string,              // tag-source filename; default project's primary source
    comment?: string              // dev-comment column in the INI
) -> { tag, source, alreadyExisted: bool }

gameplay_tags.remove(
    tag: string,
    source?: string               // remove only this source entry when the tag exists in multiple sources
) -> { tag, removed: bool }

gameplay_tags.list(
    prefix?: string,              // filter to tags starting with prefix (e.g. "Ability.")
    source?: string,              // filter to one tag-source filename
    includeImplicitParents?: bool, // include implicit "Ability", "Ability.Attack" parents; default true
    limit?: number,               // default 500
    includeTotal?: bool           // default false; true scans all matches for an exact totalMatches
) -> {
    tags: [{
        name: "Ability.Attack.Melee",
        source: "DefaultGameplayTags.ini",
        comment: "...",
        isExplicit: bool          // false for implicit parents
    }, ...],
    totalMatches: number,         // exact only when includeTotal is true or totalMatchesExact is true
    totalMatchesExact: bool,
    truncated: bool
}

gameplay_tags.add_source(
    source: string,               // filename (INI) or asset path (DataTable)
    kind?: "ini",                 // default "ini"
    searchPath?: string           // for INI: relative dir under Config/Tags/
) -> { source, kind, created: bool }
```

Notes:

- `add` and `remove` must respect the **mutating-handler save rules** —
  INI mutation writes the relevant gameplay-tags config source and refreshes
  the tag tree; the handler should not also force a project save in the same
  call.
- `list` should return the **tag source** per row, not just the tag name,
  so callers can audit where a tag came from and target `remove` safely.
  It should apply prefix/source/limit before doing exact count work. With the
  default `includeTotal=false`, the scan stops once `limit` matching rows have
  been collected, `totalMatches` is only the collected lower-bound count, and
  `totalMatchesExact=false`/`truncated=true` tells callers not to treat it as
  an exact project count. Callers that need an exact total must pass
  `includeTotal=true`.
- Recursive `FGameplayTagQuery` expression authoring is split to
  `F-gameplay-tag-query-authoring` because it has a separate validation and
  property-write surface.

**Cross-ref:** Complements the existing `gas.*` asset-authoring surface;
without registry authoring those methods can only reference tags that
already exist in the project.

## History
- `#1-no-tag-authoring-surface` `OPEN` reporter — Verified that no `gameplay_tags.*` namespace exists: grep for `GameplayTagsManager` in `Source/EditorAutomationRpcGateway/Private/Handlers/` hits only `GASHandler.cpp` (asset-tag consumers like `gas.set_ability_tags` / `gas.add_tag_to_asset`); no hits for `AddNewGameplayTagToINI` or `FGameplayTagQuery` anywhere in handlers; `rpc-method-reference.generated.md` shows no `gameplay_tag` / `tags.add` entries; no existing board ticket. Proposes a five-method `gameplay_tags` namespace — `add`, `remove`, `list`, `add_source`, `build_query` — backed by `UGameplayTagsManager` registry APIs. Flagged that `build_query` may split into its own ticket once `add`/`remove`/`list`/`add_source` are implemented; keeping bundled because all five serve a single "declare and use tags from automation" story.
- `#2-registry-handlers-split-query` `IN-REVIEW` developer — Added `gameplay_tags.add`, `gameplay_tags.remove`, `gameplay_tags.list`, and `gameplay_tags.add_source` backed by `IGameplayTagsEditorModule` mutations plus `UGameplayTagsManager::GetTagEditorData` listing metadata, added `FGameplayTagsRegistryRoundTripTest`, and split recursive query composition into `F-gameplay-tag-query-authoring`.
- `#3-source-delete-and-list-page` `IN-REVIEW` developer — Fixed review iteration 1 by making `gameplay_tags.remove(tag, source)` remove only the requested source entry for multi-source tags, changing `gameplay_tags.list` to prefix-filter before sorting and avoid exact count scans unless `includeTotal=true`, and adding regression coverage for multi-source removal plus explicit total listing.
- `#4-bounded-default-list` `IN-REVIEW` developer — Fixed review iteration 2 by making default `gameplay_tags.list` keep at most `limit` row candidates while scanning, sort only the returned candidates deterministically, and report `totalMatchesExact=false` when the limit stops the scan before proving an exact total; exact-count callers can still pass `includeTotal=true`.
- `#5-verify-bounded-list` `DONE` tester — Verified: live gateway wiki exposes `gameplay_tags.add`, `gameplay_tags.remove`, `gameplay_tags.list`, and `gameplay_tags.add_source`; `gameplay_tags.list` with `{"limit":1,"includeTotal":false}` returned one row with `totalMatchesExact=false` and `truncated=true`, while `{"limit":1,"includeTotal":true}` returned `totalMatches=401`, `totalMatchesExact=true`, and `truncated=true`.
