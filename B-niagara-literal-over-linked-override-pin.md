---
id: B-niagara-literal-over-linked-override-pin
title: "niagara.set_module_input writes a literal onto an override pin that still has an inbound link, reports success, and echoes the literal back — while the graph keeps reading the link and the value never takes effect"
status: OPEN
severity: High
category: bug
tags: [niagara, set_module_input, override-pin, dynamic-input, linked-parameter, silent-noop, false-success, false-echo]
encounters: 1
lastSeen: 2026-08-27T18:56:59+05:00
---

# A literal written over a linked override pin is a silent no-op that reports success and echoes the value

`niagara.set_module_input` with a literal `value` on an input whose override pin
already has an **inbound link** returns `success: true`, `linked: false`, and echoes
the literal it claims to have written. Nothing changes. The link — a dynamic-input
chain, or a `User.*` parameter binding — keeps driving the pin, and the module keeps
reading the old value.

The echo is faithful to the byte that was written and unfaithful to the graph: the
literal lands on the pin's `DefaultValue`, and **a pin with `LinkedTo` set reads the
link, not the default**. So the response is technically accurate and practically a
lie.

## Root cause (guilty source lines)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp:985-997`,
the literal branch of `ApplyModuleMutation`'s `SetModuleInput` operation:

```cpp
            UEdGraphPin& OverridePin = FNiagaraStackGraphUtilities::GetOrCreateStackFunctionInputOverridePin(
                *Target.ModuleNode,
                AliasedInputHandle,
                InputType,
                FGuid(),
                FGuid());
            OverridePin.Modify();
            GetDefault<UEdGraphSchema_Niagara>()->TrySetDefaultValue(OverridePin, DefaultValue, true);
            OutNodeId = OverridePin.PinId.ToString();
            if (OutWrittenValue)
            {
                // Echo the canonical pin-default we wrote (see OutWrittenValue doc).
                *OutWrittenValue = DefaultValue;
            }
```

No `LinkedTo` inspection, no `BreakPinLinks`, no `ClearModuleInputOverride`. It
fetches (or creates) the override pin, stamps `DefaultValue`, and echoes it at
`:997` regardless of whether anything is wired into that pin.

**The two sibling value modes on the very same operation both clear the prior
override first, with comments saying exactly why.** This is what makes the omission
an oversight rather than a design:

- **linked-parameter path**, `:812-816`:

  ```cpp
                  // SetLinkedParameterValueForFunctionInput checkf()s that the override pin has no
                  // existing link, so clear any prior override (literal default, dynamic-input chain,
                  // or stale link) before creating the fresh pin to wire the parameter read into.
                  NiagaraResetModuleInput::ClearModuleInputOverride(*Target.ModuleNode, AliasedInputHandle, *Target.Graph);
  ```

- **dynamicInput path**, `:911-915`:

  ```cpp
                  // Clear any prior override (literal default, dynamic-input chain, or stale link)
                  // before creating the fresh override pin the dynamic-input node wires into — the
                  // same precondition the linked-parameter path observes above.
                  NiagaraResetModuleInput::ClearModuleInputOverride(*Target.ModuleNode, AliasedInputHandle, *Target.Graph);
  ```

