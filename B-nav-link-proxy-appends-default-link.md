---
id: B-nav-link-proxy-appends-default-link
title: "create_nav_link_proxy appends the requested link as PointLinks[1], leaving the engine-default link at PointLinks[0] — so the proxy carries two links and configure_nav_link / smart-link conversion operate on the wrong (default) entry"
status: IN-REVIEW
severity: High
category: bug
tags: [navigation, create_nav_link_proxy, configure_nav_link, set_nav_link_type, pointlinks, duplicate-link, index-mismatch, navlinkproxy]
---

# `navigation.create_nav_link_proxy` adds a SECOND link instead of replacing the constructor's default link, and `configure_nav_link` then edits the wrong entry

`ANavLinkProxy`'s constructor already seeds `PointLinks` with one default
`FNavigationLink` (`C:\UE_5.7\Engine\Source\Runtime\AIModule\Private\Navigation\NavLinkProxy.cpp`,
lines 103-106: `FNavigationLink DefLink; ... PointLinks.Add(DefLink);`). That
default link is `Left {0,-50,0}`, `Right {0,50,0}`, `SnapRadius 30`,
`AreaClass NavArea_Default` and lives at `PointLinks[0]`.

`navigation.create_nav_link_proxy` builds the caller's link and then does
`NavLink->PointLinks.Add(NewLink)`
(`NavigationHandler.cpp:531`). Because it **appends** rather than replacing
`PointLinks[0]`, every proxy spawned through this RPC ends up with **two**
links:
- `PointLinks[0]` = the engine default `{0,-50,0}→{0,50,0}` (a stray link the
  caller never asked for, at a bogus location), and
- `PointLinks[1]` = the link the caller actually requested.

This is a silent-wrong-state bug: the call returns `ok` with `existsAfter:true`,
but the actor's nav-link content is not what was requested — it carries an extra
phantom link plus the intended one.

It compounds with the sibling RPCs because they address `PointLinks[0]`:

- `navigation.configure_nav_link` edits `PointLinks[0]`
  (`NavigationHandler.cpp:571-583`: `FNavigationLink& Link = NavLink->PointLinks[0];`).
  So a caller who created a proxy and then "fine-tunes the endpoints / snap
  radius" is editing the **stray default link**, NOT the link they created. The
  caller's real link (`PointLinks[1]`) is left with its original values.
- Smart-link conversion reads `PointLinks[0]` too: `ANavLinkProxy` copies
  `PointLinks[0].Left/.Right/.Direction` into the smart link component
  (`NavLinkProxy.cpp:376`: `SmartLinkComp->SetLinkData(PointLinks[0].Left, PointLinks[0].Right, PointLinks[0].Direction)`),
  so the smart link is driven by the index-0 entry that `create_nav_link_proxy`
  did NOT populate with the caller's geometry (it populated index 1).

