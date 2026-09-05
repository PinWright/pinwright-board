---
id: E-wiki-describe-ops-boolean-material-stale
title: "model.describe_ops.md still teaches the pre-fix boolean-material rule — all three clauses of its line 22 are now false — and the two new PWMODEL_BOOLEAN_MATERIAL_* codes appear on one wiki page only"
status: OPEN
severity: Medium
category: ergonomic
tags: [docs, wiki, wiki-src, model, describe_ops, boolean, materials, stale-documentation, diagnostics-discoverability]
encounters: 1
lastSeen: 2026-09-05T17:59:42Z
---

# The page an author reads *before* writing an op teaches the behaviour the fix removed

`Saved/PinWright/wiki/model.describe_ops.md:22` reads, verbatim:

> `context` separates part ops from collision entries, so `box` appears twice with different
> parameters. **Boolean ops have `acceptsMaterial: false`: they discard their tool, and
> `material=` is an error.**

Every clause of that second sentence is false on the current build. Measured, not inferred —
`model.describe_ops {op:"subtract"}` against the live parser table returns:

```
name "subtract"   boolean true   acceptsBlock true   acceptsMaterial TRUE
params: fill_holes, simplify_output, allow_empty_result,
        material  "Material slot for the faces this operation CREATES - the walls a subtract
                   opens, the tool surface a union keeps, the caps fill_holes closes. ...
                   Untagged, new faces inherit the slot of the geometry the boolean was applied
                   to ... warns PWMODEL_BOOLEAN_MATERIAL_AMBIGUOUS."
```

| the page says | the build does |
|---|---|
| `acceptsMaterial: false` | `acceptsMaterial: true` |
| "they discard their tool" | the tool surface a `union` keeps is taggable; `subtract` walls inherit |
| "`material=` is an error" | `material` is a published parameter with its own semantics |

## Why this one matters more than an ordinary stale line

`model.describe_ops` is the op-table page — the thing an author consults **before** writing an op,
and the page `model.authoring.materials.md` itself points at for what each op accepts. So it
teaches the pre-fix rule to precisely the audience the fix was built for, and it teaches it as a
prohibition, which is the kind of statement a careful author obeys without testing. The sibling
page `model.authoring.materials.md` is fully current, which makes the disagreement worse rather
than better: two generated pages in the same wiki now contradict each other on whether a legal
parameter is an error.

The likely mechanical cause is that the `Docs/wiki-src/` overlay for `model.describe_ops` was not
updated alongside `model.authoring.md` when the boolean-material work landed. The wiki regenerates
at editor start from those overlays plus the handler registry, so the live `params` array is
correct while the hand-written prose above it is not — the page is half-generated and only half of
it moved.

## Second half: the new diagnostics are documented on exactly one page

`PWMODEL_BOOLEAN_MATERIAL_AMBIGUOUS` and `PWMODEL_BOOLEAN_MATERIAL_UNUSED` were both confirmed
reachable on this build (via `model.validate`, which writes nothing). `grep -rl` across the whole
generated wiki finds each on **one** page:

```
PWMODEL_BOOLEAN_MATERIAL_AMBIGUOUS -> Saved/PinWright/wiki/model.authoring.materials.md
PWMODEL_BOOLEAN_MATERIAL_UNUSED    -> Saved/PinWright/wiki/model.authoring.materials.md
```

They should also appear on **`model.compile.md`** and **`model.validate.md`**, which are the two
pages a caller reads when a code comes back in `diagnostics` and they need to know what it means.
`model.compile.md` already carries a diagnostics discussion and `diagnosticSummary` contract, so
there is a place for them; today a caller who receives one of these codes has no path from the verb
page to its explanation.

## Fix

- Rewrite `model.describe_ops`'s line 22 in its `Docs/wiki-src/` overlay to match the shipped
  behaviour: booleans accept `material=` for the faces they create, do not recolour geometry that
  already existed, and inherit the target's slot when untagged.
- Add both `PWMODEL_BOOLEAN_MATERIAL_*` codes to `model.compile.md` and `model.validate.md`.
- Worth a generated cross-check: any `PWMODEL_*` code reachable from the compiler that appears on
  fewer than the verb pages that can emit it is a discoverability gap of this same shape, and it is
  mechanically detectable rather than something a reviewer has to notice.

## History
- `#1-stale-boolean-material-clause-and-uncrossreferenced-codes` `OPEN` reporter - Found while verifying `B-pwmodel-boolean-output-takes-slot-zero` on the FPS build after the wave 4-9 plugin rebuild, 2026-09-05. `model.describe_ops.md:22` still states booleans have `acceptsMaterial: false`, discard their tool, and that `material=` is an error; the live table for `subtract` returns `acceptsMaterial: true`, `boolean: true` and a documented `material` parameter whose description names `PWMODEL_BOOLEAN_MATERIAL_AMBIGUOUS`. Re-derived directly rather than relayed. The page is the op table an author reads before writing an op and is referenced from `model.authoring.materials.md`, which is itself current — so two generated pages in one wiki now disagree about whether a legal parameter is an error, and the stale one states it as a prohibition. Cause is almost certainly an un-updated `Docs/wiki-src/` overlay: the generated `params` array below the prose is correct, so only the hand-written half is stale. Separately, both new codes appear on exactly one wiki page (`model.authoring.materials.md`) and on neither `model.compile.md` nor `model.validate.md`, so a caller who receives one in `diagnostics` has no route from the verb page to its meaning. Fix: correct the overlay, cross-reference both codes onto the two verb pages, and consider a generated check that every reachable `PWMODEL_*` code is documented on the verb pages that can emit it.
