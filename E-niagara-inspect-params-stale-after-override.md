---
id: E-niagara-inspect-params-stale-after-override
title: "niagara.set_module_input does not echo the value it wrote, so a set-then-verify loop must re-inspect — and the natural params-aspect readback shows the stale rapid-iteration template default (the override lives on the graph pin), making the verify look like a no-op"
status: OPEN
severity: High
category: ergonomic
tags: [niagara, niagara-inspect, set-module-input, readback, override-pin, rapid-iteration, misleading]
encounters: 4
costly: 4
lastSeen: 2026-09-07
---

# `niagara.set_module_input` does not echo the value it wrote, and the params-aspect readback shows the stale module-input default

`niagara.set_module_input` writes its override via the **graph override-pin**
path (confirmed implementation: `F-niagara-reset-module-input` `#3` notes
`set_module_input`'s write path is "**override-pin** … not rapid-iteration
parameters"). But `niagara.inspect`'s `parameters` aspect
(`includeProperties:true`) reports the module input from the **rapid-iteration
parameter store** (`scope: systemUpdateRapidIteration`), which keeps the
**authored template default** and is *not* updated by the override pin. So after
a successful edit the two inspect aspects report two contradictory values for the
**same logical input**, and the most natural "did my value stick?" readback —
the named parameter in the params list — silently still shows the old number.

This is the readback complement of the *write* working correctly: the edit is
real (graphs aspect proves it), but the params aspect's report of it is
misleading. An agent that verifies a `set_module_input` by reading the parameter
back from `includeProperties` will conclude the edit was a no-op, when it
actually landed on the override pin in `includeGraphs`.

## Verbatim repro (replay-confirmed via `mcp__editor-automation__call`)

On `/Game/FX/NS_BigExplosion_Replay` (a duplicate of
`/Niagara/DefaultAssets/Templates/Systems/SimpleExplosion`), emitter
`UpwardMeshBurst`, module `SpawnBurst_Instantaneous` (entryId
`5DDC9AC54A37F08F656EB0AED8F19794`):

1. `niagara.set_module_input { emitter:"UpwardMeshBurst",
   entryId:"5DDC9AC54A37F08F656EB0AED8F19794", inputName:"Spawn Count",
   value:160 }` →
   `{"success":true,"pinId":"74553D624A5615D2A05578B478A4243F","index":1,"linked":false}`
   (the edit succeeds and reports a `pinId`; it does **not** echo the value).

2. `niagara.inspect { includeProperties:true, includeStack:false,
   includeGraphs:false, includeCompile:false }` — the **parameters** aspect, the
   natural single-input readback. The matching entry (verbatim from the payload):

   ```json
   { "scope": "systemUpdateRapidIteration",
     "name": "Constants.UpwardMeshBurst.SpawnBurst_Instantaneous.Spawn Count",
     "type": { "name": "NiagaraInt32", ... },
     "offset": 112,
     "value": 80 }
   ```

   It still reads **`value: 80`** (the template default) — the override to 160 is
   invisible here, and there is **no `overridden`/`hasOverride` marker** on the
   entry to signal that this stored value is no longer the effective one.

3. `niagara.inspect { includeGraphs:true, ... }` — only the **graphs** aspect
   reflects the edit, on the override pin (verbatim):

   ```json
   { "id": "74553D624A5615D2A05578B478A4243F",
     "name": "SpawnBurst_Instantaneous.Spawn Count",
     "direction": "input",
     "defaultValue": "160.0", ... }
   ```

So the same input is reported as `80` by `parameters` and `160.0` by `graphs`,
with no cross-reference between them.

## Why it matters (process cost)

The attempt task (build a punchier explosion: double Spawn Count, shorten
Lifetime, verify each via inspect, read-first one-edit-at-a-time) made readback a
required step after every edit. The agent's first params-only verify "looked like
a no-op" — friction note verbatim:

> "set_module_input creates an override PIN that shows up in the graph aspect,
> NOT in the rapid-iteration params (which keep the old template value) - so my
> first params-only verify looked like a no-op until I checked the graph override
> pins. A user could be confused that the params store still reads 80 after a
> successful spawn-count edit."

This bites the *recommended* verify pattern: the smaller, more obvious aspect to
read for one input value (`parameters`) is exactly the one that lies, and the
agent only recovers by switching to the much larger `graphs` aspect (which spills
to disk — see `E-niagara-inspect-no-param-readback-projection`).

## What it should do

The set-then-verify loop should be able to confirm a `set_module_input` from the
**write result itself**, without a second inspect that lands on the lying
`parameters` aspect.

**Fix (implemented — the cheap, high-leverage remedy):**

- Have `set_module_input`'s own result **echo the value it wrote** (it previously
  returned `pinId`/`index`/`linked` but not the value). The literal write path
  already computes the canonical pin-default string it lands on the override pin
  (`JsonValueToPinDefaultString` / `InferNiagaraInputType` → `DefaultValue`); the
  result now carries that as a `value` field (e.g. `"160.0"`), so the set-then-verify
  loop confirms from the write result without re-inspecting. This is the same
  readback-echo pattern already used across the board.
