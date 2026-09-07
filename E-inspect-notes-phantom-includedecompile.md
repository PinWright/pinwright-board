---
id: E-inspect-notes-phantom-includedecompile
title: "blueprint.inspect wiki Notes tell the caller to pass `includeDecompile`, a parameter the same page's Parameters block omits and the handler hard-rejects with UNKNOWN_PARAMS"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, blueprint-inspect, wiki, docs, param-name, phantom-param]
encounters: 1
lastSeen: 2026-09-07T00:00:00Z
---

# The generated page contradicts itself: prose advertises a param the same page's schema does not list

`Saved/PinWright/wiki/blueprint.inspect.md` is a single generated page whose `**Parameters**`
block comes from the handler registry and whose `## Notes` prose comes from the `wiki-src`
overlay. The two halves disagree. The Notes tell the caller to pass `includeDecompile`; the
Parameters block does not list it; the dispatcher rejects it.

The wiki is the sanctioned discovery path — the MCP server instructions say *"Read the exact
method page before execution"* — so a caller who does exactly what they were told to do gets a
hard failure on a read-only verb.

## What was called

```
blueprint.inspect {assetPath: "/Game/FPS/Player/BP_FPSCharacter", includeDecompile: false, includeReferences: false}
```

## What came back — verbatim

```
[UNKNOWN_PARAMS] Unknown parameter(s) for 'blueprint.inspect': [includeDecompile]. Valid parameters: [assetPath, requestedPath, path, name, blueprintPath, blueprint_path, blueprintCandidates, candidates, includeReferences, includeScriptRefs, includeProperties]. Call 'blueprint.inspect' with no 'args' field to fetch its wiki page.
```

## What misled the caller — verbatim

`Saved/PinWright/wiki/blueprint.inspect.md`, `## Notes` section, line 25:

> Lightweight vs deep mode: pass `includeDecompile: false` for metadata only (no graphs, much
> faster) when you only need the variable / function / component shape. Use `includeReferences:
> true` to fold in the same data `blueprint.references` returns, and `includeScriptRefs: true` to
> surface script-level dependents. Use `includeProperties: true` when you need CDO property values
> with the same sparse inheritance-tagged shape as `asset.dump` `properties.json`.

The three other parameters named in that same sentence (`includeReferences`, `includeScriptRefs`,
`includeProperties`) are all real and all appear in the Parameters block eleven lines above it.
`includeDecompile` is the only one that does not exist — which is what makes the sentence
convincing rather than obviously stale.

The Parameters block on the same page lists exactly four entries and no `includeDecompile`:

```
- `assetPath` (`path`, required): Blueprint asset path Aliases: `requestedPath`, `path`, `name`, `blueprintPath`, `blueprint_path`. Typed aliases: `blueprintCandidates` (`array`), `candidates` (`array`).
- `includeReferences` (`bool`, optional): Include asset dependencies Default: `true`.
- `includeScriptRefs` (`bool`, optional): Include /Script/ references (filtered by default) Default: `false`.
- `includeProperties` (`bool`, optional): Include sparse CDO property diff with inheritance tags Default: `false`.
```

## What was expected

Either the parameter exists and the Parameters block lists it, or the Notes do not tell the caller
to pass it. A generated page that merges overlay prose with the handler registry should not be able
to ship a sentence recommending a parameter the registry has never heard of.

## Impact

One wasted round trip plus the time to re-derive a working call — pure friction on this occasion,
because `UNKNOWN_PARAMS` is a clear, self-diagnosing error that names the valid parameters. The
worse shape is a caller who reads the Notes and batches this parameter into a scripted sequence of
calls: a hard failure lands on a **read-only** verb, in a position where the caller had no reason
to expect one.

## Source of the prose (verified, read directly)

`Plugins/PinWright/Docs/wiki-src/blueprint.md:236` carries the offending sentence verbatim; it is
the only occurrence of `includeDecompile` anywhere under the plugin's `Docs/` or `Source/` trees.
That is the file the overlay merge reads.

