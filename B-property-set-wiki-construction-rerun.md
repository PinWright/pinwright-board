---
id: B-property-set-wiki-construction-rerun
title: "property.set wiki says the verb skips the path that reruns construction scripts, but its PostEditChangeProperty reruns them on every placed actor target"
status: OPEN
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