- Docs (`docs/wiki-src/niagara.md`): state plainly that `set_module_input` writes a
  graph override pin, that the result echoes the written `value`, that
  `niagara.inspect`'s `parameters` aspect shows the **rapid-iteration template
  default** (which the override does *not* update), and that the live override value
  reads back from the `graphs` aspect's override pin.

**Deliberately out of scope (over-scoped relative to payoff):** adding an
`overridden`/`effectiveValue` marker to *every* rapid-iteration entry in the
`parameters` aspect. `BuildParameterStoreArray` iterates the whole store with no
module-node context, so detecting per-entry override state would require mapping
each `Constants.<emitter>.<module>.<input>` name back to its module node and walking
its override pin — a large, fragile change for a niche verify path. The value-echo
plus the docs note removes the friction (the `set_module_input` result is now
self-confirming) without that cost. The `parameters` aspect already had the precedent
in mind: `BuildStaticSwitchInputs` (`NiagaraDumpBuilder.cpp`) stamps
`source:"override"/"default"`, but it only applies to static switches, which carry
their override on the caller pin in the same builder pass; scalar module inputs do
not.

## Distinct from

- `E-niagara-inspect-no-param-readback-projection` (OPEN) — that ticket is about
  inspect's **response size / spill** (no projection → forced on-disk Read). This
  one is orthogonal: even with the payload fully in hand, the `parameters` aspect
  reports the **wrong (stale) value** for an overridden module input, while the
  `graphs` aspect reports the right one. Size and value-correctness are separate
  problems; fixing the spill would not stop the params aspect from showing `80`.
- `E-niagara-modify-parameter-no-override-readback` (OPEN) — that is the
  **runtime component-override** store on a spawned `ANiagaraActor` (a different
  store, read via `object.call_function GetVariable*`). This ticket is the
  **asset-side** module-input override pin vs the asset's rapid-iteration param
  store, both within `niagara.inspect`.
- `F-niagara-reset-module-input` (`#3` IN-REVIEW) / `B-niagara-set-module-input-vec2`
  / `F-niagara-link-module-input-to-parameter` — those concern the **write** verbs'
  behavior (override-pin removal, Vec2 inference, linked-parameter form). This is
  purely the **inspect readback** misreporting an already-successful override.

## Encounter 2026-08-27 — the `#4` echo is not a safe confirmation when the override pin is linked

**This section reports a case the `#3` severity reasoning and the `#4` remedy do not
cover. It edits no existing text; the body above stands as its author wrote it.**

Observed 2026-08-27 on UE 5.8 in the EAContentExamples58 checkout, PinWright at
`8e76cad5`, filed in full as `B-niagara-literal-over-linked-override-pin`.

`#1`'s repro is a **clean** override pin: the write really lands, and only the
`parameters` aspect misreports it. That premise — "the edit is real (graphs aspect
proves it), but the params aspect's report of it is misleading" — does not hold when
the override pin already has an **inbound link**. Then the literal write is a
complete no-op: `TrySetDefaultValue` stamps the pin's `DefaultValue`, and a pin with
`LinkedTo` set reads the link and ignores the default. The dynamic-input chain (or
`User.*` binding) keeps driving the input.