**Root cause is not established** — no handler source was opened for this ticket, and no claim is
made here about *why* the parameter is absent from the registry (never implemented, implemented
under a different name, or removed and the prose left behind). Whichever it is, the fix is either
to delete/correct that sentence in `Docs/wiki-src/blueprint.md` so the prose stops advertising a
parameter that does not exist, or to implement `includeDecompile` on the handler so the Parameters
block grows the entry the prose already promises. The regression-test precedent for the docs-only
route is the wiki render-assert pattern used by `E-add-mapping-example-wrong-param` and
`E-montage-wiki-example-snake-case-params` (drive `WikiHandler::RenderPage("blueprint.inspect")`
and assert the rendered page does not contain `includeDecompile`).

## Severity

**Low** on the pure-friction band: a documentation defect on a read-only verb whose error message
names every valid parameter, so recovery is one call. Reach argues upward — the same Notes block
calls `blueprint.inspect` *"the default first call when you need to understand an unfamiliar
Blueprint"* — and the two closest precedents on this board were both retriaged Low→Medium, so a
triage bump would be defensible. Filed Low deliberately rather than pre-empting that.

## Related

- `B-blueprint-get-omits-inputkey-events` history entry `#4-verifier-hygiene-correction` — **not a
  duplicate, but the plugin side already knows.** That entry records a verifier rejecting a plugin
  test because its direct `blueprint.inspect` payload *"carried undeclared `includeDecompile`"*, and
  the parameter was removed from the test. So the fact that `includeDecompile` is undeclared was
  established inside the plugin repo, on this very verb — but nobody swept it out of the
  user-facing `wiki-src` prose, which still tells callers to pass it. That ticket's subject is
  `events[]` omitting `K2Node_InputKey` entry points; it says nothing about this doc defect.
- `E-add-mapping-example-wrong-param` (IN-REVIEW) — same class on `input.add_mapping`: the page's
  code Example used a key (`imcPath`) that the page's own Parameters list contradicts. Fixed by an
  overlay edit plus a `RenderPage` assert.
- `E-montage-wiki-example-snake-case-params` (IN-REVIEW) — same class on the montage chain,
  including a documented `end_time` that the handler hard-rejects with `UNKNOWN_PARAMS`. Its
  `#4-additional-notes-self-contradiction` entry is the closest analogue to this ticket: a **Notes
  paragraph** contradicting the same page's own summary and schema.
- `B-attach-render-target-phantom-param` — the runtime half of the "phantom parameter" family: a
  nonexistent parameter accepted silently rather than rejected. Here the rejection is correct and
  the docs are wrong.

## History
- `#1-filed` `OPEN` PLAYER-critic — Hit live via `mcp__pinwright__call`: `blueprint.inspect {assetPath: "/Game/FPS/Player/BP_FPSCharacter", includeDecompile: false, includeReferences: false}` returned `[UNKNOWN_PARAMS] Unknown parameter(s) for 'blueprint.inspect': [includeDecompile]. Valid parameters: [assetPath, requestedPath, path, name, blueprintPath, blueprint_path, blueprintCandidates, candidates, includeReferences, includeScriptRefs, includeProperties].` The parameter was passed **because the generated wiki page for that very verb instructs the caller to**: `Saved/PinWright/wiki/blueprint.inspect.md` `## Notes` (:25) says verbatim *"Lightweight vs deep mode: pass `includeDecompile: false` for metadata only (no graphs, much faster) when you only need the variable / function / component shape."* — while the same page's generated `**Parameters**` block (:11-14) lists only `assetPath`, `includeReferences`, `includeScriptRefs`, `includeProperties`. The three sibling params named in that same sentence are all real, which is what makes the phantom one convincing. Prose source verified by direct read: `Plugins/PinWright/Docs/wiki-src/blueprint.md:236`, the only occurrence of `includeDecompile` under the plugin's `Docs/` or `Source/` trees. **Root cause not established — no handler source opened, no claim made about why the registry lacks the param;** fix is presumably to delete/correct that sentence in the overlay, or to implement the parameter. Cost: one wasted round trip plus re-derivation; the broader risk is a caller batching the documented param into a scripted sequence and taking a hard failure on a read-only verb, when the wiki is the sanctioned discovery path (*"Read the exact method page before execution"*). Not a duplicate of `B-blueprint-get-omits-inputkey-events` (subject: `events[]` omitting InputKey nodes), though its `#4-verifier-hygiene-correction` records a verifier removing "undeclared `includeDecompile`" from a plugin **test** — the plugin side already knew the param does not exist, and the user-facing prose was never swept. Severity Low: docs friction on a read-only verb with a self-diagnosing error that names every valid parameter.
