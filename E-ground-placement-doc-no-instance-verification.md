---
id: E-ground-placement-doc-no-instance-verification
title: "`spatial.ground-placement.md` is the family page every ISM/HISM scatter caller is routed to, and it has no verification path for instances: its `## Verifying` section documents only `spatial.verify_grounding`, which refuses an instance holder with `HOLDER_NOT_SEATABLE`, and prescribes matching the `undersideModel` you seated with — a parameter `spatial.ground_instances` rejects as `UNKNOWN_PARAMS`"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, wiki-src, spatial, ground_instances, ground_actors, verify_grounding, underside-model, unknown-params, holder-not-seatable, ism, hism, verification, discoverability]
encounters: 1
lastSeen: 2026-08-29T20:10:00+03:00
---

# The page tells instance callers where to go and then documents only the actor route

`Plugins/PinWright/docs/wiki-src/spatial.ground-placement.md` is the conceptual page for grounding.
Its scope line names two verbs (`:3`, *"The two verbs are `spatial.ground_actors` (seat a batch) and
`spatial.verify_grounding` (measure a batch, non-mutating)"*), but the page is the only prose that
exists for the family, and it addresses `spatial.ground_instances` directly at `:39`:

> This matters most on `spatial.ground_instances`, which seats instances against whatever surrounds
> them: a neighbouring scatter is nameable on no other axis.

That is its **one** mention of the verb. The page then runs a `## Verifying` section and a
`## Suggested workflow` that begins and ends with `spatial.verify_grounding` — a verb an instance
caller cannot use.

## The three things an instance caller cannot learn from the page

**1. `verify_grounding` refuses their holder.** Its own registered summary says so
(`GroundPlacementHandler.cpp:1038`): *"An ISM/HISM scatter HOLDER is refused with
HOLDER_NOT_SEATABLE, naming the component, rather than judged: its bounds are the union of every
instance, so a verdict about it would describe nothing."* Emitted at
`GroundPlacementUtils.cpp:694`. `spatial.ground_actors` refuses the same actor with the same code
(`GroundPlacementHandler.cpp:718`, emit `GroundPlacementUtils.cpp:1189`) — which is precisely what
routes the caller onto `ground_instances` in the first place. So the page's entire verification
chapter, and step 1 and step 3 of its suggested workflow, are unreachable for them, and the page
never says so. The only instance-side check is `spatial.ground_instances {apply: false}`, whose own
limits are `F-ground-instances-failure-rows-need-a-non-applying-seat`.

**2. The one verification instruction names a parameter their seat verb rejects.**
`spatial.ground-placement.md:67`:

> Verify with the **same** `surface` and the same `samples` / `undersideModel` you seated with, or the
> two are answering different questions.

`spatial.ground_instances` declares no `undersideModel` (17 parameters, `GroundPlacementHandler.cpp:1251-1329`),
and the dispatcher rejects any undeclared key with `UNKNOWN_PARAMS` before the handler runs
(`Dispatch/RpcDispatcher.cpp:126-162`, error emitted at `:159`). The value is hardcoded
`EUndersideModel::BoundsPlane` at `GroundPlacementHandler.cpp:1428`. A caller following the sentence
literally gets a rejection naming a parameter the docs told them to match.

**3. The page never states the hardcode or its reason.** `bounds_plane` appears **nowhere** in
`spatial.ground-placement.md`. Across all of `docs/wiki-src/` it appears once, at `spatial.md:275`,
and there it documents `ground_actors`' *choosable* `undersideModel`. So the caller-visible picture is
a parameter the sibling verb has, this verb lacks, and no page explains — indistinguishable from an
oversight. The reason is good and is written down in exactly one place a caller does not read
(`GroundPlacementHandler.cpp:1416-1418`: an instance has no per-column probeable underside, because
`LineTraceComponent` answers from every body at once), plus the response echo at `:1573`.

The same page contains **no occurrence of "align" or "tilt"**, so the missing `alignToSurface` /
`maxTilt` (`F-ground-instances-align-to-surface`) is equally invisible; and the generated method page
`Saved/PinWright/wiki/spatial.ground_instances.md` is pure registry auto-content with no overlay, so
it adds nothing.

## Premise corrected during filing

The report this ticket was filed from cited `spatial.ground-placement.md:69` and quoted it as *"the
same `undersideModel` you seated with"*. Both were checked:

- The line is **`:67`** in `Plugins/PinWright/docs/wiki-src/spatial.ground-placement.md`. `:69` is the
  line number in the **generated runtime copy** `Saved/PinWright/wiki/spatial.ground-placement.md`,
  which carries a 2-line header. A fixer edits the wiki-src file, so `:67` is the actionable citation.
- The wording is *"the same `samples` / `undersideModel`"*. `samples` **is** a valid
  `ground_instances` parameter, so only half the sentence is unusable.
- The sentence sits inside `## Verifying`, whose subject is `verify_grounding`. **So "the docs tell
  instance callers to do something impossible" overstates it** — the sentence is addressed to actor
  callers and is correct for them. The defect that survives is the one above: the page is the family
  page, it routes instance callers in at `:39`, and it then documents a verification path that
  excludes them without saying so.

## Ask

A `### spatial.ground_instances` section on `spatial.ground-placement.md` (or in the
`### spatial.ground_instances` overlay of `docs/wiki-src/spatial.md`, whichever the wiki owner
prefers — `F-ground-instances-align-to-surface`'s residual paragraph already nominated the latter),
saying four things:

1. `spatial.verify_grounding` refuses an ISM/HISM holder (`HOLDER_NOT_SEATABLE`), so the `## Verifying`
   section and the suggested workflow's steps 1 and 3 are actor-only. The instance check is
   `ground_instances {apply: false}`.
2. `undersideModel` is not a parameter here; it is fixed at `bounds_plane`, echoed in the response,
   and the reason is `LineTraceComponent`'s inability to attribute a hit to one instance body.
3. `alignToSurface` / `maxTilt` are not parameters here either (whatever
   `F-ground-instances-align-to-surface` lands, this line either states the gap or documents the new
   parameters).
4. Scope the `:67` sentence — "Verify with the same `surface` and the same `samples` / `undersideModel`"
   — to `spatial.ground_actors` explicitly, so an instance caller is not sent to match a rejected key.

## Related, and why this is not folded into them

- **`F-ground-instances-align-to-surface`** (OPEN, Medium) — its § *Scope* records the same
  documentation residue for `undersideModel` in one paragraph, as a note attached to a feature ask.
  That ticket will be worked as a C++ parameter addition; this one is a doc edit with a different
  owner and a different acceptance test, and it covers two things that ticket does not: the absent
  instance verification path, and the `:67` sentence's unusable half.
- **`F-ground-instances-failure-rows-need-a-non-applying-seat`** (OPEN, Medium) — owns the *capability*
  gap in instance verification. This ticket owns only the fact that the docs never say the capability
  is where it is.
- **`E-wiki-root-index-omits-standalone-guides`** — sibling wiki-coverage defect, different page.

## Severity

**Low.** The rubric's Low band is *docs, discoverability, naming* — a doc that fails to help. Nothing
here is a wrong result: the verbs behave correctly and their refusals are loud and correctly coded
(`HOLDER_NOT_SEATABLE`, `UNKNOWN_PARAMS`), so a caller who follows the page gets an error rather than
a silent bad seat. The cost is the round trip and the source dive to find out why.

**Not Medium**, argued rather than assumed. Medium would need the caller to be blocked, or forced into
a documented workaround. They are not: `ground_instances {apply: false}` works and is discoverable
from the method page's own `apply` parameter. What the page costs them is the *knowledge that the
family's verification chapter does not apply to them*, which they recover after one rejected call.

**Reach modifier declined.** `spatial.ground_instances` is the only per-instance grounding verb and
both sibling verbs' `HOLDER_NOT_SEATABLE` refusals funnel every ISM/HISM scatter caller onto it, so
every such caller reads this page — an argument for the bump. Declined because the grounding family is
a level-building path rather than an almost-every-session one, and because the failure is
self-announcing: the first `UNKNOWN_PARAMS` tells the caller the page is wrong for them.

## History
- `#1-family-page-has-no-instance-verification` `OPEN` reporter — Filed from a planting-diagnosis pass
  on `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8). Verified at HEAD `6d0e91a3`:
  `spatial.ground-placement.md:3` scopes itself to `ground_actors` + `verify_grounding`, mentions
  `spatial.ground_instances` exactly once (`:39`) and routes callers to it, then documents a
  `## Verifying` section and a suggested workflow built on `verify_grounding` — which refuses an
  ISM/HISM holder with `HOLDER_NOT_SEATABLE` per its own summary (`GroundPlacementHandler.cpp:1038`,
  emit `GroundPlacementUtils.cpp:694`), the same refusal that sent the caller there from
  `ground_actors` (`GroundPlacementHandler.cpp:718`, emit `:1189`). `:67`'s instruction to match the
  `undersideModel` you seated with cannot be followed on instances: the verb declares 17 parameters
  and not that one (`GroundPlacementHandler.cpp:1251-1329`), the dispatcher rejects undeclared keys
  with `UNKNOWN_PARAMS` (`Dispatch/RpcDispatcher.cpp:126-162`, emit `:159`), and the value is
  hardcoded at `GroundPlacementHandler.cpp:1428`. `bounds_plane` appears nowhere on the page and only
  once in all of `docs/wiki-src/` (`spatial.md:275`, about `ground_actors`); the page contains no
  occurrence of "align" or "tilt"; the generated `Saved/PinWright/wiki/spatial.ground_instances.md`
  has no overlay content. **Three premises corrected during filing:** the cited line is `:67` in
  wiki-src, not `:69` (that is the generated runtime copy, +2 for its header); the sentence names
  "the same `samples` / `undersideModel`", and `samples` IS valid on `ground_instances`; and the
  sentence sits under `## Verifying`, whose subject is `verify_grounding`, so "the docs tell instance
  callers to do the impossible" overstates — the surviving defect is that the family page routes
  instance callers in and then documents a verification chapter that excludes them, silently. Ask is a
  four-point `### spatial.ground_instances` section and an actor-scoping edit to `:67`. Not folded into
  `F-ground-instances-align-to-surface` (whose § Scope carries the `undersideModel` residue as one
  paragraph) because that is a C++ parameter ask with a different acceptance test, and this covers the
  absent verification path it does not. Severity Low on the docs band; Medium declined because
  `ground_instances {apply:false}` works and both refusals are loud and correctly coded, so the caller
  loses a round trip rather than being blocked; reach bump declined with the argument stated.
