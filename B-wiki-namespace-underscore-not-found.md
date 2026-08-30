---
id: B-wiki-namespace-underscore-not-found
title: "Wiki namespace page returns Not Found for underscore-named namespaces (game_framework, gameplay_tags, ...)"
status: IN-REVIEW
severity: High
category: bug
tags: [wiki, discovery, render-page, normalize-query]
---

# Wiki namespace page returns Not Found for underscore-named namespaces

Navigating to a **registered, documented** single-token namespace whose name
contains an underscore returns a `# Not found:` wiki page instead of the
namespace index — and the "Did you mean" suggestion circularly lists the very
namespace that was requested. The namespace's methods are fully registered and
dispatch correctly; only the namespace **index page** is broken.

Observed live against `mcp__editor-automation__call`:

```
call(method="game_framework")     →  # Not found: `game_framework`
                                       Did you mean:
                                       - `game_framework`        ← circular
                                       - `gameplay_tags`
                                       - `geometry.create_torus`
                                       - `geometry.mirror`
                                       - `geometry.create_cone`

call(method="gameplay_tags")      →  # Not found: `gameplay_tags`
                                       Did you mean:
                                       - `gameplay_tags`         ← circular
                                       - `gameplay_tags.add`     ← its own methods exist!
                                       - `gameplay_tags.list`
                                       - `gameplay_tags.remove`
                                       - `geometry.taper`
```

By contrast, dotless namespaces **without** an underscore render correctly and
return the on-disk page reference:

```
call(method="gas")        →  {"page": ".../wiki/gas.md", ...}
call(method="ai")         →  {"page": ".../wiki/ai.md", ...}
call(method="networking") →  {"page": ".../wiki/networking.md", ...}
```

This is not a stale-disk artifact: the `WikiDiskGenerator` run at editor launch
writes every per-method page correctly (`game_framework.configure_game_rules.md`,
`gameplay_tags.add.md`, etc. all exist and are valid, same mtime as the rest of
the tree), but the namespace **index** files `game_framework.md` /
`gameplay_tags.md` contain the broken "Not found" body — because the disk
generator renders them through the same `WikiHandler::RenderPage` that fails at
runtime. The live `call()` likewise returns the inline not-found fallback for
these namespaces rather than a `{page}` reference.

The defect affects a whole class of namespaces, not just two. A qmd sweep of the
committed `wiki-generated/` tree shows the same broken namespace index for every
underscore-named single-token namespace:

```
Not found: `pose_search`
Not found: `world_partition`
Not found: `post_process`
Not found: `game_framework`
Not found: `_test`
```

## Root cause

`Catalog/WikiHandler.cpp`. `RenderPage` (line ~483) runs every path through
`NormalizeQuery` before `ClassifyNode`:

```cpp
const FString Normalized = NormalizeQuery(OriginalPath);
const EWikiNodeKind Kind = ClassifyNode(Normalized, Cache);
```

`NormalizeQuery` (lines ~247-258) collapses legacy flattened MCP tool names
(`actor_spawn_from_blueprint` → `actor.spawn_from_blueprint`) by converting the
**first** underscore to a dot whenever the input has no dot:

```cpp
FString NormalizeQuery(const FString& In)
{
    FString Out = In.ToLower();
    if (!Out.Contains(TEXT(".")))
    {
        int32 FirstUnderscore = INDEX_NONE;
        if (Out.FindChar(TEXT('_'), FirstUnderscore))
        {
            Out = Out.Left(FirstUnderscore) + TEXT(".") + Out.Mid(FirstUnderscore + 1);
        }
    }
    return Out;
}
```

For a real underscore-containing namespace name with no method part this mangles
the input: `"game_framework"` → `"game.framework"`, `"gameplay_tags"` →
`"gameplay.tags"`, `"world_partition"` → `"world.partition"`. `ClassifyNode`
then finds no category/method/topic named `game.framework`, returns `NotFound`,
and `RankSuggestions` fuzzy-matches the mangled string back to the real
`game_framework` (substring/Levenshtein), producing the circular "Did you mean".

The heuristic cannot distinguish "legacy flattened tool name" from "real
namespace whose name legitimately contains an underscore" because both are
dotless tokens with an underscore.

## Impact

Namespace discovery — the documented `call("<namespace>")` doc-fetch convention
— is broken for an entire class of namespaces. An agent that follows the
intended discovery flow (drill into a namespace to enumerate its methods) hits a
dead-end "Not found" whose only suggestion is the thing it already asked for.
The methods still dispatch, so the impact is on discoverability/onboarding, not
execution; agents must already know the exact `namespace.method` name (or read
the disk wiki directly) to proceed.

