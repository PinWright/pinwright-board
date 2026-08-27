---
id: E-niagara-inspect-params-stale-after-override
title: "niagara.set_module_input does not echo the value it wrote, so a set-then-verify loop must re-inspect — and the natural params-aspect readback shows the stale rapid-iteration template default (the override lives on the graph pin), making the verify look like a no-op"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [niagara, niagara-inspect, set-module-input, readback, override-pin, rapid-iteration, misleading]
encounters: 2
lastSeen: 2026-08-27T18:56:59+05:00
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
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed via `mcp__editor-automation__call` on a fresh `asset.duplicate` of `SimpleExplosion` → `/Game/FX/NS_BigExplosion_Replay`. `niagara.set_module_input` (UpwardMeshBurst SpawnBurst_Instantaneous "Spawn Count" = 160) returned `success:true, pinId:74553D62…, linked:false`. A subsequent `niagara.inspect includeProperties:true` reported the matching rapid-iteration param `Constants.UpwardMeshBurst.SpawnBurst_Instantaneous.Spawn Count` with `scope:"systemUpdateRapidIteration"` and `value:80` (the template default, unchanged, with no override marker), while `niagara.inspect includeGraphs:true` reported the override pin `74553D62…` `SpawnBurst_Instantaneous.Spawn Count` with `defaultValue:"160.0"` — same input, two contradictory values. Seed method was `niagara.set_module_input` (which worked correctly); the misreporting culprit is `niagara.inspect`'s parameters aspect. Outcome ergo. Ripgrep across OPEN/closed found no existing ticket on the params aspect showing the stale module-input default after an override (the two neighbor tickets are about response-size spill and runtime component overrides respectively — see "Distinct from").
- `#2-retriage` `OPEN` triage — Low→High: inspect params aspect reports the stale template default after a successful override so a params-only verify reads as a no-op, silent wrong-readback, niche path keeps it from Critical.
- `#3-reword-to-value-echo` `OPEN` developer — Reworded toward the narrower, higher-leverage scope (per the adversarial validity lens). The fat remedy in `#1`/`#2` (add an `overridden`/`effectiveValue` marker to every rapid-iteration entry in the inspect `parameters` aspect) is over-scoped: `BuildParameterStoreArray` (NiagaraDumpBuilder.cpp:467-492) iterates the whole store with no module-node context, so per-entry override detection would need to map each `Constants.<emitter>.<module>.<input>` back to its module node and walk its override pin — large/fragile for a niche verify path. Re-scoped Fix to the cheap remedy the ticket itself listed: echo the value `set_module_input` wrote. Severity High→Medium (readback omits a field, doable via the graphs aspect / now via the echoed result — no data loss, no silent false-success of a write). Status OPEN→IN-REVIEW; see `#4`.
- `#4-echo-written-value` `IN-REVIEW` developer — `niagara.set_module_input` now echoes the value it wrote so the set-then-verify loop confirms from the write result without a second (lying) `parameters`-aspect inspect. `ApplyModuleMutation` (NiagaraEditHandler.cpp) gained an optional `OutWrittenValue` out-param set in the literal SetModuleInput branch to the canonical pin-default string actually landed on the override pin (the same `DefaultValue` from `JsonValueToPinDefaultString`/`InferNiagaraInputType` that `TrySetDefaultValue` writes, e.g. `"160.0"`); the `set_module_input` handler sets it as `value` on the success result (literal path only — the linked path already reports `linked`/`parameter`/`parameterType`). Docs: `docs/wiki-src/niagara.md` gained a `### niagara.set_module_input` note (override-pin write, result echoes `value`, `parameters` aspect shows the stale rapid-iteration template default, live override reads from the `graphs` aspect). Files: `Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp`, `docs/wiki-src/niagara.md`. Test: `Source/PinWright/Private/Tests/Niagara/TestNiagaraSetModuleInput.cpp::FNiagaraSetModuleInputEchoesWrittenValueTest` sets `Spawn Rate=160` on a real InitializeParticle/SpawnRate module via the dispatcher and asserts the result's `value` echoes the written number (it would be absent/empty if the echo were reverted), with a 2.5-spelled case to prove the canonical normalization is echoed rather than the raw request.
- `#5-encounter-echo-unsafe-on-linked-pin` `IN-REVIEW` reporter — Additional evidence, no status change and no edit to existing text. `#1`'s repro assumes a clean override pin, where the write lands and only the `parameters` aspect misreports it. When the override pin has an inbound link (a dynamic-input chain or a `User.*` binding) the literal write is a complete no-op — `TrySetDefaultValue` stamps `DefaultValue` and a pin with `LinkedTo` reads the link — and the `#4` echo (`*OutWrittenValue = DefaultValue`, `Handlers/Niagara/NiagaraEditHandler.cpp:997`) confirms it anyway, because it echoes the byte written rather than the value the graph uses. Observed 2026-08-27 on UE 5.8 in the EAContentExamples58 checkout (PinWright `8e76cad5`): on `/Game/Atlantis/VFX/NS_Bubbles_Stream` emitter `Bubbles`, `set_module_input {Lifetime:16}` / `{Sprite Size:{x:60,y:60}}` returned `success:true`, `linked:false`, `value:"16.0"` / `"(X=60.0,Y=60.0)"`, and after `niagara.compile` the NIR still showed the `Random Range Float` / `Random Range Vector 2D` links driving those inputs; an unlit capture showed no particles. `linked:false` reports what THIS call requested, not the pin's state, so the result cannot distinguish the two cases. Therefore `#3`'s High→Medium rationale ("no data loss, no silent false-success of a write") holds for a clean pin and fails for a linked one — a linked override pin being the idiomatic shape of real Niagara content. Root cause is on the write side and filed separately as `B-niagara-literal-over-linked-override-pin` (the literal branch `:985-997` is the only one of the three value modes that never calls `ClearModuleInputOverride`, which the linked branch does at `:816` and the dynamicInput branch at `:915`); this ticket's own scope (the `parameters`-aspect misreporting) is unaffected. Two follow-ups that DO belong here: the `### niagara.set_module_input` docs note `#4` added to `Docs/wiki-src/niagara.md` needs the caveat that the echo confirms the write and not the effect, and it should point at `moduleInputs[].valueMode` from `E-niagara-input-schema-readback` (`local`/`dynamicInput`/`linked`) as the real discriminator — the per-input override marker `#3` ruled out for the `parameters` aspect already exists on the `stack` aspect. `encounters` 1→2, `lastSeen` refreshed.
