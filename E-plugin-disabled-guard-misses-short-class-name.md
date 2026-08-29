---
id: E-plugin-disabled-guard-misses-short-class-name
title: "The optional-plugin guard early-returns on anything without a `/Script/` prefix, so `graphClass: \"ProceduralVegetationGraph\"` — a short name the plugin's own resolver accepts and its own schema advertises — answers CLASS_NOT_FOUND instead of PLUGIN_DISABLED"
status: OPEN
severity: Low
category: ergonomic
tags: [pcg, create_graph, add_node, plugin-disabled, class-not-found, short-class-name, resolveuclass, optional-engine-plugin, procedural-vegetation, error-code, test-gap]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The resolver accepts a form the guard cannot see

`pcg.create_graph` with `graphClass: "ProceduralVegetationGraph"` answers `CLASS_NOT_FOUND`
("Could not resolve a concrete UPCGGraph subclass: ProceduralVegetationGraph"). The same call with
`graphClass: "/Script/ProceduralVegetation.ProceduralVegetationGraph"` answers `PLUGIN_DISABLED`,
naming the plugin, its experimental status and the four plugins it drags in. Two spellings of one
class, two different diagnoses, and the wrong one is the one that tells the caller to go hunt a typo
in a name that is correct.

`pcg.add_node` has the same split on `nodeClass`.

## Cause

`PinWrightPCG::SendPluginDisabledForScriptPath` refuses to look at anything that is not a full
script path. `Source/PinWrightPCG/Private/Handlers/PCG/PCGHandlerHelpers.h:105-111`, the first
statement after the function opens:

```cpp
inline bool SendPluginDisabledForScriptPath(FHandlerContext& Ctx, const FString& ScriptPath)   // :105
{
    const FString ScriptPrefix(TEXT("/Script/"));                                              // :107
    if (!ScriptPath.StartsWith(ScriptPrefix))                                                  // :108
    {
        return false;                                                                          // :110
    }
```

It needs the prefix because the module name is parsed out of the path — `ScriptPath.Mid(...)` up to
the first `.`, then matched against `IPluginManager`'s discovered plugins. Given a short name there
is no module segment to extract, so it returns `false` and the caller falls through to its
`CLASS_NOT_FOUND` branch. That is a reasonable thing for *this* function to do; the defect is that
nothing else covers the gap.

## The mismatch is the point

`graphClass` is deliberately resolved through the plugin's shared resolver, which accepts **both**
forms. `PCGGraphCreate.cpp:47-57`:

```cpp
UClass* GraphClass = UPCGGraph::StaticClass();                                     // :47
const FString GraphClassInput = Ctx.GetString(TEXT("graphClass")).TrimStartAndEnd();
if (!GraphClassInput.IsEmpty())
{
    GraphClass = ResolveUClass(GraphClassInput);                                   // :51
    if (!GraphClass && PinWrightPCG::SendPluginDisabledForScriptPath(Ctx, GraphClassInput))  // :52
    {
        return true;
    }
```

`Utils/ClassUtils::ResolveUClass` (declared `ClassUtils.h:16`, defined `ClassUtils.cpp:102`) is
described in its own header as resolving *"a full path, a blueprint class path, or a short class
name"*, and step 4 onward (`ClassUtils.cpp:150-180`) is entirely short-name handling: a fixed
`/Script/...` package sweep, then a `TObjectIterator<UClass>` name match, then prefix-stripping
retries. So the request is routed through a resolver whose whole contract is *both forms are
equal*, and then into a guard that only recognises one of them. The two spellings are equivalent
right up to the moment the answer is chosen.

**The short form is advertised, and by this verb's own schema.** `PCGGraphCreate.cpp:29-33`:

```cpp
RPC_PARAM_DEF("graphClass", "string",
    "UPCGGraph subclass to instantiate: full path (/Script/<Module>.<Class>) or short class "
    "name. ...
```

That sentence is served verbatim on the generated per-method page,
`Saved/PinWright/wiki/pcg.create_graph.md`, as the `graphClass` param row — so the page a caller
reads before the call offers the short form, and the code path behind it degrades when they take it.
(Correction to how this was reported to me: `Docs/wiki-src/pcg.md` does **not** mention the short
form anywhere — grep for "short" in that file returns nothing. The offer lives in the handler
schema and reaches the caller through the generated page, not through the namespace overlay. The
namespace overlay's own `graphClass` bullet, `pcg.md:12`, uses the full path throughout and ends
*"the call returns `PLUGIN_DISABLED` naming the plugin, not `CLASS_NOT_FOUND`"* — which is true only
for the spelling it happens to use.)

## Coverage fact, not a criticism of the test

`PinWright.pcg.add_node.NamesDisabledPlugin`
(`Source/PinWrightPCG/Private/Tests/PCG/TestPCGAddNodeOptionalPluginGuard.cpp:26-28`) pins exactly
the contract this ticket is about — *"a typed refusal that NAMES the plugin, never a bare
CLASS_NOT_FOUND"* (`:22-24`), asserted at `:71-76`. It passes, and it could not have caught this,
because the only class string it ever sends is the full form, `:33`:

```cpp
const TCHAR* const NodeClassPath = TEXT("/Script/ProceduralVegetation.PVBaseSettings");
```

One extra block passing the short name `PVBaseSettings` through the same skip-gated path would close
it. Worth noting that this test already skips honestly on hosts where the plugin is absent or
enabled (`:36-52`), so the second block costs nothing on those hosts either.

## What it should do

Either is defensible; the second is cheaper and does not widen the helper's contract:

1. Have `SendPluginDisabledForScriptPath` accept a bare class name — sweep discovered plugins for a
   disabled one whose modules could own a class of that name. This is guesswork without a module
   segment and can misattribute, so it is the weaker option.