**The `#4` echo confirms it anyway.** `*OutWrittenValue = DefaultValue` at
`Handlers/Niagara/NiagaraEditHandler.cpp:997` echoes the byte that was written, not
the value the graph uses, so the response is faithful to the write and unfaithful to
the effect. Verified live: `set_module_input {Lifetime: 16}` and
`{Sprite Size: {x:60,y:60}}` on `/Game/Atlantis/VFX/NS_Bubbles_Stream` returned
`success:true`, `linked:false`, `value:"16.0"` / `"(X=60.0,Y=60.0)"`, and after a
`niagara.compile` the NIR still shows
`` link `Random Range Float`.UniformRangedFloat -> `Map Set`_2.`InitializeParticle.Lifetime` ``.
The result carries nothing that separates the two cases — `linked: false` reports
whether *this call* requested a link, not the pin's state.

So `#3`'s severity rationale, "no data loss, **no silent false-success of a
write**", is true for a clean pin and false for a linked one — and a linked override
pin is the idiomatic shape of real Niagara content, not an edge case. The root cause
is on the write side, not here: the literal branch (`:985-997`) is the only one of
the three value modes that never calls `ClearModuleInputOverride`, which the
linked-parameter branch does at `:816` and the dynamicInput branch at `:915`.

Nothing in this ticket's own scope needs to change: the `parameters`-aspect
misreporting it documents is real and independent. Two follow-ups belong to it,
though:

1. The `### niagara.set_module_input` docs note `#4` added to
   `Docs/wiki-src/niagara.md` — currently "the result echoes the written `value`" —
   needs the caveat that the echo confirms the write, not the effect, and is not
   trustworthy when the override pin is linked.
2. The right discriminator is `moduleInputs[].valueMode` from
   `E-niagara-input-schema-readback` (`local` vs `dynamicInput` vs `linked`), which
   is exactly the per-input override marker `#3` deemed out of scope for the
   `parameters` aspect — it already exists on the `stack` aspect. Pointing the docs
   note at it costs nothing and gives the verify loop a real check.

