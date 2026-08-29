---
id: E-foliage-remove-mode-is-output-only
title: "foliage.remove's own page tells callers to check `mode`, the response emits `mode`, and passing `mode` back is rejected UNKNOWN_PARAMS — the verb has promoted an output field to the caller-facing name for its scope while the input still spells that scope as a boolean plus an optional path"
status: OPEN
severity: Low
category: ergonomic
tags: [foliage, remove, params, unknown-params, response-echo, input-output-drift, naming, docs, precedence]
encounters: 1
lastSeen: 2026-08-29T18:00:00+05:00
---

# The response's vocabulary is better than the request's, and only the response is allowed to use it

`foliage.remove` registers exactly two parameters
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:592-596`):

```cpp
RPC_PARAM_OPT("foliageTypePath", "string", ...),
RPC_PARAM_OPT("removeAll",       "boolean", ...)
```

and writes a third name into its response, derived from the local flag rather than echoed from
input:

```cpp
Resp->SetStringField(TEXT("mode"), bRemoveAll ? TEXT("all") : TEXT("type"));
```
`FoliageHandler.cpp:694`

Anything not in the registered list is refused by the dispatcher's generic gate — allowlist built at
`Plugins/PinWright/Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:128-132`, rejection at `:159`
(`Ctx.SendError(TEXT("UNKNOWN_PARAMS"), ...)`). So `{"mode": "type"}` is refused by the same call
that will answer `"mode": "type"`.

## Why this is more than a name

The interesting part is not the round-trip failure. It is that the plugin has already decided `mode`
is the right vocabulary and has told callers to rely on it. The generated page
(`Saved/PinWright/wiki/foliage.remove.md:21`) says, on the ambiguous input:

> **`removeAll` wins over a co-supplied `foliageTypePath`.** … The response echoes the
> interpretation as `mode` (`"all"` when a wholesale wipe was applied, `"type"` for a scoped
> removal) so the over-broad clear is never silent — **check `mode` if you co-supply both.**

and `:25` lists it as a documented success field. That paragraph is the landed fix from
`E-foliage-remove-silent-edge-inputs` (IN-REVIEW, Medium) and it is a good one: the over-broad wipe
is no longer silent. But look at the shape it settles on — the caller states the scope in two
loosely-coupled parameters that can disagree, the handler resolves the disagreement by a documented
precedence rule, and the caller is instructed to read an output field afterwards to find out which
operation actually ran.

**Accepting `mode` as an input makes the disagreement inexpressible.** `mode:"all"` and
`mode:"type"` are mutually exclusive by construction, so there is no precedence rule to document, no
echo to check, and no window in which a caller believes they scoped a removal and wiped the level.
That converts a *verify-afterwards* contract into a *cannot-be-stated-wrong* one, which is the
stronger form and the one the rest of this board keeps asking for.

## Ask

Accept `mode` as an optional input on `foliage.remove`, valued `"all"` | `"type"`, alongside the
existing two parameters:

- `mode:"all"` — equivalent to `removeAll:true`; a co-supplied `foliageTypePath` is an
  `INVALID_ARGUMENT` rather than a silently-ignored value, because the caller has now said two
  contradictory things explicitly instead of ambiguously.
- `mode:"type"` — requires `foliageTypePath`; missing it is `INVALID_ARGUMENT`, which is what
  omitting both already produces (`foliage.remove.md:23`).
- Both existing parameters keep working unchanged. This is additive; nothing that works today stops.
- The response keeps emitting `mode` exactly as it does now, so a caller can echo a response
  straight back into a repeat call — which is the thing that fails today.

If that is judged too much design for a Low ticket, the minimum honest version is one sentence on
`Docs/wiki-src/foliage.md` saying `mode` is an **output-only** field and naming
`removeAll` as its input counterpart, so the page that tells callers to check `mode` also tells them
they cannot send it.

## Filed separately from `E-foliage-remove-silent-edge-inputs`, deliberately

That ticket (IN-REVIEW, Medium) owns the honesty of the responses on edge inputs, its fix has landed
in the handler and the page, and it is awaiting verification. Appending an API-shape request to it
would widen a ticket a tester is about to close and would make the verification ambiguous. This is
the follow-on its own remedy suggests: having introduced `mode` as the field that disambiguates
scope, the natural next step is to let the caller say it.

Also distinct from `B-foliage-remove-empties-ledger-not-component` (OPEN, **Critical**), which is
the same verb's real defect — the removal never touches the component. Nothing in this ticket should
be worked before that one; it is filed apart precisely so a Low naming item cannot dilute a Critical
data-loss item's picker ranking. See that ticket's `#2` for the split note.