**Workaround:** read the per-method wiki pages on disk directly (they generate
correctly under their real `namespace.method` names), or address methods by
their full dotted name without ever fetching the namespace index.

**Fix:** make `NormalizeQuery` (or its caller) underscore-collapse only as a
**fallback** — classify the original input first and only attempt the
underscore→dot rewrite if the verbatim input does not resolve to a known
category/method/topic node. Equivalently, skip the rewrite when
`Cache.CategoryNodes`/`AllNodes` already contains the input verbatim. That
preserves the legacy `actor_spawn` collapse while leaving real underscore-named
namespaces intact.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live against `mcp__editor-automation__call`: `call("game_framework")` and `call("gameplay_tags")` both return `# Not found:` with a circular self-suggestion, while `call("gas")`/`call("ai")`/`call("networking")` return the normal `{page}` reference. The two failing namespaces are fully registered (every `game_framework.*` / `gameplay_tags.*` method dispatches, and `gameplay_tags`'s own methods appear in its own did-you-mean list), and all per-method wiki pages generate correctly on disk at the same launch — only the namespace index page is broken. Root-caused to `NormalizeQuery` in `Catalog/WikiHandler.cpp` (lines ~247-258) unconditionally rewriting the first underscore to a dot for dotless inputs, so `game_framework`→`game.framework` misses `ClassifyNode`. Confirmed broad blast radius via a qmd sweep of committed `wiki-generated/` (`pose_search`, `world_partition`, `post_process`, `game_framework`, `_test` all render "Not found"). Proposed fix: only apply the underscore→dot collapse as a fallback when the verbatim input does not resolve to a registered node.
- `#2-evidence-world-partition-live` `OPEN` reporter — Cross-task evidence (struggle-audit of a `performance.configure_world_partition` open-world streaming task). The agent's discovery flow drilled `(root index)` then `performance` (both rendered) then `world_partition` to enumerate the namespace's data-layer/cell methods, and the namespace index returned the inline `Not found: world_partition` body (error_text recorded verbatim in the call log as "Not found: world_partition — index page not generated"). Confirms the underscore-namespace blast radius this ticket already predicts is hit live, not just in the committed `wiki-generated/` sweep: `world_partition` is a real, fully-registered namespace (`world_partition.create_datalayer`, `world_partition.load_cells` both dispatched and succeeded in the same task) whose index is the only broken surface. Agent recovered by addressing the methods by full dotted name. Same root cause (`NormalizeQuery` first-underscore-to-dot in `Catalog/WikiHandler.cpp`); no new fix needed — appended as live confirmation.
- `#3-fix-verbatim-first-classification` `IN-REVIEW` developer — Implemented the fallback-first fix. Added `ResolveCanonicalLower(In, Cache)` in `Private/Catalog/WikiHandler.cpp`: it classifies the verbatim lowercased input first and applies the legacy underscore->dot collapse (`NormalizeQuery`) only when the verbatim form does not resolve to a registered node. `RenderPage` now resolves via `ResolveCanonicalLower` instead of the unconditional `NormalizeQuery`, so `game_framework`/`gameplay_tags`/`world_partition`/`post_process`/`pose_search` classify verbatim as `LeafNamespace` and render the method index, while a legacy flat name like `actor_spawn` (which does not resolve verbatim) still collapses to `actor.spawn`. To keep the live `call()` disk-path lookup consistent with the disk generator (which writes `game_framework.md` from `EnumerateAllSlugs`'s verbatim slugs), `NormalizeSlug` is now cache-aware: added a `NormalizeSlug(const FRpcDispatcher&, const FString&)` overload routing through `ResolveCanonicalLower`, and the parameterless overload resolves against the live subsystem dispatcher (pure-`NormalizeQuery` fallback when no dispatcher). Files: `Source/EditorAutomationRpcGateway/Private/Catalog/WikiHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Catalog/WikiHandler.h`. Tests (in `Private/Tests/Infra/TestWikiHandler.cpp`, exercising production `WikiHandler::RenderPage`/`NormalizeSlug`): `Namespace.UnderscoreResolves` asserts `RenderPage("gameplay_tags")` is not a `# Not found:` page and contains `## Methods`; `NormalizeSlug.UnderscoreNamespacePreserved` asserts `NormalizeSlug("gameplay_tags") == "gameplay_tags"` (preserved verbatim) while an unresolved flat name still collapses its first underscore. Both fail if the verbatim-first change is reverted. Not compiled/tested here — left for the test phase.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