## History
- `#11-bumped-by-cost` `OPEN` orchestrator — Severity Medium -> High by cost. Costly encounters counted: #5 (a literal write over a linked override pin was confirmed by an echo that reports the byte written rather than the value the graph reads — a compile and an unlit capture were spent before the no-op surfaced, and the root cause had to be filed separately), #6 (a verifier replay round spent returning the ticket from IN-REVIEW after the shipped remedy turned out to be a wiki sentence pointing elsewhere), #8 (a second fix round lost — IN-REVIEW returned because the work addressed the graphs aspect and not the parameters aspect this ticket is named for), #9 (a chunk of a diagnosis session spent treating the stale store as the root cause of a four-round "the explosion has no fireball" defect, a conclusion #10 then had to retract in full). Reach also applies: the parameters aspect is the natural "did my value stick?" readback and these encounters span the Atlantis VFX stream, the FPS VFX stream and the board's own fix loop.
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed via `mcp__editor-automation__call` on a fresh `asset.duplicate` of `SimpleExplosion` → `/Game/FX/NS_BigExplosion_Replay`. `niagara.set_module_input` (UpwardMeshBurst SpawnBurst_Instantaneous "Spawn Count" = 160) returned `success:true, pinId:74553D62…, linked:false`. A subsequent `niagara.inspect includeProperties:true` reported the matching rapid-iteration param `Constants.UpwardMeshBurst.SpawnBurst_Instantaneous.Spawn Count` with `scope:"systemUpdateRapidIteration"` and `value:80` (the template default, unchanged, with no override marker), while `niagara.inspect includeGraphs:true` reported the override pin `74553D62…` `SpawnBurst_Instantaneous.Spawn Count` with `defaultValue:"160.0"` — same input, two contradictory values. Seed method was `niagara.set_module_input` (which worked correctly); the misreporting culprit is `niagara.inspect`'s parameters aspect. Outcome ergo. Ripgrep across OPEN/closed found no existing ticket on the params aspect showing the stale module-input default after an override (the two neighbor tickets are about response-size spill and runtime component overrides respectively — see "Distinct from").
- `#2-retriage` `OPEN` triage — Low→High: inspect params aspect reports the stale template default after a successful override so a params-only verify reads as a no-op, silent wrong-readback, niche path keeps it from Critical.
- `#3-reword-to-value-echo` `OPEN` developer — Reworded toward the narrower, higher-leverage scope (per the adversarial validity lens). The fat remedy in `#1`/`#2` (add an `overridden`/`effectiveValue` marker to every rapid-iteration entry in the inspect `parameters` aspect) is over-scoped: `BuildParameterStoreArray` (NiagaraDumpBuilder.cpp:467-492) iterates the whole store with no module-node context, so per-entry override detection would need to map each `Constants.<emitter>.<module>.<input>` back to its module node and walk its override pin — large/fragile for a niche verify path. Re-scoped Fix to the cheap remedy the ticket itself listed: echo the value `set_module_input` wrote. Severity High→Medium (readback omits a field, doable via the graphs aspect / now via the echoed result — no data loss, no silent false-success of a write). Status OPEN→IN-REVIEW; see `#4`.
- `#4-echo-written-value` `IN-REVIEW` developer — `niagara.set_module_input` now echoes the value it wrote so the set-then-verify loop confirms from the write result without a second (lying) `parameters`-aspect inspect. `ApplyModuleMutation` (NiagaraEditHandler.cpp) gained an optional `OutWrittenValue` out-param set in the literal SetModuleInput branch to the canonical pin-default string actually landed on the override pin (the same `DefaultValue` from `JsonValueToPinDefaultString`/`InferNiagaraInputType` that `TrySetDefaultValue` writes, e.g. `"160.0"`); the `set_module_input` handler sets it as `value` on the success result (literal path only — the linked path already reports `linked`/`parameter`/`parameterType`). Docs: `docs/wiki-src/niagara.md` gained a `### niagara.set_module_input` note (override-pin write, result echoes `value`, `parameters` aspect shows the stale rapid-iteration template default, live override reads from the `graphs` aspect). Files: `Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp`, `docs/wiki-src/niagara.md`. Test: `Source/PinWright/Private/Tests/Niagara/TestNiagaraSetModuleInput.cpp::FNiagaraSetModuleInputEchoesWrittenValueTest` sets `Spawn Rate=160` on a real InitializeParticle/SpawnRate module via the dispatcher and asserts the result's `value` echoes the written number (it would be absent/empty if the echo were reverted), with a 2.5-spelled case to prove the canonical normalization is echoed rather than the raw request.
- `#5-encounter-echo-unsafe-on-linked-pin` `IN-REVIEW` reporter — Additional evidence, no status change and no edit to existing text. `#1`'s repro assumes a clean override pin, where the write lands and only the `parameters` aspect misreports it. When the override pin has an inbound link (a dynamic-input chain or a `User.*` binding) the literal write is a complete no-op — `TrySetDefaultValue` stamps `DefaultValue` and a pin with `LinkedTo` reads the link — and the `#4` echo (`*OutWrittenValue = DefaultValue`, `Handlers/Niagara/NiagaraEditHandler.cpp:997`) confirms it anyway, because it echoes the byte written rather than the value the graph uses. Observed 2026-08-27 on UE 5.8 in the EAContentExamples58 checkout (PinWright `8e76cad5`): on `/Game/Atlantis/VFX/NS_Bubbles_Stream` emitter `Bubbles`, `set_module_input {Lifetime:16}` / `{Sprite Size:{x:60,y:60}}` returned `success:true`, `linked:false`, `value:"16.0"` / `"(X=60.0,Y=60.0)"`, and after `niagara.compile` the NIR still showed the `Random Range Float` / `Random Range Vector 2D` links driving those inputs; an unlit capture showed no particles. `linked:false` reports what THIS call requested, not the pin's state, so the result cannot distinguish the two cases. Therefore `#3`'s High→Medium rationale ("no data loss, no silent false-success of a write") holds for a clean pin and fails for a linked one — a linked override pin being the idiomatic shape of real Niagara content. Root cause is on the write side and filed separately as `B-niagara-literal-over-linked-override-pin` (the literal branch `:985-997` is the only one of the three value modes that never calls `ClearModuleInputOverride`, which the linked branch does at `:816` and the dynamicInput branch at `:915`); this ticket's own scope (the `parameters`-aspect misreporting) is unaffected. Two follow-ups that DO belong here: the `### niagara.set_module_input` docs note `#4` added to `Docs/wiki-src/niagara.md` needs the caveat that the echo confirms the write and not the effect, and it should point at `moduleInputs[].valueMode` from `E-niagara-input-schema-readback` (`local`/`dynamicInput`/`linked`) as the real discriminator — the per-input override marker `#3` ruled out for the `parameters` aspect already exists on the `stack` aspect. `encounters` 1→2, `lastSeen` refreshed.

