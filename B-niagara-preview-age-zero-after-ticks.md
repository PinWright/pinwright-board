---
id: B-niagara-preview-age-zero-after-ticks
title: "Niagara preview capture reports `achievedAgeSeconds: 0` alongside a non-zero `tickCount` — by the provider's own contract that means the simulation really is at age 0 after 480 ticks, so no captured Niagara system can be proven to animate"
status: OPEN
severity: High
category: bug
tags: [render, capture_asset_preview, niagara, capture-subject, simulation, evidence-field, animation-proof]
encounters: 1
lastSeen: 2026-08-27T18:47:15+05:00
---

# `render.capture_asset_preview` on a Niagara system reports 480 simulated ticks and `achievedAgeSeconds: 0` in the same response

A Niagara preview capture with a non-zero requested age returned
`achievedAgeSeconds: 0` while the same response reported **480** simulated ticks.
Either the simulation genuinely is not advancing — in which case every Niagara
preview capture shows frame 0 and no captured system can ever be proven to
animate — or the field is misreported and the plugin's only published
proof-of-motion for Niagara is worthless. Both readings are serious.

**The provider's own contract makes the first reading the likely one.** See
below: `achievedAgeSeconds` is emitted *only* when the age was actually read off
the running simulation, and its absence is deliberately reserved to mean
something different. A present zero is therefore a measurement, not a default.

## Root cause (what the source constrains, and what it does not)

The drive step, `Plugins/PinWright/Source/PinWright/Private/Handlers/Render/CaptureSubjectProviders_Niagara.h:387-393`:

```cpp
        // MEASURED, not assumed: bSimulated is what the response publishes.
        OutStep.bSimulated = EnsureSystemInstance(Component);
        if (OutStep.bSimulated && OutStep.TickCount > 0)
        {
            Component.AdvanceSimulation(OutStep.TickCount, TickDeltaSeconds);
        }

        OutStep.bAgeMeasured = TryReadSimulatedAge(Component, OutStep.AchievedAgeSeconds);
```

`TryReadSimulatedAge` (`CaptureSubjectProviders_Niagara.h:283-291`) returns false
when there is no valid controller, and otherwise reads
`Controller->GetAge()` directly:

```cpp
        FNiagaraSystemInstanceControllerConstPtr Controller = Component.GetSystemInstanceController();
        if (!Controller.IsValid() || !Controller->IsValid())
        {
            return false;
        }
        OutAgeSeconds = static_cast<double>(Controller->GetAge());
        return true;
```

And the response block, `CaptureSubjectProviders_Niagara.h:697-702`:

```cpp
        // Present only when it was actually read off the running simulation. Absent means "no
        // instance existed to ask", which is a different statement from "the age was 0".
        if (Step.bAgeMeasured)
        {
            Detail->SetNumberField(TEXT("achievedAgeSeconds"), Step.AchievedAgeSeconds);
        }
        Detail->SetBoolField(TEXT("ageMeasured"), Step.bAgeMeasured);
```

**What that pins down.** The provider deliberately distinguishes "no instance to
ask" (field absent, `ageMeasured: false`) from "the age was 0" (field present at
0). So a **present** `achievedAgeSeconds: 0` means a valid controller existed,
was asked, and answered zero — after `AdvanceSimulation` had been called with 480
ticks. That is a genuine failure to advance, not a reporting default.

**What is NOT established, and must not be written into a fix.** Whether
`AdvanceSimulation` failed to advance the instance, or the instance was reset or
re-created between the advance at `:390` and the read at `:393`, is untraced. The
session that found this did not record whether the field was *present* at 0 or
merely reported as 0 by the reader, so **the first thing a fixer should do is
re-run the capture and check `ageMeasured` in the raw response.** If
`ageMeasured` is `false`, this is an absent-field-read-as-zero reporting problem
and a much smaller ticket. If it is `true`, the simulation is not advancing.

## Verbatim repro

```
render.capture_asset_preview {subject: {kind: "niagara",
                                        path: "/Game/Atlantis/VFX/NS_Bubbles_Stream"},
                              times: [2.0]}
```

Observed on `/Game/Atlantis/VFX/NS_Bubbles_Stream`, 2026-08-27, UE 5.8, this
checkout. The response carried 480 simulated ticks and `achievedAgeSeconds: 0`.

**Read `ageMeasured` and `tickCount` off the raw response**, not a summary — the
whole diagnosis turns on whether `achievedAgeSeconds` was present.

## Impact

`render.capture_asset_preview` with `time` / `times` is the plugin's published
method for proving a particle system animates: drive it to an instant, capture,
compare instants. If the age is not advancing, every such proof on this build is
a picture of frame 0 compared against another picture of frame 0, and a
difference of zero would be read as "the system does not animate" when the
capture harness is what failed. If instead the field is misreported, the one
number a caller would check to catch that is itself unreliable.

Re-verifying this is currently expensive and dangerous: repeated Niagara preview
captures are what took the editor down four times in the session that found this
(`B-capture-asset-preview-no-safe-close-mode`), so the natural
investigate-by-repetition loop is blocked behind a Critical crash.

## Distinct from related tickets

- `B-capture-asset-preview-no-safe-close-mode` (Critical) — blocks the
  investigation of this ticket, because it makes repeated Niagara preview
  captures unsafe. Fix that first, or investigate this one through the level
  route instead.
- `B-niagara-api-authored-emitters-do-not-simulate` (if filed from the same
  session) — an emitter authored module-by-module through `niagara.add_module`
  renders a static clump and never moves, with a stock engine system in the same
  level moving correctly. **Same family of question, different surface**: that
  one is about the asset not simulating in the level, this one about the preview
  provider not advancing age. Worth checking whether one explains the other —
  `NS_Bubbles_Stream` is the asset in both.
- `F-animated-capture-verbs` (DONE, Medium) built `poseChanged` / `poseSampled`
  evidence fields for *skeletal* animation capture and explicitly scoped Niagara
  out. It is the precedent for what a trustworthy motion-evidence field looks
  like; the Niagara provider's equivalent is what is in question here.

severity rationale: impact=an evidence field the caller relies on to prove motion is either wrong or reporting a real failure to simulate, and either way a downstream conclusion ("this system does not animate") is drawn from it on a normal path x reach=the only published method for proving any Niagara system animates -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Carried over from the session defect log as an **unresolved observation**: a Niagara preview capture on `/Game/Atlantis/VFX/NS_Bubbles_Stream` returned `achievedAgeSeconds: 0` while reporting 480 simulated ticks. The original entry could not tell whether the age genuinely is not advancing or the field is misreported, and it was not traced into source. Traced into source during filing: `CaptureSubjectProviders_Niagara.h:387-393` calls `Component.AdvanceSimulation(TickCount, TickDeltaSeconds)` then immediately `TryReadSimulatedAge` (`:283-291`, reading `Controller->GetAge()`), and the response block at `:697-702` emits `achievedAgeSeconds` **only** when `bAgeMeasured`, with an explicit comment that absence means "no instance existed to ask", "a different statement from 'the age was 0'". By that contract a PRESENT zero is a real measurement of a simulation that did not advance — which narrows the original either/or substantially. The narrowing is conditional and stated as such: the session did not record whether the field was present or absent, so the first action for a fixer is to re-run and read `ageMeasured` in the raw response. Mechanism (advance failing versus instance reset between `:390` and `:393`) remains untraced and is NOT filed as a finding. Not re-verified live during filing: repeated Niagara preview captures are blocked behind `B-capture-asset-preview-no-safe-close-mode`, which killed the shared editor four times in that session.
