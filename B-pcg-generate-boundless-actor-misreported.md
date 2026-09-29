---
id: B-pcg-generate-boundless-actor-misreported
title: "pcg.generate on an actor with no bounds logs PCG engine Errors and answers PCG_GENERATION_CANCELLED"
status: IN-REVIEW
severity: Medium
category: bug
tags: [pcg, honest-errors, log-noise, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T10:26:43Z
---

# A boundless actor is reported as a cancelled generation

PCG generates inside its owner's bounds (`PCGHelpers::GetGridBounds`: the union of the actor's primitive components, or a
landscape's extent). `pcg.generate` never checked them. On an actor with no primitive component it added and registered a
`UPCGComponent` (engine `LogPCG: Error: [RegisterOrUpdateExecutionSource] Component has invalid bounds, not registered nor updated.`,
`PCGTrackingManager.cpp:265-268`), then the generate logged `LogPCG: Error: [ScheduleComponent] Didn't schedule any task.` plus two
`Process Graph was called but aborted` warnings, and the synchronous abort fired `OnPCGGraphCancelledDelegate`, so the caller got
`PCG_GENERATION_CANCELLED` ("cancelled or aborted before completing") naming no cause.

The misreport also hid a test gap: `PinWright.pcg.generate.TicketedKickoffDoesNotBlock` built its fixture on a bare `AActor`, so it
only ever reached its `generation-resolved-inline` skip (`PCG_GENERATION_CANCELLED`), attributed in its comment to "a bare automation
world without a live PCG scheduler"; the ticket-contract assertions never ran. Evidence: `Saved/Logs/pw_gapwave_full_offscreen2.log:43901-43909` (errors at 43901, 43903).
Four more `pcg.generate.*` tests logged the registration Error from the same fixture shape (`:43842,43849,43874-43875,43882`).

**Fix:** refuse inline with `ACTOR_HAS_NO_BOUNDS` before the component is added; give the test fixtures a box root.

## History
- `#1-boundless-actor-cancelled` `OPEN` reporter — Found while triaging tests newly exposed by the per-test `bSuppressLogErrors` reset: five pcg.generate tests logged "Component has invalid bounds"; the ticketed-kickoff test skipped its assertions on every run for the same cause.
- `#2-bounds-guard-and-fixture` `IN-REVIEW` developer — `PCGGenerateHandler.cpp` checks `PCGHelpers::GetGridBounds(Actor, nullptr).IsValid` right after actor/graph resolution and answers `ACTOR_HAS_NO_BOUNDS` (registered code) before touching the actor; `docs/wiki-src/pcg.md` lists it among the inline codes. `TestPCGGenerateHandler.cpp` gains `PcgGenerateTestFixture::GiveActorBounds` (registered `UBoxComponent` root, extent 500, no collision) called before every PCG component registration, and a new `PinWright.pcg.generate.BoundlessActorRejected` asserting the code and that no component was added. Compile-checked (`-SingleFile`) only; needs a suite run, which must show `TicketedKickoffDoesNotBlock` WITHOUT its skip marker — with bounds the generation now schedules and the ticket path runs for the first time.