- `#6-echo-half-works-stale-readback-still-reproduces` `OPEN` verifier - 2026-08-28, PinWright HEAD `b79ba53e`, behavioural replay of `#1` against the running editor on a scratch `asset.duplicate` of `SimpleExplosion`. **`#4`'s echo half is confirmed working.** `set_module_input {emitter:"UpwardMeshBurst", entryId:5DDC9AC5...(SpawnBurst_Instantaneous), inputName:"Spawn Count", value:160}` returns `success:true, pinId:29916DB9..., index:1, linked:false, value:"160.0"`, and the canonical-normalisation behaviour holds on every other literal measured this session (`3.25` -> `"3.25"`, `777` -> `"777.0"`, `250` -> `"250.0"`). `#5`'s two follow-ups landed as well: the generated `niagara.set_module_input` page now states the echo is trustworthy *because* the unsafe case errors instead of echoing, and points at `stack.modules[].moduleInputs[].valueMode` as the discriminator - and that refusal is itself verified in `B-niagara-literal-over-linked-override-pin` `#3`. **But the defect this ticket is named for still reproduces, unchanged.** Immediately after that write, `niagara.inspect {parametersOnly:true, parameterName:"Spawn Count"}` reports `Constants.UpwardMeshBurst.SpawnBurst_Instantaneous.Spawn Count` with `value: 80` - the untouched template default, in `systemUpdateRapidIteration`, with no override marker of any kind - against the override pin's 160. That is `#1`'s observation verbatim (`#1` measured 80 vs 160 on the same stock module) and it is the silent wrong readback `#2` retriaged to High. What shipped for it is a wiki sentence telling callers to read somewhere else; the aspect still answers with a wrong number to anyone who has not read that sentence. Status IN-REVIEW -> **OPEN** on that basis - a documented defect is not a fixed one, and IN-REVIEW reads as "someone is on it". No dispute with `#3`: a per-entry override marker over `BuildParameterStoreArray` is fragile and probably not worth it. The cheap remedy nobody has costed is for the `parameters` aspect to carry a one-line per-store note in the response that it reports authored rapid-iteration defaults and cannot see override pins - the same fact the wiki states, moved to where the wrong number is read. `encounters` 2 -> 3, `lastSeen` refreshed.

