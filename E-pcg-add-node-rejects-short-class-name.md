---
id: E-pcg-add-node-rejects-short-class-name
title: "pcg.create_graph resolves its class parameter with the plugin's mandated ResolveUClass and pcg.add_node resolves its own with the FindObject<UClass>(nullptr, shortName) anti-pattern agent-conventions.md names by name — one namespace, one concept, two contracts, and only one of them documented"
status: OPEN
severity: Medium
category: ergonomic
tags: [pcg, add_node, create_graph, short-class-name, resolveuclass, findobject, class-resolution, asymmetry, convention-violation, error-message, procedural-vegetation]
encounters: 1
lastSeen: 2026-08-29T17:25:00+05:00
---

# Two verbs, one namespace, one concept, two resolvers

Both verbs ask the caller to name a `UClass`. `pcg.create_graph`'s `graphClass` accepts a full path
*or* a short class name and says so. `pcg.add_node`'s `nodeClass` accepts only a full path, says
nothing about it, and refuses the short form with an error that blames the caller's class name.

## Measured

On a host with the Procedural Vegetation Editor plugin **enabled** — so no optional-plugin guard is
in play on any row:

| call | result |
|---|---|
| `create_graph {graphClass: "ProceduralVegetationGraph"}` | created a `ProceduralVegetationGraph` asset |
| `add_node {nodeClass: "/Script/ProceduralVegetation.PVSeedGeneratorSettings"}` | created `SeedGenerator_0` |
| `add_node {nodeClass: "PVSeedGeneratorSettings"}` | `CLASS_NOT_FOUND` |
| `add_node {nodeClass: "PCGCreatePointsSettings"}` | `CLASS_NOT_FOUND` |

The last row is the one that isolates the cause. `UPCGCreatePointsSettings` belongs to the
always-on PCG plugin, is certainly loaded, and is the very class `add_node`'s own schema uses as its
example. Its short name is refused anyway, so this has nothing to do with optional plugins or with
Procedural Vegetation — `add_node` refuses the short form for every class, always.

The error text: `Could not resolve UPCGSettings subclass: PCGCreatePointsSettings`. Which is untrue
of the class and misleading about the cause.

## Cause

`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphAuthoring.cpp:62-66`:

```cpp
UClass* SettingsClass = FindObject<UClass>(nullptr, *NodeClassPath);     // :62
if (!SettingsClass)
{
    SettingsClass = LoadClass<UPCGSettings>(nullptr, *NodeClassPath);    // :65
}
```

Two lookups, neither of which can see a short name. `LoadClass` needs a resolvable object path.
And `FindObject<T>(nullptr, shortName)` **searches only the root package** — which is not an
incidental limitation but an entry in `agent-conventions.md` § *Lookalike-API traps*, listed
verbatim with its remedy:

> `FindObject<T>(nullptr, shortName)` searches ONLY the root package — misses game/plugin types. Use
> the 3-tier chain: full-path load → `FindFirstObjectSafe` → `TObjectIterator` fallback
> (`ResolveUClass`/`ResolveUEnum` in `Utils/ClassUtils`).

So `add_node` contains the exact anti-pattern the conventions document names, and the prescribed
helper is not merely available — **its sibling verb in the same namespace already uses it.**
`grep -rn "ResolveUClass" Source/PinWrightPCG/` returns exactly one hit in the entire module:
`PCGGraphCreate.cpp:51`, `graphClass`'s resolution. `PCGGraphAuthoring.cpp` has zero.

`Utils/ClassUtils::ResolveUClass` (declared `ClassUtils.h:16`, defined `ClassUtils.cpp:102`)
describes itself as resolving *"a full path, a blueprint class path, or a short class name"*, and
`ClassUtils.cpp:150-180` is entirely short-name handling — a fixed `/Script/...` package sweep, a
`TObjectIterator<UClass>` name match, then prefix-stripping retries.

## Only one of the two contracts is written down

`create_graph` advertises the short form in its own schema, `PCGGraphCreate.cpp:30`:

> `"UPCGGraph subclass to instantiate: full path (/Script/<Module>.<Class>) or short class name. …"`

`add_node` advertises full paths only, `PCGGraphAuthoring.cpp:40`:

> `RPC_PARAM_REQ("nodeClass", "string", "UPCGSettings subclass path, e.g. /Script/PCG.PCGCreatePointsSettings.")`

and `Docs/wiki-src/pcg.md:11` matches it, using a full path throughout.

**So `add_node` is not breaking a promise, and that is deliberately not the complaint here.** The
complaint is that two verbs a caller reaches within seconds of each other — you cannot call
`add_node` without having called `create_graph` first — take the same kind of argument under
different rules, that nothing in either page mentions the difference, and that the failure surfaces
as an error which asserts a loaded class could not be resolved. A caller who learns the short form
on the first verb and carries it to the second gets told their class name is wrong when it is
correct, which is the outcome `agent-conventions.md` § *Error conventions* singles out: *"Generic
'Could not resolve X' makes callers blame the tool instead of their input"* — here, the inverse and
worse, it makes them blame their input when the input is fine.

