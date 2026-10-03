---
id: B-property-set-wiki-construction-rerun
title: "property.set wiki says the verb skips the path that reruns construction scripts, but its PostEditChangeProperty reruns them on every placed actor target"
status: DONE
severity: Low
category: bug
tags: [property, wiki, construction-script, docs, gap-analysis-2026-09-30]
encounters: 1
---

# property.set wiki denies a construction-script rerun that happens

The `property.set` wiki (`docs/wiki-src` source of `Saved/PinWright/wiki/property.set.md:64-67`) says the verb "deliberately skips" the editor's `PreEditChange`/`PostEditChange` reregister pair, "that path flushes rendering commands and reruns construction scripts". But the verb calls `PostEditChangeProperty` on the target (`PinWright::NotifyPropertyChanged`, `Source/PinWright/Private/Utils/PropertyChangeNotify.h:88-110`). For a placed actor, `AActor::PostEditChangeProperty` reruns construction (`C:\UE_5.8\Engine\Source\Runtime\Engine\Private\ActorEditor.cpp:~225/240`) whenever `ReregisterComponentsWhenModified()` holds (`:341`: not a template, not PIE, has a world).

A caller reading the page believes an actor property write leaves construction-built components alone. In fact they are destroyed and rebuilt, which can reset per-instance state. The pinwright.com/compare note for "Construction script preview" repeats the same wrong reading.

**Fix:** state the side effect on the page: an actor-target write reruns that actor's construction script; a component or CDO target does not. Optionally report it in the response (for example `constructionRerun: true`), measured, not inferred.

**Acceptance:**
- On a placed actor whose construction script adds components, `property.set` of an actor property changes the construction-built component set as the page now describes.
- The page no longer claims the rerun is skipped for actor targets.

Effort S.

## History
- `#1-wiki-contradicts-engine` `OPEN` reporter — Found during the 2026-09-30 competitor gap analysis. Engine source shows the rerun (`ActorEditor.cpp:~225/240`, gate at `:341`); the wiki says the opposite.
- `#2-docs-and-acceptance-test` `IN-REVIEW` developer — Premise confirmed at 7230b41d: `docs/wiki-src/property.md` still said the verb skips the reregister pair that "reruns construction scripts"; engine reruns via `AActor::PostEditChangeProperty` (`ActorEditor.cpp:170` gate, `:225`/`:240` rerun, `ReregisterComponentsWhenModified` `:341`). Docs-only fix, no behaviour change: the page now scopes the skip to component targets and adds a paragraph stating an actor-target write reruns construction, destroys and rebuilds construction-built components, resets state the component instance-data cache does not carry, is not reported in the response, and does not happen for `ActorLabel`, PIE actors (Simulate excepted), CDO/archetype targets, or component targets (incl. dotted hops). Same correction to the `WHY NO PreEditChange` comment in `Source/PinWright/Private/Utils/PropertyChangeNotify.h` and the `Notify without PreEditChange` bullet in `docs/rpc-design.md`; CHANGELOG 1.0.0 entry. The optional `constructionRerun` response field was not added (callers now learn it from the page; add it if an agent still loses state unaware). Not fixed here: the pinwright.com/compare "Construction script preview" note is not in this repo. Tests (`Source/PinWright/Private/Tests/Utility/TestPropertySetConstructionRerun.cpp`): `PinWright.property.set.ActorTargetRerunsConstructionScript` (BP with one SCS component, placed in the editor world: a component-target write keeps the component, an actor-target `InitialLifeSpan` write destroys it and a new one is built) and `PinWright.infra.wiki_handler.MethodPage.PropertySetConstructionRerun` (rendered page states the rerun, old wording gone).
- `#3-verified-linux` `DONE` tester — Verified on the committed tree, PinWright ae877ccc (pushed to origin/master), UE 5.8 Linux Vulkan. run3/full: offscreen full suite, 5827/5827 passed, 0 failed, 73 skipped. Fix commit 4363f322. Passed non-skipped in run3/full: `PinWright.property.set.ActorTargetRerunsConstructionScript` and `PinWright.infra.wiki_handler.MethodPage.PropertySetConstructionRerun`. Acceptance 1: on a Blueprint with one SCS component placed in the editor world, a component-target write keeps the component. An actor-target `InitialLifeSpan` write destroys it and builds a new one, as the page now says. Acceptance 2: the rendered `property.set` page states the rerun, and the old "deliberately skips ... reruns construction scripts" wording is gone. The doc test asserts both. The commit also corrects `PropertyChangeNotify.h`, the `UtilityPropertyHandler.cpp` comment and `docs/rpc-design.md`, and adds the same note to `property.reset` and `container.md`. The optional `constructionRerun` response field was not added; the ticket marked it optional. Not in this repo: the pinwright.com/compare "Construction script preview" note. The site owner has to correct it.