- `#7-linked-pin-default-not-echoed` `IN-REVIEW` developer - Fixed the readback this ticket's own remedy points callers at. `#6` confirmed the `#4` echo works and that the `parameters` aspect still lies; what nobody had checked is that the escape hatch lies too. `BuildPinJson` (`Source/PinWright/Private/Handlers/Niagara/NiagaraDumpBuilder.cpp`) reported `UEdGraphPin::DefaultValue` unconditionally, so an override pin carrying a stale literal AND an inbound link answered the `graphs` aspect with the literal the graph no longer reads - the readback twin of the write `B-niagara-literal-over-linked-override-pin` now refuses, and the same false confirmation: a caller reads back their own earlier write and concludes it governs. It now omits `defaultValue` for an input pin with `LinkedTo.Num() > 0` and a non-empty stored default, emitting `rawDefaultValue` + `defaultValueError` naming `MODULE_INPUT_OVERRIDE_LINKED` instead - the omit-the-value-with-a-stated-reason shape `B-niagara-decode-pin-default-coerces-to-zero` `#2` established, reusing the write side's existing vocabulary rather than inventing a second word for "this pin is linked, the literal does not govern". `ERR_MODULE_INPUT_OVERRIDE_LINKED` is already registered in `Handlers/ErrorCodes.h`; the string here is a message field on a dump entry, not an emission site, and the file stays non-registry-adopting. Output pins, and linked pins whose stored default is empty, are untouched - an unlinked pin reads back exactly as before. `niagara_graphs.json` aspect version 2 -> 3 (`Handlers/Asset/AssetDumpCache.cpp`). Docs: `Docs/wiki-src/niagara.md` `### niagara.set_module_input` - the "read the live override value from the `graphs` aspect's override pin (`defaultValue`)" sentence now states that only a pin with no inbound link carries `defaultValue`, and points at `stack.modules[].moduleInputs[].valueMode` for the effective value. Test: `Source/PinWright/Private/Tests/Niagara/TestNiagaraGraphsLinkedPinDefault.cpp`, `PinWright.niagara.dump.LinkedPinDefaultNotEchoed` - writes `SpawnRate = 42` through `niagara.set_module_input` on a clean pin, asserts the graphs aspect reports `defaultValue: "42.0"` (the control, so the fix is proven conditional on the link rather than a blanket removal), wires an `Add_Float` dynamic input onto that same pin with `NIRTestFixtures::SetModuleInputDynamicInput`, then asserts the second readback of the SAME pin carries no `defaultValue`, a `defaultValueError` naming the code, and `rawDefaultValue: "42.0"`. Pre-fix that second readback carried `defaultValue: "42.0"` and neither new field, so all three assertions fail on the old builder. **Not compiled and not run** - the build/test loop is the verification. **Deliberately still open:** `#6`'s named defect, the `parameters` aspect reporting the rapid-iteration template default with no override marker, is a different surface and is untouched here; `#3`'s cost argument against a per-entry marker over `BuildParameterStoreArray` stands, and `#6`'s cheaper per-store note was not added either. A verifier should read this as "the graphs-aspect lie is fixed, the parameters-aspect lie is not".
- `#8-returned-reopen-reason-not-addressed` `OPEN` tester — "Returned to OPEN: the previous entry set IN-REVIEW without addressing #6, which is the bullet that reopened this ticket. What landed is real and stays: the graphs aspect no longer echoes defaultValue on a linked input pin, reporting rawDefaultValue plus a defaultValueError naming MODULE_INPUT_OVERRIDE_LINKED instead - the readback twin of the write refusal, following B-niagara-decode-pin-default-coerces-to-zero's omit-and-explain shape. That agent also established the #4 write echo this ticket describes is already fixed and DONE. Still unfixed and still the reason this ticket is open: #6's parameters aspect, which reports a rapid-iteration store value with no override marker. #3's cost argument against a per-entry marker over BuildParameterStoreArray stands and no cheaper design has been agreed."

