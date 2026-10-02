---
id: F-smooth-skin-bone-exclusion
title: "Allow bones to be excluded from smooth skin solves"
status: WONTFIX
severity: Low
category: feature
tags: [skinning, geometry, model]
encounters: 1
---

# Allow bones to be excluded from smooth skin solves

Neither `geometry.bind_skin_weights` nor `.pwmodel` smooth skinning lets a caller remove named
bones from the automatic solve. Both pass the full reference skeleton to the engine solver, whose
options contain no bone filter. Rigid part binding can overwrite selected vertices afterward, but
it cannot stop an unwanted bone influencing the remaining smooth region.

The product shape needs evidence before an API is chosen. A likely shared rule is a validated set
of excluded bone names used by both entry points, with explicit exact-bone versus descendant
semantics and readback proving excluded bones received zero weight. Do not add a one-off parameter
until those semantics and at least one real use case are agreed.

## History
- `#1-exclusion-gap-confirmed` `OPEN` reporter — Source, parameter tables, docs, and the engine solve options expose no bone exclusion; filed for product-shape review rather than adding a speculative API.
- `#2-re-rated` `OPEN` triager — Severity Medium -> Low. With a post-solve route available (`skeleton.set_vertex_weights`, or rigid part binding for selected vertices) the impact is at most Medium's many-extra-calls band, and the ticket records no encounter or real use case yet (it asks for product-shape review first), which is a rare edge path — bump down one to Low.
- `#3-stale-sweep-yagni` `WONTFIX` developer — The ticket itself records no use case and asks for product-shape review before any API (`#1`, `#2`), so it is parked with no gate; the board's Deferring rule says an ungated defer is a WONTFIX. A post-solve route exists: `skeleton.set_vertex_weights` (registered at plugin HEAD `10212ee4`, `SkeletalMeshHandler.cpp:1073`) or rigid part binding for selected vertices. Reopen with a real skinning task that needs exclusion.
