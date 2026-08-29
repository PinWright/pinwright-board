---
id: B-material-param-setters-wrong-class-error
title: "The three typed material parameter setters answer ASSET_NOT_FOUND when handed a base UMaterial — the asset exists and loaded, it is the wrong class, and the correct UNSUPPORTED_ASSET_CLASS helper is 80 lines below in the same file and already used by six other handlers"
status: OPEN
severity: Medium
category: bug
tags: [material, material-authoring, error-codes, wrong-diagnosis, asset-not-found, unsupported-asset-class, set_scalar_parameter_value, set_vector_parameter_value, set_texture_parameter_value]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# A wrong-class asset is reported as a missing one, sending the caller to re-check a correct path

Point `material.authoring.set_vector_parameter_value` or
`set_scalar_parameter_value` at a base `UMaterial` and the answer is:

    ASSET_NOT_FOUND: Could not load material instance

The asset is there. It loaded. It is a `UMaterial` and these verbs take a
`UMaterialInstanceConstant`. The error names the one thing that is not wrong.

All three typed setters do a bare `LoadObject<UMaterialInstanceConstant>` and
treat the null as "missing":

| verb | registration | load | error |
|---|---|---|---|
| `set_scalar_parameter_value` | `MaterialAuthoringHandler.cpp:2112` | `:2127` | `:2130` |
| `set_vector_parameter_value` | `:2151` | `:2166` | `:2168` |
| `set_texture_parameter_value` | `:2200` | `:2217` | `:2219` |

Each is the same three lines:

```cpp
    UMaterialInstanceConstant* Instance = LoadObject<UMaterialInstanceConstant>(nullptr, *AssetPath);
    if (!Instance)
    {
        Ctx.SendError(TEXT("ASSET_NOT_FOUND"), TEXT("Could not load material instance."));
        return true;
    }
```

## This is already solved in this file, and the file says so

`LoadMaterialInstanceOrError` sits at `MaterialAuthoringHandler.cpp:2251-2272`,
80 lines below the last of the three, and does exactly the right thing: try the
instance, and if that fails but *something* loads at the path, answer
`UNSUPPORTED_ASSET_CLASS` naming the class actually received, falling back to
`ASSET_NOT_FOUND` only when nothing loads at all. Its own comment says it "emits
the mirror of get_material_info's UNSUPPORTED_ASSET_CLASS branch when the asset
exists but isn't an instance."

**Six handlers in the same file already call it** — `:2290`, `:2402`, `:2443`,
`:2483`, `:2612`, `:2749` — including `clear_parameter_override`, which is the
verb a caller reaches for immediately after the setters. So the same file answers
the same mistake two different ways depending on which verb you happened to call.

The plugin has also already written down why this particular wrong code is
expensive. `MaterialFinders.h:107-111`, on the mirror-image helper:

> "Both used to answer ASSET_NOT_FOUND, which sends the caller off to re-check a
> path that was correct all along — the most expensive wrong error in this
> family, because a material INSTANCE is the single most common thing to point
> one of these verbs at. ... UNSUPPORTED_ASSET_CLASS is the spelling this handler
> family already uses for class discrimination (get_material_info,
> LoadMaterialOrFunctionForMutationOrReportError, LoadMaterialInstanceOrError),
> so no new code."

That fix was applied to the verbs that expect a `UMaterial` and get an instance
(`ReportMaterialLoadFailure`, `MaterialFinders.h:113-131`, used by
`material.graph.get_node_details` at `MaterialGraphHandler.cpp:342` among others).
The three verbs that expect an instance and get a `UMaterial` — the exact
opposite mistake, equally common — were left behind.

## Impact

The message is not merely unhelpful, it is misdirecting. A caller who reads
"Could not load material instance" concludes the path is wrong and goes off
re-checking it, re-listing the folder, or re-creating the asset. The actual fix
is to create or target a material instance, or — if what they wanted was to
change the base material's parameter default — to discover that no verb does
that at all (`F-base-material-param-default-setter`). Neither conclusion is
reachable from the error they were given.

## Fix