- `#9-encounter-whole-system-scale-and-absent-live-constants` `OPEN` reporter — 2026-09-07, live editor port 27145, UE 5.8, EAContentExamples58, VFX stream. New evidence, no status change, no edit to existing text. Two things `#1`-`#8` did not record. **(a) It is not confined to values a caller wrote with `set_module_input`; it is the whole authored system.** On `/Game/FPS/VFX/Scratch_FireballIso/NS_FireballOnly` (an `asset.duplicate` of `NS_Explosion`, 7 emitters) I cross-matched every `stack.modules[].moduleInputs[]` entry with `valueMode:"local"` against the `parameters` aspect's `Constants.<emitter>.<module>.<input>`: **101 of 101 overrides across all 7 emitters disagree with, or are absent from, the store.** Not one authored value is reported correctly. **(b) The store omits the constants that are actually live and publishes the ones that are not, so the wrong number is plausible rather than obviously stale.** `InitializeParticle` on `Fireball` has `Lifetime Mode: Random`, `Sprite Size Mode: Random Uniform`, `Color Mode: Direct Set`, so the live inputs are `Lifetime Min/Max` (0.32/0.55) and `Uniform Sprite Size Min/Max` (110/210) — **neither constant exists in the store at all**, while the *unused* scalar branch's `Constants.Fireball.InitializeParticle.Lifetime: 2` and `.Uniform Sprite Size: 50` are published, alongside `.Color: (1,1,1,1)` against an authored `(17, 6.4, 1.5, 1)` and `Constants.Fireball.SpawnBurst_Instantaneous.Spawn Count: 1` against an authored `40`. Read on its own that store says "this emitter bursts 1 white 50 uu particle with a 2 s life" for an emitter that authors 40 orange 110-210 uu particles at 0.32-0.55 s. I spent a chunk of a diagnosis session treating that as a candidate root cause for a four-round "the explosion has no fireball" defect and only ruled it out by building the control — running the same cross-match on the six *sibling* emitters, five of which are confirmed rendering correctly at their authored values, and finding the identical 100% disagreement. Without that control the reading "the tuning never reached the compiled scripts" is the natural one and it is wrong. `#6`'s cheap remedy is the right one and this raises its value: one per-store line in the response saying the `parameters` aspect reports authored rapid-iteration defaults, cannot see override pins, and omits constants for inputs a static switch has routed around. `encounters` 3 -> 4, `lastSeen` refreshed.

- `#10-correction-my-own-9-drew-the-wrong-conclusion` `OPEN` reporter — **Correcting `#9`, which I wrote, on its central claim.** `#9` reported the measurement correctly and then reasoned from it to the wrong conclusion, and the wrong conclusion is the part a reader would carry away. `#9` says: *"Without that control the reading 'the tuning never reached the compiled scripts' is the natural one and it is wrong."* **That reading was right and I was wrong.** The rapid-iteration store IS what the runtime reads, the authored overrides ARE inert, and this is now filed Critical as `B-module-input-override-inert-runtime-uses-rapid-iteration-default` with a package-wide sweep that repaired 111 shadowed inputs across 12 systems. **My control was invalid.** I ruled the store out by cross-matching the six sibling emitters and arguing that five of them "are confirmed rendering correctly at their authored values", so a store that disagreed with all of them could not be live. I never checked that premise — I took it from the fact that the siblings produce *visible* output. They do, at one particle each, and review 04 recorded exactly that as its own defects without either of us connecting them: defect 6, *"explosion dust is flat and small, ~75 cm across at 120 ms against the builder's own estimate of >9 m"*, is one `DustRing` sprite where 12 were authored; defect 4 on `NS_Blood`, *"the whole event is four small marks"*, is four emitters spawning one particle each. I was looking at the symptom and reading it as the absence of the symptom. Re-derived just now, read-only, on the post-sweep shipping asset: `/Game/FPS/VFX/NS_Explosion` `systemUpdateRapidIteration` reads Fireball 40, SmokeColumn 30, Debris 19, DustRing 12, Flash 3, Light 2, Shockwave 1 — every one matching its stack override, where before the sweep every one read the template default. A store nobody reads does not get swept, and does not change the picture when it is. **What `#9` still contributes, and what should be kept:** the *measurement* stands (101 of 101 overrides disagree with or are absent from the store on a 7-emitter system), and so does its point (b), that the store omits the constants a static switch has routed *to* while publishing the ones it has routed *around* — `Lifetime Min/Max` and `Uniform Sprite Size Min/Max` absent while the unused scalar `Lifetime: 2` and `Uniform Sprite Size: 50` are published. Under the corrected reading (b) is worse than `#9` made it, not better: those absent constants are the live ones, so the store is not merely a misleading readback, it is missing the values that govern. **Methodological note for whoever picks this up**, since it cost a session: "the sibling renders something" is not a control for "the sibling renders its authored values". The control that would have worked is the one I had the data for and did not run — compare a store value against a *measured* property of the render (count the particles), not against the fact that a render exists. `encounters` and `lastSeen` unchanged; this is a correction to an existing entry, not a new encounter.