Net effect for the natural "create → configure → set type smart → configure
behavior" flow: the proxy ends up with two duplicate-geometry links holding
**conflicting** `SnapRadius` (50 vs 30) and `AreaClass` (Default vs None), and
the entry the user explicitly created via `create_nav_link_proxy` is the one
that `configure_nav_link` never touches. The attempt agent noticed the residue
("after conversion PointLinks holds two entries … harmless but slightly
confusing") but it is not merely cosmetic — the two RPCs disagree on which array
index is the live link.

**Repro (verbatim, replayed through `mcp__editor-automation__call`):**

1. `navigation.create_nav_link_proxy`
   `{actorName:"ReplayLink_A", location:{x:600,y:0,z:0}, startPoint:{x:600,y:0,z:300}, endPoint:{x:900,y:0,z:0}, direction:"BothWays"}`
   → `ok` (`existsAfter:true, actorClass:"NavLinkProxy"`).
   `actor.describe ReplayLink_A` → `PointLinks` has **2 entries**:
   - entry[0]: `Left [0,-50,0]  Right [0,50,0]  SnapRadius 30  Direction BothWays  AreaClass /Script/NavigationSystem.NavArea_Default`  ← stray engine default, never requested
   - entry[1]: `Left [600,0,300] Right [900,0,0] SnapRadius 30  Direction BothWays  AreaClass None`  ← the link the caller asked for

2. `navigation.configure_nav_link`
   `{actorName:"ReplayLink_A", startPoint:{x:600,y:0,z:300}, endPoint:{x:900,y:0,z:0}, direction:"BothWays", snapRadius:50}`
   → `ok` (`modified:true`).
   `actor.describe ReplayLink_A` → `PointLinks` still **2 entries**:
   - entry[0]: `Left [600,0,300] Right [900,0,0] SnapRadius 50  AreaClass /Script/NavigationSystem.NavArea_Default`  ← the stray default got the snapRadius 50 edit
   - entry[1]: `Left [600,0,300] Right [900,0,0] SnapRadius 30  AreaClass None`  ← the caller's real link is UNTOUCHED (still SnapRadius 30)

So `configure_nav_link`'s `snapRadius:50` landed on the entry the caller did not
create, while the caller's own link kept `SnapRadius 30`. The two links now have
identical geometry but divergent snap radius and area class.

**Workaround:** none from the RPC surface — there is no verb to delete a
PointLinks entry or to target an index, so the duplicate cannot be cleaned up
and the index mismatch cannot be sidestepped. (`set_nav_link_type` is innocent
here; it only flips `bSmartLinkIsRelevant` and enables the smart comp.)

**Fix:** in `create_nav_link_proxy`, **replace** the constructor default rather
than append — e.g. if `PointLinks.Num() > 0` set `PointLinks[0] = NewLink`,
otherwise `PointLinks.Add(NewLink)` (mirroring the `Num()==0` guard
`configure_nav_link` already has at `NavigationHandler.cpp:571`). That makes the
caller's link the single `PointLinks[0]`, so `configure_nav_link` (which edits
`[0]`) and the smart-link conversion (which reads `[0]`) all operate on the same
link the caller created, and no phantom default link is left behind.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed via `mcp__editor-automation__call` on a NavLinkProxy "smart traversal link" task (seed `navigation.set_nav_link_type`, SEED mode). Root cause is in `navigation.create_nav_link_proxy` (the culprit, not the seed): it does `PointLinks.Add(NewLink)` (`NavigationHandler.cpp:531`) on top of the constructor's pre-seeded default link (`ANavLinkProxy::ANavLinkProxy` → `PointLinks.Add(DefLink)` at engine `NavLinkProxy.cpp:103-106`), so every spawned proxy has TWO PointLinks: a stray default at index 0 and the requested link at index 1. `configure_nav_link` edits `PointLinks[0]` (`NavigationHandler.cpp:571-583`) and smart-link conversion reads `PointLinks[0]` (engine `NavLinkProxy.cpp:376`), i.e. both target the stray default, NOT the caller's index-1 link. Verbatim replay: after `create_nav_link_proxy {startPoint{600,0,300}, endPoint{900,0,0}}`, `actor.describe` showed entry0 `Left[0,-50,0] Right[0,50,0] SnapRadius30 AreaClass NavArea_Default` and entry1 `Left[600,0,300] Right[900,0,0] SnapRadius30 AreaClass None`; after `configure_nav_link {snapRadius:50}`, entry0 became `Left[600,0,300] Right[900,0,0] SnapRadius50 AreaClass NavArea_Default` while entry1 stayed `SnapRadius30 AreaClass None` — the caller's own link is never touched and a phantom duplicate persists. Silent success with wrong state; no RPC verb exists to remove a PointLinks entry or target an index, so no workaround. Deduped against all nav/link board tickets (B-level-build-navigation-no-completion-signal, B-navigation-rebuild-navigation-no-completion-signal, E-nav-modifier-create-no-areaclass-echo, E-navmesh-config-requires-bounds-volume-prereq) — none covers the duplicate-default-link / index-mismatch. Proposed fix: replace `PointLinks[0]` instead of appending in `create_nav_link_proxy`.
- `#2-fix` `IN-REVIEW` developer — Fixed in `Source/EditorAutomationRpcGateway/Private/Handlers/AI/NavigationHandler.cpp` (`navigation.create_nav_link_proxy`): replaced `NavLink->PointLinks.Add(NewLink)` with a guard that sets `PointLinks[0] = NewLink` when `Num() > 0` (else `Add`), mirroring the `Num()==0` guard `configure_nav_link` already uses. The constructor-seeded default at index 0 is now overwritten with the caller's link, so the proxy carries a single `PointLinks[0]` holding the requested geometry, and `configure_nav_link` (edits `[0]`) plus the engine simple→smart copy (reads `[0]`) all operate on the caller's link — no phantom duplicate. Added behavioral regression `FNavCreateNavLinkProxyReplacesDefaultLinkTest` (`EditorAutomationRpcGateway.navigation.create_nav_link_proxy.ReplacesDefaultLink`) in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAIHandlers.cpp`: it spawns a proxy via the RPC, reads the live `ANavLinkProxy` back from the editor world, asserts `PointLinks.Num()==1` with `PointLinks[0]` holding the caller's start/end geometry (not two entries with a stray default), then calls `configure_nav_link {snapRadius:50}` and asserts the radius landed on `PointLinks[0]` with the count still 1. The test fails if the append is restored (proxy would carry 2 links and snapRadius:50 would land on the phantom default while the caller's link kept SnapRadius 30).