Replace the three bare loads at `:2127`, `:2166` and `:2217` with the existing
`LoadMaterialInstanceOrError(Ctx, AssetPath)` at `:2251`, matching the six
handlers below. It is a three-line change, needs no new error code, no new
helper and no new string, and `MaterialFinders.h:111` already declares this the
house spelling. Consider having the wrong-class message also name the per-base
counterpart the way `ReportMaterialLoadFailure`'s `PerInstanceRouteVerb`
parameter does in the other direction — though for these three that counterpart
does not exist yet, which is `F-base-material-param-default-setter`.

Regression test: point each of the three setters at a `UMaterial` and assert
`UNSUPPORTED_ASSET_CLASS` with the class name in the message; point one at a
genuinely absent path and assert `ASSET_NOT_FOUND` still. `TestMaterialInstanceBasePropertyOverrides.cpp:127-128`
already asserts an `UNSUPPORTED_ASSET_CLASS` code and is the pattern to copy.

severity rationale: impact — nothing false is returned as *data* and nothing is
built on the result (the call correctly fails; only the explanation is wrong), so
this is not the High band's "the caller trusts a result that is a lie and builds
on it". By impact class alone it is the Low band's diagnostic/naming friction.
Reach modifier: I **take the bump-up**. These three setters are the most-used
verbs in the material-authoring surface, and confusing a base material with its
instance is the single most common way to misaim them — the plugin's own comment
at `MaterialFinders.h:109-110` calls the mirror of this "the most expensive wrong
error in this family" and fixed it there for that reason. A Low-impact defect on
a near-every-material-session path is exactly the case the rubric says to raise.
I **decline** any further bump to High for the reason in the first sentence.
-> **Medium**.

## Same shape as

- `F-base-material-param-default-setter` (OPEN, Low) — the capability gap behind
  the same call. That ticket is "no verb edits a base UMaterial's parameter
  default"; this one is "the verb you tried tells you the wrong thing about why".
  Filed separately rather than folded in because the fix here is mechanical and
  worth landing whether or not that feature ever ships, and because bundling a bug
  into a feature request would tie its schedule to the feature's. This ticket's
  measured error is recorded in that ticket's `#2` encounter.
- `F-material-instance-overrides-incomplete` (IN-REVIEW, High) — records the
  mirror case working correctly: `get_material_info` rejects a
  `UMaterialInstanceConstant` with `UNSUPPORTED_ASSET_CLASS` (its `#2` cites the
  class check at `:2147`), and its `#3` gave `get_material_instance_info` the same
  guard. Different axis: that ticket is about missing instance *capabilities*
  (clear, static switch, reparent, readback, batch), not about error vocabulary —
  it is cited here only as the precedent that this handler family already
  discriminates class correctly everywhere except these three lines.

## History
- `#1-wrong-code-on-wrong-class` `OPEN` reporter — Observed live: `material.authoring.set_vector_parameter_value` / `set_scalar_parameter_value` against a base `UMaterial` return `ASSET_NOT_FOUND: Could not load material instance`, when the asset exists, loads, and is merely the wrong class. Re-derived at HEAD in this checkout: all three typed setters do a bare `LoadObject<UMaterialInstanceConstant>` and send `ASSET_NOT_FOUND` on null — scalar `MaterialAuthoringHandler.cpp:2112`/`:2127`/`:2130`, vector `:2151`/`:2166`/`:2168`, texture `:2200`/`:2217`/`:2219`. The correct helper is in the SAME FILE at `:2251-2272` (`LoadMaterialInstanceOrError` — `UNSUPPORTED_ASSET_CLASS` naming the received class, `ASSET_NOT_FOUND` only when nothing loads) and six handlers already call it (`:2290`, `:2402`, `:2443`, `:2483`, `:2612`, `:2749`), so one file answers the same mistake two ways. `MaterialFinders.h:107-111` names this exact anti-pattern in its own comment and calls the mirror of it "the most expensive wrong error in this family"; that fix landed on the UMaterial-expecting side (`ReportMaterialLoadFailure`, `:113-131`) and skipped the three instance-expecting setters. Fix is three lines: swap the loads for the existing helper. Dedup: board searches for `ASSET_NOT_FOUND` (20 files, all about assets genuinely not written or not found — none about a wrong-class misreport) and for the literal message "Could not load material instance" (0 files). Split out of `F-base-material-param-default-setter`'s `#2` encounter deliberately — different defect, mechanical fix, independent value — and cross-linked both ways.