## The ask

Route `nodeClass` through `ResolveUClass`, keeping the existing `IsChildOf(UPCGSettings)` gate at
`PCGGraphAuthoring.cpp:74` and the `CLASS_NOT_FOUND` emit at `:76` for genuine misses. That is the
plugin's mandated resolver, already linked into this module, already used five files away by the
sibling verb, and it makes both `pcg` class parameters obey one rule. Then say so in the schema
string at `:40` and in `pcg.md:11`.

The delta this actually buys, stated precisely so the change is not oversold: `LoadClass` at `:65`
already handles blueprint class paths (`/Game/…/BP_X.BP_X_C`), so what `ResolveUClass` adds is short
names plus its prefix-stripping retries and `TObjectIterator` fallback.

**Sequencing note, and it matters.** Landing this alone will reproduce
`E-plugin-disabled-guard-misses-short-class-name`'s defect on `add_node` in its true form. Today a
short name on `add_node` fails in the resolver, before any guard is consulted. Once `ResolveUClass`
is in place, a short name naming a class from a **disabled** plugin will resolve to nothing and fall
through to `SendPluginDisabledForScriptPath` (`PCGGraphAuthoring.cpp:67`), which bails on any input
without a `/Script/` prefix (`PCGHandlerHelpers.h:108`) — so the caller gets `CLASS_NOT_FOUND` where
`PLUGIN_DISABLED` is the right answer, exactly the gap that ticket describes on `create_graph`.
**Land the two together, or land the guard fix first.** Recording this because a fixer taking this
ticket in isolation will hand the next tester a regression that looks like the same bug.

## Same shape as

The board's existing "short class-name form breaks" family, all verified present:

- `B-actor-find-by-class-short-name-fails` (IN-REVIEW, Medium) — *"silently returns count:0 for short
  class names despite documenting them"*.
- `B-inspect-class-short-name-fails` (DONE, Medium) — *"requires full `/Script/` path despite
  short-name promise"*.
- `B-asset-list-short-class-ensure` (DONE, Medium) — *"fires `FTopLevelAssetPath` ensure on short
  class names"*.
- `E-plugin-disabled-guard-misses-short-class-name` (OPEN, Low) — the fourth, on `create_graph`, and
  the one whose `add_node` claim this ticket corrects (see below).

This is the fifth, and the first where the verb never advertised the short form — which is why it is
filed `E-` rather than `B-`, and why it is about consistency between siblings rather than a broken
promise. Per the no-umbrella policy the family stays as separate tickets, but they share one fix
shape (use `ResolveUClass`) and a fixer working any of them should carry the rest.

## Corrects `E-plugin-disabled-guard-misses-short-class-name`

That ticket's `#1` says *"`pcg.add_node` has the same split on `nodeClass`"* and prescribes a single
edit covering both verbs. The `PCGCreatePointsSettings` row above disproves the shared diagnosis:
on `add_node` the short form does not fail because the plugin guard cannot see it, but because the
verb never accepted short names at all. That ticket's own prescription — *"normalise the caller's
class input to a canonical `/Script/` path before the guard"* — presupposes a resolver that can turn
a short name into a path, which `create_graph` has and `add_node` does not, so there is nothing to
normalise from until this ticket lands. The correction is recorded on that ticket as its `#2` and in
its body, and its `create_graph` half stands untouched.

## Dedup

Board-wide search for `add_node`, `nodeClass`, `ResolveUClass`, `FindObject` and short-class-name
tickets. `E-pcg-add-node-echo-pin-labels` (OPEN, Low, `encounters: 6`) is the same verb but the
response shape. `B-pcg-add-node-no-abstract-class-guard` (OPEN, Medium) is the same resolution block
but the *gate* after it, not the lookup — and worth flagging as a co-located fix: both changes land
within a dozen lines of each other in `PCGGraphAuthoring.cpp`, so whoever opens the file should read
both. `F-pcg-create-graph-class-parameter` (DONE) is the ticket that put `ResolveUClass` on the
sibling verb and thereby created this asymmetry; nothing in it is being questioned. No ticket owns
`add_node`'s class lookup.

## Severity

**Medium, argued.** Impact class is the rubric's Medium — *"Doable, but only via a documented
workaround, a **source dive**, or many extra calls"*. The workaround (spell the full path) is
documented on this verb, but discovering *why* the short name works two calls earlier and fails here
is a source dive by construction: neither wiki page mentions that the two verbs differ, and the
error text asserts the class could not be resolved when it is loaded and resolvable, so the caller's
first three hypotheses — typo, wrong class, plugin not loaded — are all wrong.

**The Low reading, and why it is refused.** *"Spell the full path, it is documented"* makes this look
like pure friction, which is the Low band. It is refused for two reasons. First, the friction is
only cheap for a caller who already knows the two verbs differ, and nothing in the shipped surface
tells them; the cost is not "type more characters" but a failed call plus a misleading diagnosis.
Second, and deciding: this is not a naming preference but a **convention violation** —
`FindObject<UClass>(nullptr, shortName)` is listed by name in `agent-conventions.md` § *Lookalike-API
traps* with `ResolveUClass` prescribed as the remedy, in a module that already links that helper and
already calls it from the sibling verb. A rule the project wrote down and then broke in one of two
adjacent call sites is worth more than the Low band's *"docs, discoverability, naming"*.

