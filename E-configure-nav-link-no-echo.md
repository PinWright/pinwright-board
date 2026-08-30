---
id: E-configure-nav-link-no-echo
title: "navigation.configure_nav_link doesn't echo the snapRadius/endpoints it applied; confirming the re-tune forces an actor.describe readback (which then spills)"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [navigation, configure_nav_link, readback, echo, navlink, docs]
---

# `navigation.configure_nav_link` doesn't echo the applied snapRadius / endpoints, so confirming a re-tune needs a separate (spilling) readback

`navigation.configure_nav_link` mutates the link's endpoints (`Left`/`Right`),
`Direction`, and `SnapRadius` on the proxy's `PointLinks` entry
(`Handlers/AI/NavigationHandler.cpp`, the `FNavigationLink& Link =
NavLink->PointLinks[0]; … Link.SnapRadius = …` block). But its success result
reports only the modified/ok flags — it never echoes back the resolved
`startPoint` / `endPoint` / `direction` / `snapRadius` it just wrote. So the
re-tune call gives the caller no positive confirmation of *which* endpoint
geometry and snap radius actually landed.

The natural intent — "move the endpoints to start {0,0,0} / end {350,0,140},
keep BothWays, set snapRadius 40, then confirm the re-tune took" — therefore
has no in-result confirmation path. The caller has to issue a separate read to
verify the values landed, and on a NavLinkProxy that read is `actor.describe`
(the only verb that surfaces the `PointLinks` entry's `Left`/`Right`/`SnapRadius`),
which **overflows the 10k display limit and spills to a HttpResponses file** —
forcing a manual off-disk `Read` just to confirm a 4-field re-tune.

This is the same author-then-verify round-trip gap already tracked on the
sibling navigation create verb,
[E-nav-modifier-create-no-areaclass-echo](E-nav-modifier-create-no-areaclass-echo.md)
(`create_nav_modifier_component` applies `AreaClass`/`failsafeExtent` but
echoes only `componentName`/`existsAfter`), and on
[E-set-transition-settings-no-echo](E-set-transition-settings-no-echo.md) /
[E-lighting-set-ao-exposure-no-echo](E-lighting-set-ao-exposure-no-echo.md) in
other namespaces — here it surfaces on the navigation **configure/re-tune**
verb. The closing-half of the friction (the readback spilling) is already
aggregated under
[E-actor-describe-no-header-only-read](E-actor-describe-no-header-only-read.md)
`#3` (same navigation namespace, NavLinkProxy `actor.describe` spill); **this
ticket is the upstream root cause** — if `configure_nav_link` echoed its
applied values, the spilling readback would not be needed at all.

Distinct from [B-nav-link-proxy-appends-default-link](B-nav-link-proxy-appends-default-link.md)
(IN-REVIEW): that is the duplicate-default-link / index-mismatch *bug* in
`create_nav_link_proxy` (and that `configure_nav_link` edits `PointLinks[0]`);
this is purely the verify-side **no-echo ergonomic** on `configure_nav_link`'s
own result — orthogonal to which index is live.

## What it should do

Have `configure_nav_link` add the resolved values it applied to the success
result — e.g. `startPoint` / `endPoint` (the `Left`/`Right` it set),
`direction`, and `snapRadius` (the actual values on the edited `PointLinks`
entry after the write). That closes the loop at the configure call with no
second round-trip and no dependence on the spilling `actor.describe` readback.
Additive + optional in the same spirit as `set_nav_area_class`, which already
echoes `areaClass`.

**Docs angle (`docs/wiki-src/navigation.md`):** the navigation overlay is a
4-line stub with no readback guidance. Until/unless the echo lands, it should
tell callers that `configure_nav_link` does not echo applied endpoint/snapRadius
values and that the readback verb for a NavLinkProxy's link geometry is a
**component-scoped** `actor.describe` (e.g. `nameMatch` on the link component,
via the shipped `E-component-read-filter` filters) rather than the unfiltered
`actor.describe`, which spills.

## Evidence

From the `navigation.configure_nav_link` struggle audit (namespace
`navigation`, outcome **clean**, 15 calls). The task created a NavLinkProxy
`GapJumpLink`, then re-tuned it via
`configure_nav_link {startPoint:{0,0,0}, endPoint:{350,0,140}, direction:BothWays,
snapRadius:40}`. Friction note (verbatim): *"Minor: configure_nav_link's success
result does not echo the applied snapRadius/endpoint values, so I had to read
them back via actor.describe (and first guessed actor.get_properties, which
404'd but helpfully suggested alternatives); also rebuild_navigation is async
(returns a ticket) so I had to poll system.job_status, and its result reported
nothing_to_build=true because the level has 0 nav bounds volumes."* Call-log
cost for this finding: 1 `configure_nav_link` → no echo → fallback chain of
`actor.get_properties` (404, self-recovered via suggestions) → `actor`
(wiki-nav) → `actor.find_by_name` → `actor.describe GapJumpLink` (read-back;
**payload spilled to file**) — i.e. a 404 misfire plus a spilling describe just
to confirm a 4-field re-tune that the configure result could have echoed inline.
(The async-rebuild and 0-bounds-volume halves of the note are already covered by
`B-navigation-rebuild-navigation-no-completion-signal` (DONE) and
`E-navmesh-config-requires-bounds-volume-prereq` respectively.)

## History
- `#2-echo-applied-geometry` `IN-REVIEW` developer — Implemented the additive echo. `navigation.configure_nav_link`'s success result now echoes the resolved link geometry read straight off the edited `PointLinks[0]` after the write: `startPoint`/`endPoint` (the `Left`/`Right` it set, serialized `{x,y,z}` to match the input shape via a new `MakeVectorJsonNav` helper), `direction` (BothWays/LeftToRight/RightToLeft), and `snapRadius` — closing the author-then-verify loop inline so no spilling `actor.describe` readback is needed. Mirrors the sibling `set_nav_area_class` `areaClass` echo (`NavigationHandler.cpp:442`). Reading off the stored link (not the raw payload) means the echo reflects what actually landed. File: `Source/EditorAutomationRpcGateway/Private/Handlers/AI/NavigationHandler.cpp` (handler block ~601-635; helper ~66). Regression test `FNavConfigureNavLinkEchoesAppliedGeometryTest` (`EditorAutomationRpcGateway.navigation.configure_nav_link.EchoesAppliedGeometry` in `Private/Tests/Gameplay/TestAIHandlers.cpp`) spawns a NavLinkProxy, re-tunes it (`start {0,0,0}` / `end {350,0,140}` / BothWays / `snapRadius 40`), and asserts the success result echoes back exactly those four values — fails if the echo were reverted (fields absent). The docs-overlay note (`docs/wiki-src/navigation.md`) is intentionally left for the tester/docs pass; the echo lands and removes the readback need, so the overlay's "no echo + spilling describe" steer no longer applies. Did not compile/run (later phase).
- `#1-initial-audit` `OPEN` reporter — Process/struggle audit of a NavLinkProxy re-tune task (focus `navigation.configure_nav_link`, outcome clean, 15 calls). `configure_nav_link` applies endpoints/`Direction`/`SnapRadius` to the proxy's `PointLinks` entry but its success result echoes none of them, so confirming the re-tune (`startPoint {0,0,0}` / `endPoint {350,0,140}` / BothWays / `snapRadius 40`) forced a separate readback. The agent first guessed `actor.get_properties` (404, self-recovered via the returned suggestions), then `actor.find_by_name` → `actor.describe GapJumpLink`, which overflowed the 10k display limit and spilled to a HttpResponses file (manual off-disk Read). Friction note quoted in Evidence. Same author-then-verify no-echo family as `E-nav-modifier-create-no-areaclass-echo` (sibling nav create verb), `E-set-transition-settings-no-echo`, `E-lighting-set-ao-exposure-no-echo` — different method (the navigation re-tune verb). Upstream root cause of the spilling-describe half already aggregated in `E-actor-describe-no-header-only-read` `#3` (same navigation namespace). Distinct from `B-nav-link-proxy-appends-default-link` (the duplicate-link/index bug). Other two halves of the friction note are pre-covered (`B-navigation-rebuild-navigation-no-completion-signal` DONE; `E-navmesh-config-requires-bounds-volume-prereq`). E-/docs: result was correct, the gap is verify-side echo + a navigation-overlay readback note. No existing `configure_nav_link` echo ticket (ripgrep across OPEN + closed; qmd unavailable). Proposes echoing resolved `startPoint`/`endPoint`/`direction`/`snapRadius` in the `configure_nav_link` result, plus a `docs/wiki-src/navigation.md` readback note steering NavLinkProxy geometry reads to a component-scoped `actor.describe`.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
