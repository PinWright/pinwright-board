---
id: F-niagara-dynamic-input-nested-inputs
title: "niagara.set_module_input: recursive nested-input authoring on an assigned dynamic input"
status: OPEN
severity: Low
category: feature
tags: [niagara, authoring, dynamic-input, parity-ue58]
encounters: 2
lastSeen: 2026-08-27T18:56:59+05:00
---

# niagara.set_module_input: recursive nested-input authoring on an assigned dynamic input

Split out from **F-niagara-dynamic-input-authoring** (which delivered assigning a dynamic input to a module input via `{ dynamicInput: "<path>" }`). That leaves the assigned dynamic input at its in-script default inputs. Many real dynamic inputs must have their OWN inputs set to be useful (a Curve-over-Life needs its curve; an Add/Multiply needs its addend/factor). This ticket adds recursive nested-input authoring:

```
value: { "dynamicInput": "<ScriptAssetPath>", "inputs": { "<inputName>": <literal | { dynamicInput, inputs }> } }
```

so a chain can be authored in one call.

Implementation note / why separate: setting a nested input requires the assigned dynamic-input node's authorable stack inputs (their names AND types) to resolve `inputName` and type the nested override pin. `NiagaraEdit::EnumerateScriptInputs` does NOT provide these — it surfaces only the script's parameter-map input node (observed: Add_Float enumerates a single `NewInput` of type `NiagaraParameterMap`, not its value inputs). The correct source is `FNiagaraStackGraphUtilities::GetStackFunctionInputs` (NIAGARAEDITOR_API) which needs an `FCompileConstantResolver` built from the target system/emitter + script usage. That machinery (and a nested-input regression test that discovers a real input name via the same resolver) is the work this ticket tracks.

Acceptance: assign a dynamic input with a nested input set (e.g. `{ dynamicInput: "Add_Float", inputs: { "<A>": 42.0 } }`); the call succeeds, the module input override pin is driven by the dynamic-input node, and the named nested input carries the set literal (or a further nested dynamic-input node). Depth-guard runaway/cyclic chains.

## Encounter 2026-08-27 — hit in real authoring; adds a failing call, a cheaper implementation path, and a severity argument

Observed 2026-08-27 on UE 5.8 in the EAContentExamples58 checkout, PinWright at
`8e76cad5`, while building the Atlantis level's VFX. Three things this ticket did
not previously record.

**1. The observed failure has an error code, and this ticket records no failing call
at all.** Assigning `{ dynamicInput: ... }` creates the node as designed, but the
node's own inputs never become authorable pins, and `niagara.set_module_input`
aimed at the **dynamic-input node's own `entryId`** — the natural thing to try next —
returns `INVALID_STACK`. That is not a "feature absent" response; it is the same
error a caller gets for a malformed module target, so a caller cannot tell the
capability is missing from the response. Worth wording the eventual refusal (or the
docs) so the two are distinguishable. The source acknowledgement this ticket is
named in is confirmed still present at
`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp:1495-1499`.

**2. A cheaper implementation path this ticket never considered.** The nested values
**do** land in the emitter rapid-iteration store, addressable as
`Constants.<Emitter>.<FunctionName>.<Input>`. So the feature is reachable today
through `niagara.set_parameter` with an emitter rapid-iteration scope — **no**
`GetStackFunctionInputs` and **no** `FCompileConstantResolver` machinery required.
That route is blocked only by `B-niagara-set-parameter-emitter-scope-unreachable`
(the emitter scopes demand an `emitter` argument the verb never declares, so they
answer `EMITTER_REQUIRED` without it and `UNKNOWN_PARAMS` with it). Fixing that
one-line schema gap may deliver most of this ticket's user-visible value for a
fraction of the scoped work. It would not cover *name/type discovery* of the nested
inputs — the caller would still have to know the input's name — but it does cover
setting them.

**3. The `Low` severity looks under-rated — recording the argument, not changing the
rating.** `#1` set `Low` on scheduling grounds ("tracked separately at lower
priority"), not on impact. In practice `RandomRangeFloat` and `RandomRangeVector2D`
are stuck at their 0–1 script defaults, which means **per-particle random ranges are
unavailable at all** — a basic requirement of almost any particle effect (this
session had to drive variation from `ScaleSpriteSizeBySpeed` / `ScaleColorBySpeed`
instead). And the documented escape hatch — "set a literal over the dynamic input
instead" — does not work either: `B-niagara-literal-over-linked-override-pin`
(filed from this session) shows that write is a silent no-op that reports success
and echoes the literal back. So the workaround that kept this at `Low` is gone. By
the board rubric this reads as a hard blocker with no workaround on a common path.
Leaving `severity` untouched — a reporter recommends, the ticket's author or a
triager decides.

## History
- `#1-split-from-dynamic-input` `OPEN` reporter — Split from F-niagara-dynamic-input-authoring during implementation: the flat assignment shipped, but recursive nested `inputs` needs resolver-based stack-input enumeration (`GetStackFunctionInputs` + `FCompileConstantResolver`) that `EnumerateScriptInputs` cannot supply, plus its own test, so it is tracked separately at lower priority.
- `#2-encounter-error-code-cheaper-path-severity` `OPEN` reporter — Additional evidence from real authoring (Atlantis level VFX; map as forcing function, see host `CLAUDE.md`), 2026-08-27, UE 5.8, PinWright at `8e76cad5` in the EAContentExamples58 checkout. Three new things over `#1`: (a) the observed failure now has a code — `set_module_input` against the dynamic-input node's own `entryId` returns `INVALID_STACK`, indistinguishable from a malformed module target, where this ticket previously recorded no failing call at all; (b) a cheaper implementation path — the nested values DO land in the emitter rapid-iteration store as `Constants.<Emitter>.<FunctionName>.<Input>`, so the feature is reachable via `niagara.set_parameter` with a rapid-iteration scope with no `GetStackFunctionInputs` / `FCompileConstantResolver` machinery, blocked today only by `B-niagara-set-parameter-emitter-scope-unreachable`'s missing `emitter` declaration (it would still not solve nested-input name/type discovery); (c) a severity argument — `#1`'s `Low` was set on scheduling grounds, but `RandomRangeFloat`/`RandomRangeVector2D` stuck at 0–1 defaults means per-particle random ranges are unavailable at all, and the documented escape hatch ("set a literal over the dynamic input") is itself a silent no-op per `B-niagara-literal-over-linked-override-pin`, so the workaround that justified `Low` does not exist. Recommending a re-rate; `severity` deliberately left unchanged. Source acknowledgement re-verified in this tree at `NiagaraEditHandler.cpp:1495-1499`. `encounters` 1→2, `lastSeen` refreshed.