2. Resolve the caller's input to a canonical `/Script/` path **before** the guard, or try the guard
   again with the class's full path when `ResolveUClass` fails on a short name. The verbs already
   accept both spellings; normalising once at the top of the handler makes every downstream branch —
   guard, abstract check, error text — see one form.

Whichever is chosen, `pcg.add_node`'s `nodeClass` needs the same treatment in the same change: both
call sites go through the identical helper (`PCGGraphAuthoring.cpp:67`, `PCGGraphCreate.cpp:52`).

## Same shape as

There is an existing "short class-name form breaks" family on this board, all three verified
present:

- `B-actor-find-by-class-short-name-fails` (IN-REVIEW, Medium) — *"silently returns count:0 for
  short class names despite documenting them"*.
- `B-inspect-class-short-name-fails` (DONE, Medium) — *"requires full `/Script/` path despite
  short-name promise"*.
- `B-asset-list-short-class-ensure` (DONE, Medium) — *"fires `FTopLevelAssetPath` ensure on short
  class names"*.

Each is a separate verb degrading on the short form while advertising it; this is the fourth, and
the first where the degradation is a *wrong error code* rather than a wrong result. Filed
separately per board policy on umbrellas, but a fixer working the family should take this with them.

## Distinct from

- `E-pv-graph-path-needs-optional-plugin-guard` (being closed DONE this session, and closed with
  this gap named) — the ticket that *introduced* `SendPluginDisabledForScriptPath`. Its analysis of
  why a runtime probe is required, and why `__has_include` / `TryAddConditionalModule` cannot
  express a dependency carried in a parameter value, is correct and is not in question. This is the
  guard's input domain being narrower than the parameter's.
- `F-pcg-create-graph-class-parameter` (IN-REVIEW) — the ticket that added `graphClass` and wired it
  into that guard. Its `#2` records the resolver choice (`ResolveUClass`, *"the plugin's shared
  full-path-or-short-name resolver"*) as a deliberate decision, which it should stay; the mismatch
  is with the guard, not with the resolver.

## History
- `#1-short-name-falls-through-the-guard` `OPEN` reporter — Measured: `pcg.create_graph {graphClass: "ProceduralVegetationGraph"}` answers `CLASS_NOT_FOUND`, while the same class as `/Script/ProceduralVegetation.ProceduralVegetationGraph` answers `PLUGIN_DISABLED`; `pcg.add_node`'s `nodeClass` splits the same way. Cause re-derived at HEAD: `PinWrightPCG::SendPluginDisabledForScriptPath` (`Source/PinWrightPCG/Private/Handlers/PCG/PCGHandlerHelpers.h:105`) returns `false` at `:108-111` for any input not starting with `/Script/`, because it parses the module name out of that prefix; the caller then falls into its `CLASS_NOT_FOUND` branch (`PCGGraphCreate.cpp:61-68`). The mismatch: `graphClass` is resolved by `Utils/ClassUtils::ResolveUClass` (`PCGGraphCreate.cpp:51`; declared `ClassUtils.h:16` as accepting "a full path, a blueprint class path, or a short class name", short-name paths at `ClassUtils.cpp:150-180`), so the resolver accepts a spelling the guard structurally cannot see. The short form is advertised by the verb's own schema, `PCGGraphCreate.cpp:30-31` ("full path (/Script/<Module>.<Class>) or short class name"), served verbatim as the `graphClass` param row on the generated page `Saved/PinWright/wiki/pcg.create_graph.md`. Correcting the report as given: `Docs/wiki-src/pcg.md` does NOT offer the short form — no occurrence of "short" in that file — its `graphClass` bullet at `:12` uses the full path throughout, so the offer reaches the caller through the handler schema and the generated page only. Coverage fact, not a criticism: `PinWright.pcg.add_node.NamesDisabledPlugin` (`Tests/PCG/TestPCGAddNodeOptionalPluginGuard.cpp:26-28`, assertions `:71-76`) pins this exact contract and passes, but sends only the full form (`:33`), so it could not have caught this; one extra block on the short name inside the same skip gate (`:36-52`) closes it. Ask: normalise the caller's class input to a canonical `/Script/` path before the guard runs (preferred), or teach the guard to handle a bare name; both `pcg.add_node` and `pcg.create_graph` route through the same helper (`PCGGraphAuthoring.cpp:67`, `PCGGraphCreate.cpp:52`) and need the same edit. Cross-linked `E-pv-graph-path-needs-optional-plugin-guard` (the guard's origin ticket; its runtime-probe analysis stands) and `F-pcg-create-graph-class-parameter` (whose `#2` chose `ResolveUClass` deliberately — the resolver is right, the guard is narrower). Verified the pre-existing short-name family on the board: `B-actor-find-by-class-short-name-fails`, `B-inspect-class-short-name-fails`, `B-asset-list-short-class-ensure` all exist; recorded as a `## Same shape as` block rather than merged, per the no-umbrella policy. Severity Low, argued: impact class is Medium — a soft blocker with a workaround (spell the full path) that `pcg.md:12` does document — and the rubric's reach modifier then bumps it down one, because this is a genuinely rare edge path: it needs the PCG plugin enabled, a non-default `graphClass`, and the experimental off-by-default Procedural Vegetation Editor, a combination the plugin's own `GraphClassSelectsSubclass` test skips on most hosts for lack of any loaded `UPCGGraph` subclass. Declining to hold it at Medium on the grounds that the failure rate *within* that narrow population is 100%: hit rate inside a population is not the rubric's reach term, and `encounters` is the tiebreak for that, never a severity input. Not rated High: the call refuses, nothing wrong is created, and the response is an honest failure carrying a misleading label.
