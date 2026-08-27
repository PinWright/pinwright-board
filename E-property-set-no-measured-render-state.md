---
id: E-property-set-no-measured-render-state
title: "property.set marks a component's render state dirty but publishes no measured field saying the renderer picked the change up"
status: OPEN
severity: Medium
category: ergonomic
tags: [property, container, render-state, measured-vs-requested, response-honesty, derived-state]
encounters: 1
lastSeen: 2026-08-27
---

# The write is now notified properly; whether the renderer saw it is still unreported

`B-property-set-container-empty-change-event` is fixed: `property.set`, `property.reset` and the 11
`container.*` mutators now emit a populated `FPropertyChangedEvent` naming the written property, and a
`UActorComponent` target additionally gets its render state marked dirty.

The original ticket also asked for a **measured** `renderStateRefreshed` field, and that is not
delivered. All 13 response shapes are unchanged: they still report only `applied` and `markedDirty`,
neither of which claims a downstream effect.

Deliberate. A field claiming a measured refresh needs a render-state probe the fix did not add, and
`rpc-design.md`'s response-honesty rule rules out reporting a request as a measurement -- publishing
`renderStateRefreshed: true` because we called `MarkRenderStateDirty` would be exactly the class of lie
the parent ticket was about.

**Fix:** a real probe. Worth scoping against how much a caller can already infer from
`markedDirty` plus a follow-up read, because this may not be worth the machinery.

## History
- `#1-unmet-half-of-the-parent-ticket` `OPEN` reporter -- Recorded by the agent fixing
  `B-property-set-container-empty-change-event`, which explicitly declined to fake the field.