## Family

Input/output vocabulary drift on the same verb, all `E-`, all Low-to-Medium:
`E-asset-path-vs-assetpath-list-drift`, `E-add-variable-name-vs-variablename`,
`E-actor-verbs-reject-actorpath-slot`. This instance is the sharper variant because the drifting
name is not merely an alias the caller might guess — it is a name the plugin's own documentation
instructs the caller to read.

## Severity

**Low.** Impact class is the rubric's Low band, verbatim: *"pure friction. Docs, discoverability,
naming"*. Nothing is wrong, nothing is silent, and the refusal is loud and correctly coded
(`UNKNOWN_PARAMS`, from the generic gate at `RpcDispatcher.cpp:159`, which is working exactly as
designed). A caller who reads the parameter list sends `removeAll` and never notices.

**Considered and rejected: Medium.** The argument for it is that `mode` as an input would eliminate
a class of over-broad wipe by construction, and an over-broad foliage wipe is not friction. It is
rejected because that class is already handled — `E-foliage-remove-silent-edge-inputs`' fix
documents the precedence and makes the wipe visible in the response, so what remains is a better
shape, not an open hazard. Filing the improvement at the severity of the hazard it would have
prevented would double-count a fix that has landed.

**Reach modifier declined in both directions.** `foliage.remove` is not an every-session verb, so no
bump up. No bump down either: this is not an edge path within the verb — `mode` appears in every
success response the verb produces, and the page names it in three places.

## History
- `#1-mode-emitted-never-accepted` `OPEN` reporter — Observed during the look-dev polish pass over
  `PW_VegetationTest` (`Docs/map/vegetation-polish.md` § 5.1): `foliage.remove` rejected `mode` as
  `UNKNOWN_PARAMS` on a call whose own response carried `"mode":"type"`. Re-derived at HEAD:
  the verb registers only `foliageTypePath` and `removeAll` (`FoliageHandler.cpp:592-596`), writes
  `mode` explicitly from the local `bRemoveAll` at `:694` — so it is output-only and not a
  request echo — and the dispatcher's generic allowlist gate refuses anything else
  (`RpcDispatcher.cpp:128-132`, error at `:159`). The sharp part is not the round-trip: the shipped
  page already elevates `mode` to the caller-facing name for the operation's scope and instructs
  callers to rely on it — `Saved/PinWright/wiki/foliage.remove.md:21` (*"check `mode` if you
  co-supply both"*) and `:25` — which is the landed remedy from `E-foliage-remove-silent-edge-inputs`
  (IN-REVIEW). Asked for: accept `mode:"all"|"type"` as an optional input, which makes the
  contradictory both-supplied combination inexpressible instead of resolved-by-documented-precedence
  and checked-afterwards, and lets a caller echo a response straight back; existing parameters
  unchanged and additive. Minimum version if that is too much design for a Low: one line on
  `Docs/wiki-src/foliage.md` marking `mode` output-only and naming `removeAll` as its input
  counterpart. Filed separately from `E-foliage-remove-silent-edge-inputs` on purpose — that ticket
  is IN-REVIEW awaiting a tester and widening it would blur the verification — and separately from
  `B-foliage-remove-empties-ledger-not-component` (Critical) so a naming item cannot dilute the
  picker ranking of the same verb's data-loss defect. Rated **Low** (pure naming friction; the
  refusal is loud and the gate is behaving correctly); **Medium considered and rejected**, because
  the over-broad-wipe hazard the input form would eliminate by construction is already handled by
  the landed precedence fix, and rating the improvement at the hazard's severity would double-count
  it. Reach declined both ways: not an every-session verb, but `mode` is in every success response
  the verb emits, so not an edge path within it either.
