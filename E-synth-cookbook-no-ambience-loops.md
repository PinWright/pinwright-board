---
id: E-synth-cookbook-no-ambience-loops
title: "audio.synth.cookbook has no ambience / seamless-loop family, and nothing anywhere documents what makes a loop bed seam-free"
status: DONE
severity: Medium
category: ergonomic
tags: [audio, synth, cookbook, wiki, ambience, looping, discontinuity, documentation-gap]
encounters: 1
costly: 3
lastSeen: 2026-09-02T20:45:00Z
---

# The cookbook covers four one-shot families and no sustained/looping one

`audio.synth.cookbook` opens by naming itself "concrete recipes for the four SFX families the
`audio.synth` recipe grammar is built to cover" and delivers: impacts/destruction,
explosions/weapons, creatures/voices, spells/magic/UI. Every one is a **one-shot**. There is no
section for the fifth family every game ships — the **sustained ambience bed that loops forever
under a level** (wind, sea, room tone, machinery). `audio.synth.md`, `audio.synth.generate.md`
and `audio.synth.describe_schema` do not cover it either.

That is not a missing convenience. Looping is where the grammar's constraints bite hardest, and
none of the required knowledge is derivable from the schema:

1. **`analysis.technical.startDiscontinuity` / `endDiscontinuity` are undocumented.** They are
   returned by every `generate` call and are the only published measurement of whether a loop
   will click at the seam. No wiki page defines them, gives units, or says what value is
   acceptable. I used them as the seam gate on three beds purely by inferring the name.
2. **Nothing says the first and last `ampEnvelope` values must match.** They must, on every
   layer, or the bed pumps once per loop. The cookbook's advice for one-shots is the opposite
   ("omitting `ampEnvelope` ... clicks at both edges" — so envelope to zero), and following it
   on a loop bed produces a hole at the seam instead.
3. **Nothing says a `modulation.rateHz` must complete a whole number of cycles in `durationMs`.**
   `rateHz: 0.25` in a 12000 ms recipe is seamless (3 cycles); `0.18` is not, and the modulator
   steps at the wrap. The one time-varying block the grammar offers is the one most likely to
   break the loop, and the schema page presents `rateHz` as a free 0.01..20000 range.
4. **The documented "no automation on effect parameters" workaround does not transfer.** The
   cookbook's fix for a moving filter is "three to four time-offset layers each with its own
   static cutoff". For a *bed* the equivalent problem is a moving level, and the non-obvious
   part is that time-offsetting the layers' `ampEnvelope`s **flattens** the mix instead of
   animating it: my first wind bed had four layers each swinging 0.25..0.95 and measured
   `loudnessRangeLu 1.27` with a visibly flat waveform, because the layers' swells were filling
   each other's troughs. Aligning all four onto one gust rhythm with small (~300-500 ms) offsets
   was what produced audible ebb and flow. A one-line warning would have saved two renders.
5. **No target ranges for the family.** Every other family ends with concrete target bands. An
   ambience author has nothing to compare `loudnessRangeLu`, `crestDb` or `flatness` against,
   and `targets` is never scored anyway (`B-synth-targets-never-scored`), so the analysis
   scalars are all there is.

## Impact

`Medium`. Low impact class (documentation), bumped for reach: ambience beds are a standard
deliverable and this is the only synthesis path the plugin offers, so every author who tries one
rediscovers points 1-4 by trial and error. Nothing is wrong with the *engine* here — all three
of my beds came out with `startDiscontinuity` and `endDiscontinuity` at exactly `0` and the
grammar expressed everything I needed. It is purely that the page which exists to say "what is
good" stops one family short of the one with the most non-obvious rules.

**Workaround:** the five rules above, derived by rendering. Concretely, for a 12 s wind bed:
match `ampEnvelope[0].value` to the last point's `value` on every layer; pick `modulation.rateHz`
from `n / (durationMs/1000)`; align layer swells to a shared gust grid with 300-500 ms offsets
rather than anti-phase; give `master` `fadeInMs`/`fadeOutMs` of 30-60; and confirm
`startDiscontinuity`/`endDiscontinuity` are `0` before exporting.

**Fix:** add an "Ambience and loops" section to `audio.synth.cookbook` carrying those five
points and one worked multi-layer wind or room-tone recipe, and document
`startDiscontinuity`/`endDiscontinuity` (definition, units, acceptable value) wherever the
analysis scalars are described — they are currently returned by every render and explained
nowhere.