**Not High.** Nothing wrong is created, nothing is silently mis-written, and the call refuses
honestly — it is an honest failure carrying a misleading label, which is the same reasoning
`E-plugin-disabled-guard-misses-short-class-name` used to decline High.

**Reach modifier declined in both directions, and named.** No bump up: `pcg.add_node` is the most
frequently called verb in its namespace — a PV plant graph is 15-40 nodes, so it is called 15-40
times per graph — but PCG authoring is not an every-session activity across the plugin's whole
surface, which is the reading the rubric's reach term asks for and the same one
`B-pcg-generate-instancecount-blind-to-species` applied to this namespace. No bump down: it is not a
rare edge path either, since the short form is what the sibling verb teaches. Medium stands
unmodified.

## History
- `#1-add-node-never-accepted-short-names` `OPEN` reporter — Measured on a host with the Procedural Vegetation Editor plugin **enabled**, so no optional-plugin guard is in play: `pcg.create_graph {graphClass: "ProceduralVegetationGraph"}` succeeds, `pcg.add_node {nodeClass: "/Script/ProceduralVegetation.PVSeedGeneratorSettings"}` succeeds, and both `nodeClass: "PVSeedGeneratorSettings"` and `nodeClass: "PCGCreatePointsSettings"` return `CLASS_NOT_FOUND`. The last is the isolating case — `UPCGCreatePointsSettings` is from the always-on PCG plugin, is loaded, and is the class `add_node`'s own schema uses as its example — so `add_node` refuses the short form for every class, always, with nothing to do with optional plugins. Cause at `PCGGraphAuthoring.cpp:62-66`: `FindObject<UClass>(nullptr, *NodeClassPath)` at `:62` then `LoadClass<UPCGSettings>(nullptr, *NodeClassPath)` at `:65`, neither of which can see a short name; `FindObject<T>(nullptr, shortName)` searching only the root package is listed by name in `agent-conventions.md` § *Lookalike-API traps* with `ResolveUClass` prescribed as the remedy. `grep -rn "ResolveUClass" Source/PinWrightPCG/` returns exactly one hit in the whole module — `PCGGraphCreate.cpp:51`, the sibling verb's `graphClass` — and `PCGGraphAuthoring.cpp` has zero, so the mandated helper is linked, in use one file away, and not called here. Documented contracts diverge and only one side is written down: `PCGGraphCreate.cpp:30` offers *"full path (/Script/<Module>.<Class>) or short class name"*, while `PCGGraphAuthoring.cpp:40` offers *"UPCGSettings subclass path, e.g. /Script/PCG.PCGCreatePointsSettings."* and `pcg.md:11` matches it. Filed `E-` rather than `B-` deliberately: `add_node` never promised the short form, so this is an asymmetry between siblings plus a misleading error, not a broken promise — the error asserts a loaded, resolvable class could not be resolved, which inverts `agent-conventions.md` § *Error conventions* by making the caller blame input that is correct. Ask: route `nodeClass` through `ResolveUClass`, keep the `IsChildOf` gate at `:74` and the `CLASS_NOT_FOUND` emit at `:76`, then update the schema string and `pcg.md:11`; the honest delta is short names plus prefix-stripping and the `TObjectIterator` fallback, since `LoadClass` already covers blueprint class paths. **Sequencing note recorded in the body**: landing this alone reproduces `E-plugin-disabled-guard-misses-short-class-name`'s defect on `add_node` in its true form, because a short name for a disabled plugin's class would then resolve to nothing and fall into `SendPluginDisabledForScriptPath` (`:67`), which bails on non-`/Script/` input (`PCGHandlerHelpers.h:108`) — land the two together or the guard fix first. This ticket also **corrects** that ticket's claim that `add_node` has the same split and its prescription that one edit covers both verbs; the correction is recorded there as its `#2` and its `create_graph` half stands untouched. Dedup: `E-pcg-add-node-echo-pin-labels` is the same verb's response shape; `B-pcg-add-node-no-abstract-class-guard` is the *gate* after this lookup rather than the lookup, and is flagged as a co-located fix a dozen lines away; `F-pcg-create-graph-class-parameter` (DONE) is what put `ResolveUClass` on the sibling and created the asymmetry. Recorded as a fifth member of the board's existing short-class-name family (`B-actor-find-by-class-short-name-fails`, `B-inspect-class-short-name-fails`, `B-asset-list-short-class-ensure`, `E-plugin-disabled-guard-misses-short-class-name`) via a `## Same shape as` block rather than merged, per the no-umbrella policy. Severity Medium with the Low reading named and refused — the friction is only cheap for someone who already knows the verbs differ, and this is a convention the project wrote down and broke in one of two adjacent call sites; High refused because the call fails honestly and creates nothing; reach declined in both directions.
