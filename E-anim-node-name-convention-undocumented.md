---
id: E-anim-node-name-convention-undocumented
title: "AnimGraph nodeName convention (asset-derived title substring, no rename verb) is undocumented; agents must read plugin C++ to learn how to name/resolve player nodes"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, animation, anim-graph, node-resolution, add_graph_node, bind_player_asset, set_sync_group]
---

# AnimGraph `nodeName` convention is undocumented — agents reverse-engineer it from C++

A task that asks for AnimGraph player nodes with *chosen* names (e.g.
"add two sequence-player nodes named `IdlePlayer` and `WalkPlayer`") collides
with how the MCP actually identifies AnimGraph nodes, and the wiki gives the
agent no way to discover the right strategy at the point of decision.

The reality the agent has to learn the hard way:

1. **There is no rename verb for AnimGraph nodes.** Unlike `widget.rename_widget`
   or graph-node naming on other surfaces, no `animation.authoring` method sets
   a node title. A `UAnimGraphNode_SequencePlayer`'s title is *derived* from its
   bound asset (`Sequence Player '<AssetName>'`), so you cannot give it an
   arbitrary name like `IdlePlayer`.
2. **`nodeName` on every AnimGraph mutator (`bind_player_asset`,
   `set_sync_group`, ...) is a `GetNodeTitle(ListView)` substring match**, not a
   stable identifier. To make a node resolvable you must bind a distinctly-named
   asset *at creation* (`add_graph_node` with `bindAsset`) and then resolve it by
   a substring unique to that asset title (e.g. `"Dino_Idle"`), NOT by the
   requested logical name and NOT by the shared `"Sequence Player"` prefix (two
   unbound players are both titled exactly `Sequence Player` and are
   indistinguishable).

None of this is in the wiki. `wiki-src/animation.authoring.md` mentions the
"`nodeName` convention" **twice** — `set_sync_group` "resolving the node with
the same `nodeName` convention as `bind_player_asset`" (line 114) and the
`bind_player_asset` section itself (line 102) — but **never states what the
convention is**: not that resolution is title-substring, not that a player's
title is asset-derived, not that there is no rename, not the
"bind-a-named-asset-at-creation-then-resolve-by-asset-substring" strategy. The
`add_graph_node` material doesn't steer node naming either. So an agent that
wants to do this correctly has to read plugin source to find out.

This is a discovery/process gap, not a tool bug — every call in the observed
task succeeded first try. It is the docs companion to
`E-anim-node-name-substring-ambiguous` (the judge-filed ticket about the
resolver itself *silently* picking a node on multi-match): that ticket fixes the
**code** (ambiguity error / echo the resolved title); this ticket fixes the
**docs** so an agent knows the convention and node-naming strategy *before*
hitting the ambiguity. It is the AnimGraph analogue of
`E-widget-add-then-rename-discoverability` (name-at-creation vs. add-then-rename,
wiki doesn't steer the from-scratch-with-known-names case).

**Workaround:** Bind a distinctly-named asset to each player node at
`add_graph_node` time and resolve every later `nodeName` by a substring unique
to that asset (the bound asset's name), never by the requested logical node name
or the shared `"Sequence Player"` prefix.

**Fix (downstream, wiki only):** In `docs/wiki-src/animation.authoring.md`, where
the "`nodeName` convention" is referenced (near `bind_player_asset` /
`set_sync_group`, and at `add_graph_node`), add a short "naming &
resolving AnimGraph nodes" note that states the convention explicitly:
(1) `nodeName` is matched as a substring of the node's list-view title — not a
stable id; (2) a player node's title is **derived from its bound asset**, and
there is **no rename verb**, so you cannot give it an arbitrary name;
(3) to make a node uniquely resolvable, bind a distinctly-named asset at
`add_graph_node` time and resolve it later by a substring unique to that asset
(its asset name), not by a logical name or the shared `"Sequence Player"`
prefix; (4) cross-link `E-anim-node-name-substring-ambiguous` for the multi-match
hazard.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an
  `animation.authoring.set_sync_group` task (build `ABP_DinoLocomotion`: two
  SequencePlayer nodes the prompt asked to name `IdlePlayer`/`WalkPlayer`, shared
  `Locomotion` sync group). Every call succeeded (28 calls, no retries / no
  python fallback, "Full success check passed"), but the friction note records
  the discovery cost: "the MCP has no way to name AnimGraph nodes
  `IdlePlayer`/`WalkPlayer` or set a node title — node resolution
  (set_sync_group/bind_player_asset nodeName) is pure GetNodeTitle(ListView)
  substring matching with no rename verb ... I confirmed this gap by reading the
  plugin C++ (AnimGraphConstructionUtils FindAnimGraphNodeByTitleSubstring +
  engine GetNodeTitleHelper), since the wiki only vaguely referenced a 'nodeName
  convention' — that's a discoverability gap." So the agent had to bind assets at
  creation and resolve by the asset-name substrings `Dino_Idle`/`Dino_Walk`
  instead of the requested names. Distinct from the judge-filed
  `E-anim-node-name-substring-ambiguous` (that ticket = the resolver's silent
  multi-match *code* bug; this ticket = the undocumented convention + node-naming
  *strategy* that forced source-reading). Wiki overlay to fix:
  `docs/wiki-src/animation.authoring.md` (the two "`nodeName` convention"
  references at lines ~102/114 and the `add_graph_node` material).
- `#2-docs-fix` `IN-REVIEW` developer — Wiki-only fix. Added a
  `## Naming and resolving AnimGraph nodes (the nodeName convention)` namespace
  section to `Docs/wiki-src/animation.authoring.md` that states the convention
  explicitly: (1) `nodeName` is a list-view-title match via the **current**
  resolver `ResolveAnimGraphNodeByTitle` (exact pass, then substring pass) — not a
  stable id; (2) a player node's title is derived from its bound asset and there is
  **no rename verb**, so it can't take an arbitrary logical name; (3) the
  bind-distinctly-named-asset-at-`add_graph_node`-then-resolve-by-asset-substring
  strategy (never the shared `Sequence Player` prefix); (4) cross-links
  `E-anim-node-name-substring-ambiguous` and describes the **current** post-fix
  behavior — a multi-match now returns `AMBIGUOUS_NODE` (not a silent pick) and
  mutators echo the resolved title. The section is in the overlay prelude (before
  the first `### `), so it renders on the `animation.authoring` namespace page. Also
  added steer lines on the `### bind_player_asset`, `### set_sync_group`, and
  `### add_graph_node property writes` overlay sections pointing at the new section
  (the two pre-existing "`nodeName` convention" references now resolve to a real
  definition). Used the live resolver name, not the stale
  `FindAnimGraphNodeByTitleSubstring` cited in `#1`. Regression test:
  `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestAnimGraphNodeNamingDocs.cpp`
  (`FAnimGraphNodeNamingNamespaceDocTest` + `FAnimBindPlayerAssetNamingDocTest`)
  renders through production `WikiHandler::RenderPage` and asserts the
  overlay-exclusive markers (ResolveAnimGraphNodeByTitle, "no rename verb",
  "derived from its bound asset", bindAsset strategy, AMBIGUOUS_NODE), so reverting
  the overlay fails the suite. File changed:
  `Docs/wiki-src/animation.authoring.md`.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