## History
- `#1-filed` `OPEN` reporter — Hit while synthesizing 11 ambience waves into `/Game/FPS/Audio/Waves/Ambience/` on EAContentExamples58 (UE 5.8 editor, gateway 27145): three seamless loop beds (12 s coastal wind, 14 s distant surf, 8 s diesel generator) plus eight one-shots. The one-shots were well served — the `modal` guidance produced a usable distant hammer and rope slap on the first render, and the `formant` section's `f0Hz`/`formantShift` split produced three distinct gull cries on the first render. The three loops had no guidance at all. Concrete costs: (a) `startDiscontinuity`/`endDiscontinuity` appear in every `analysis.technical` block and are defined on no wiki page, so I gated the seams on a field whose semantics I guessed from its name; (b) my first wind bed (candidate `c83_6fc8`, four noise layers each swinging 0.25..0.95 on offset timings) measured `loudnessRangeLu 1.27` and its waveform plot was a flat band — the offset swells cancelled into a constant sum, which is the exact opposite of the cookbook's "use time-offset layers" advice for one-shot filter motion, and took two extra renders to diagnose; (c) nothing warned that `modulation.rateHz` must divide evenly into `durationMs` to survive the wrap, which I had to reason out from first principles for all three beds. No engine defect was found — the grammar expressed every bed cleanly and all three exported with both discontinuity figures at exactly 0. Filed as ergonomic/Medium: Low docs impact bumped one level for reach, since ambience is a standard deliverable and `audio.synth` is the only synthesis path offered.
- `#2-the-missing-sixth-rule-bLooping` `OPEN` reporter — Second encounter, and it turns the documentation gap into a shipped defect. Reviewing the three beds this ticket was filed while building (`/Game/FPS/Audio/Waves/Ambience/SW_Amb_Wind_Loop`, `SW_Amb_Sea_Loop`, `SW_Amb_Generator_Loop`), `audio.authoring.describe_sound_wave` returns **`"bLooping": false` on all three**. They are seamless and they do not loop: played through an `AAmbientSound` or audio component they run once — 12 s, 14 s, 8 s — and then the level is silent. The build report cited "every loop reporting `startDiscontinuity`/`endDiscontinuity` exactly 0" as its evidence of seamlessness, which is true and which I re-derived (all three at 0, and their waveform plots show matched level at both boundaries with no fade), but that pair measures a *sample-value* property of the buffer and says nothing about whether the asset will ever wrap. The trap is that it is the only loop-named measurement in the surface, so it reads as the loop check. Add a sixth rule to the workaround list, and to whatever ambience section lands: **`audio.synth.export` leaves `bLooping` at its default `false`; set it explicitly with `audio.authoring.set_sound_wave_properties { bLooping: true, save: true }` and verify from `asset.dump` -> `sound_wave.json`, because a bed can measure perfectly seamless and still be a one-shot.** Two adjacent surfaces would each have caught this and neither does: `audio.analysis.analyze` reports the discontinuities but not `bLooping` (it is a `describe_sound_wave` field, a different verb), and `audio.analysis.audit_folder` — which does sweep for technical defects and did flag DC offset and loudness outliers across these same 63 waves — has no `looping` check at all, so an asset named `*_Loop` with `bLooping: false` sweeps clean. Either verb reporting the flag, or an `audit_folder` finding for a non-looping asset whose name or `soundGroup` implies a bed, would close it.
- `#3-audit-folder-is-blind-to-discontinuities-too` `OPEN` reporter — Third encounter of the same shape, now outside ambience, which is why it belongs here rather than in a new ticket: `startDiscontinuity` / `endDiscontinuity` are the surface's only edge-click metric, they are still undocumented, and **`audio.analysis.audit_folder` does not check them.** Its seven finding types are non-finite, digital silence, clipping, DC offset, loudness outlier, sample-rate outlier and channel-count outlier — a sweep over `/Game/FPS/Audio/Waves` reported `67 examined, 67 audited, 0 flagged` with every type at zero, while two waves in that same set carry non-zero start discontinuities: `SW_Fire_AR_Mech_B` at `0.0035` and `SW_Fire_Pistol_Mech_B` at `0.0013`. Every other one of the 67 is exactly `0`, so these are not a house style, they are two assets that missed the edge treatment the rest got — and they are weapon mechanical layers that fire on roughly half of all shots, so a step at sample 0 is a click per shot. The audit is the only tool that looks at a whole folder, it is what a build reports as its hygiene gate, and it is structurally incapable of catching the defect that the per-asset analyzer measures and returns on every call. Prior encounters of the same blindness in this ticket: `#1` (the metrics exist but are undocumented, so a builder infers their meaning), `#2` (`bLooping` is not checked either, so three beds shipped as one-shots). Ask, unchanged in shape and now with a third instance: add a `discontinuity` finding to `audit_folder` (threshold alongside `dcOffsetLinear`, which already has one) and document what the two scalars mean and what value is acceptable. A folder gate that returns `0 flagged` on a folder containing a per-shot click is worse than no gate, because the build reports it as proof.
- `#4-loop-render-seam-metric-and-cookbook-family` `IN-REVIEW` developer — Implemented as a renderer construction plus docs, because the grammar alone could not express a seam-free noise bed. Independent noise layers cannot continue into each other, so the reporter's workaround either clicked at the wrap (hard cut) or ducked at it (master fades, which is why both discontinuities read exactly 0). **Render:** new `master.loopCrossfadeMs` (default 0 = one-shot; 1 ms .. `durationMs/2`; the parser refuses it with nonzero `fadeInMs`/`fadeOutMs`, at path `master.loopCrossfadeMs`). `PwRenderRecipe` renders that far past `durationMs` (envelopes hold their last value), runs the master chain over the extended bus, then `FoldLoopOverhang` crossfades the overhang onto the head before normalize and truncates, so `out[N-1] -> out[0]` is the render's own `x[N-1] -> x[N]` step. The crossfade law adapts to the head/overhang correlation: equal power for noise, equal gain for an integer-cycle drone, so there is no 3 dB bump. Serialized and published in `describe_schema` (topology `recipe`/`timing`, `defaults`). **Measurement:** new `analysis.technical.loopSeamRatio` = `|last-first| / RMS(adjacent-sample step)`, below about 3 when seamless, omitted when the buffer never moves. It answers `#1`(a) and the "the only loop-named metric is not a loop metric" trap from `#2`. **Docs:** `audio.synth.cookbook` gains an "Ambience and loops" family covering all five rules plus the `bLooping` export rule from `#2` and PCM-only, a worked 12 s wind bed, and targets. `audio.analysis` defines `startDiscontinuity`/`endDiscontinuity` (absolute first/last mono sample, linear 0..1, the playback start/stop step; one-shots want 0, a correct loop has both nonzero) and `loopSeamRatio`. `audio.synth` documents the field. **Split out:** the `audit_folder` asks of `#2`/`#3` (an edge-discontinuity finding, a loop-seam finding and a `bLooping` check) are a different verb with their own fixtures, now `F-audit-folder-no-discontinuity-or-looping-check`. Files: `Source/PinWright/Private/AudioGen/PwSynthRecipe.{h,cpp}`, `PwSynthDsp.cpp`, `PwAudioAnalysis.{h,cpp}`, `Handlers/Audio/AudioSynthSchemaHandler.cpp`, new `Tests/Media/TestPwSynthLoop.cpp`, `docs/wiki-src/audio.synth.cookbook.md`, `audio.analysis.md`, `audio.synth.md`, `CHANGELOG.md`. Tests (offline DSP, no audio device): `PinWright.audio.synth.loop.FoldMakesTheWrapAnOrdinaryStep` (a 101.25 Hz sine one-shot wraps at ratio > 50, folded at <= 1.5 with the crossfade region within 0.5 dB of the body; fails if the fold is reverted), `PinWright.audio.synth.loop.CookbookWindBedIsSeamless` (renders the cookbook recipe verbatim: ratio < 4, edges not faded, < 3 dB level step across the wrap), and `PinWright.audio.synth.loop.ParserGuardsAndRoundTrip`. Behaviour change: the canonical recipe now always carries `master.loopCrossfadeMs`, so the recipe digests of existing candidates change.
- `#5-review-hardening` `IN-REVIEW` developer — Review follow-up, no product-code change. `FoldMakesTheWrapAnOrdinaryStep` now also renders a 100 Hz / 1000 ms whole-cycle drone (r = 1), so the crossfade's equal-gain path is covered: a fixed equal-power law would bump +3 dB and fail the 0.5 dB check. `CookbookWindBedIsSeamless` now asserts `loopSeamRatio` < 3, the cookbook's own gate. It adds a hard-cut control: the same recipe with `loopCrossfadeMs: 0` over seeds 21..28 must exceed 3 on at least 5 seeds and reach above 10 on at least one. The control spans eight seeds because one seed's wrap step is a single random draw. The measured values are printed in the assertion text. Cookbook wording: the bed checks are described as `analysis.technical` fields, not recipe `targets`. The "hard-cut reads in the tens" claim is softened to match what the control asserts. Two loop-render side effects are documented: `render.layers[].framesMixed` counts the overhang, and a `granular` layer's position sweep spans `durationMs + loopCrossfadeMs`.
- `#6-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit 80034666). run3/full passed non-skipped: `PinWright.audio.synth.loop.FoldMakesTheWrapAnOrdinaryStep`, `.CookbookWindBedIsSeamless` and `.ParserGuardsAndRoundTrip`. The cookbook's worked wind bed renders verbatim with `loopSeamRatio` < 3, and the hard-cut control exceeds 3 on most seeds. The fold covers the equal-power (noise) and equal-gain (whole-cycle drone) paths within 0.5 dB. Docs at HEAD: `audio.synth.cookbook.md` has "Ambience and loops" with rules for matched first/last ampEnvelope values, whole-cycle `modulation.rateHz`, aligned rather than staggered swells, and bed targets. It also has the `#2` bLooping export rule and the 12 s wind recipe. `audio.analysis.md` defines `startDiscontinuity`/`endDiscontinuity` and `loopSeamRatio`. The cookbook supersedes the reporter's master-fade workaround with `master.loopCrossfadeMs`. The `audit_folder` asks of `#2`/`#3` moved to `F-audit-folder-no-discontinuity-or-looping-check`.