Three value modes; two clear the pin; the literal one does not. Board-side
confirmation of the same asymmetry from the sessions that built those paths:
`F-niagara-link-module-input-to-parameter` `#2-implementation` ("clear any prior
override (`NiagaraResetModuleInput::RemoveOverridePinAndChainedNodes`)") and
`F-niagara-dynamic-input-authoring` `#2-dynamic-input-assign` ("clear any prior
override (`NiagaraResetModuleInput::ClearModuleInputOverride`)").

The literal branch also picks up its `InputType` from the pre-existing linked pin
(`:970-973`, the `ExistingPin` case), so it *sees* the pin it is about to fail to
overwrite.

## Verbatim repro

On `/Game/Atlantis/VFX/NS_Bubbles_Stream`, emitter `Bubbles`, ParticleSpawn module
`InitializeParticle` (`entryId 601B69A048DB6BF934AEE58DFF6C7378`), whose `Lifetime`
is driven by a `RandomRangeFloat` dynamic input and whose `Sprite Size` is driven by
a `RandomRangeVector2D`:

```
niagara.set_module_input {inputName: "Lifetime",    value: 16}
  -> success: true, linked: false, value: "16.0"
niagara.set_module_input {inputName: "Sprite Size", value: {x: 60, y: 60}}
  -> success: true, linked: false, value: "(X=60.0,Y=60.0)"
niagara.compile
niagara.decompile_nir
```

After both writes **and a compile**, `decompile_nir` still reports the dynamic-input
links intact in `graph ParticleSpawnInterpolated`:

```
link `Random Range Float`.UniformRangedFloat  -> `Map Set`_2.`InitializeParticle.Lifetime`
link `Random Range Vector 2D`.NewOutput001    -> `Map Set`_2.`InitializeParticle.Sprite Size`
link `Float from Curve`.Value                 -> `Map Set`_10.`ScaleColor.Scale Alpha`
```

A clean literal has **no** inbound `link` for that input and appears only as
`set $InitializeParticle.Lifetime = 16.0`.

Confirmed visually as well: with `User.SpawnRate` at 400 and `Sprite Size` "set" to
60 cm, an unlit level capture 900 uu from the emitter shows **no particles at all** —
the sprites are still the dynamic input's 0–1 cm.

## Scope: any inbound link, not dynamic inputs specifically

The trigger is "the override pin has **any** inbound link". A `User.*` parameter
binding fails identically: a literal written over `SpawnRate.SpawnRate`
(`valueMode: "linked"`) was ignored while `Map Get User.SpawnRate` kept driving the
pin. Do not scope a fix to dynamic inputs.

## How to tell an affected pin from a clean one

- **Authoritative** — `niagara.decompile_nir`, then read the `graph <Stage>` block:
  an affected pin has an inbound `link` naming the source; a clean literal has none.
  This is the reading that settled it here.
- **Cheap screen** — `niagara.inspect {includeStack:true}` →
  `stack.modules[].moduleInputs[].valueMode`. A genuine literal reads `local`; a pin
  driven by a dynamic input reads `dynamicInput`; a parameter binding reads `linked`.
  That `valueMode` field is supplied by `E-niagara-input-schema-readback` (IN-REVIEW)
  — **it is the verification tool this fix needs**, and a fix should assert on it.

Honesty bound on the readback claim: `valueMode` was observed as `dynamicInput`
**before** the writes and as `local` on the same two inputs **after the module was
removed and re-added**. The intermediate state — that `valueMode` still reads
`dynamicInput` immediately after a "successful" literal write — is **INFERRED from
the NIR check above, not observed directly**. A fix should confirm it.

## This falsifies a shipped remedy, and its severity downgrade

`E-niagara-inspect-params-stale-after-override` (IN-REVIEW, Medium) shipped the very
echo this ticket proves is a false confirmation. Its `#4-echo-written-value`
implemented `OutWrittenValue` (the `:994-997` block above) so that "the set-then-verify
loop confirms from the write result without re-inspecting", and its `#3-reword-to-value-echo`
downgraded severity High → Medium on the reasoning:

> "no data loss, **no silent false-success of a write**"

That reasoning does not hold when the override pin is linked. In exactly that case
the write **is** a silent false-success, and the echo is what makes it convincing:
a caller following the `set_module_input` wiki's own advice ("Confirm a literal edit
from this result directly — no second inspect needed") gets a confirmation of a value
the graph never uses. The echo is safe only for an unlinked pin, and the result
carries nothing that distinguishes the two cases — `linked: false` refers to whether
*this call* requested a link, not to the pin's state.

The docs note that shipped alongside it (`Docs/wiki-src/niagara.md`,
`### niagara.set_module_input`) needs the same caveat. An encounter note recording
this has been appended to `E-niagara-inspect-params-stale-after-override`.

## Impact

This is what made `NS_Bubbles_Stream` render nothing for two agents in a row:
`Lifetime` stayed 0–1 s and `Sprite Size` stayed 0–1 cm while every write said
success. It also compounds `F-niagara-dynamic-input-nested-inputs` and
`B-niagara-set-parameter-emitter-scope-unreachable` from a missing feature into a
wrong one — the documented escape hatch for an unauthorable dynamic-input sub-input
is "set a literal over the dynamic input instead", and that escape hatch does not
work while claiming it does. No crash; no effect on other agents sharing the editor.

## What it should do

On the literal path, if the override pin has an inbound link, either (a) break it
and remove the orphaned chain before writing `DefaultValue` — the same
`ClearModuleInputOverride` call the other two branches already make at `:816` and
`:915` — or (b) refuse with a typed error naming the dynamic input or parameter that
owns the pin. Echoing a literal the graph does not use is the worst of the three
options. If (a) is chosen, the echo becomes true again and no doc caveat is needed;
if (b), the wiki's "confirm from this result directly" advice becomes safe because
the unsafe case now errors.

## Workaround

`niagara.remove_module` + `niagara.add_module` + `niagara.move_module` back into
position, then set literals on the fresh node **before** any dynamic input is
attached. Do **not** reach for `niagara.reset_module_input`
(`B-niagara-reset-module-input-corrupts-stack`) or `niagara.clear_module_overrides`
— both walk the same override-chain removal code. Never attach a dynamic input to an
input you intend to set literally; today it is a one-way door.

## Distinct from related tickets

- `E-niagara-inspect-params-stale-after-override` (IN-REVIEW, Medium) is the
  **readback** ticket whose remedy this contradicts. It is about the `parameters`
  aspect showing a stale rapid-iteration default after a **successful** override.
  Here the override never lands at all, and its remedy (the value echo) is what
  hides that.
- `B-niagara-set-module-input-vec2` is the **fresh-pin type-inference** branch
  (`InferNiagaraInputType` on a pin that does not yet exist). This defect is on the
  opposite branch — the pin exists and is linked, and the type is taken from it
  correctly (`:970-973`). Not a duplicate.
- `E-niagara-input-schema-readback` (IN-REVIEW) is not a duplicate either: it
  **supplies** the `valueMode` discriminator that tells an affected pin from a clean
  one, and is cited above as this fix's verification tool.
- `B-niagara-reset-module-input-corrupts-stack` is the other defect on the same
  precondition (an input driven by a dynamic input), but on the `reset` verb and with
  the opposite failure mode — loud and unrecoverable rather than silent.
- `F-niagara-link-module-input-to-parameter` / `F-niagara-dynamic-input-authoring`
  built the two branches that DO clear the pin; cited as evidence, not defects.

severity rationale: impact=silent false-success on a normal path, made worse by an echo that actively confirms the value the graph does not use, so the caller trusts a write that never happened and builds on it x reach=`set_module_input` is the central Niagara authoring verb, and the precondition (an override pin driven by a dynamic input or a `User.*` binding) is the idiomatic way real Niagara content is parameterized -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at `8e76cad5` in this checkout. On `/Game/Atlantis/VFX/NS_Bubbles_Stream` emitter `Bubbles`, ParticleSpawn `InitializeParticle` (`entryId 601B69A048DB6BF934AEE58DFF6C7378`), `set_module_input {Lifetime: 16}` and `{Sprite Size: {x:60,y:60}}` each returned `success:true`, `linked:false` and echoed `"16.0"` / `"(X=60.0,Y=60.0)"`; after both writes and a `niagara.compile`, `niagara.decompile_nir` still showed `link \`Random Range Float\`.UniformRangedFloat -> \`Map Set\`_2.\`InitializeParticle.Lifetime\`` and the Vector2D equivalent, and an unlit capture 900 uu out showed no particles (sprites still 0–1 cm). Source-confirmed in this tree: literal branch `NiagaraEditHandler.cpp:985-997` calls `GetOrCreateStackFunctionInputOverridePin` → `OverridePin.Modify()` → `TrySetDefaultValue(...)` → echoes `DefaultValue` at `:997`, with **no** `LinkedTo` inspection, `BreakPinLinks` or `ClearModuleInputOverride`; the linked-parameter branch calls `ClearModuleInputOverride` at `:816` and the dynamicInput branch at `:915`, each with a comment stating the precondition — so the literal branch is the only one of the three that skips it. Scope is "override pin has ANY inbound link", not dynamic inputs: a literal over `SpawnRate.SpawnRate` (`valueMode:"linked"`, a `User.*` binding) was ignored the same way. Honesty bound kept from the log: the "post-write `valueMode` still reads `dynamicInput`" claim is INFERRED from the NIR reading, not observed directly (`dynamicInput` was observed before the writes, `local` after a remove+re-add). Contradiction recorded: `E-niagara-inspect-params-stale-after-override` (IN-REVIEW) shipped this echo in `#4-echo-written-value` as a self-confirming write result and downgraded itself High→Medium in `#3` on the reasoning "no silent false-success of a write" — which fails precisely when the pin is linked; a dated encounter section and History entry were appended there. Not deduped against `B-niagara-set-module-input-vec2` (fresh-pin type inference, opposite branch) or `E-niagara-input-schema-readback` (supplies the `valueMode` discriminator this fix should assert on).
