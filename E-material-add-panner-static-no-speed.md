---
id: E-material-add-panner-static-no-speed
title: "material.authoring.add_panner's default Panner is static (SpeedX/SpeedY=0) and the trap is undocumented — the 'animated panner' intent silently ships static"
status: OPEN
severity: Medium
category: ergonomic
tags: [material, material-authoring, add-panner, panner, animation, speed, docs]
encounters: 1
costly: 1
lastSeen: 2026-07-01T00:17:53.1197644+03:00
---

# add_panner creates a static Panner and the static default is undocumented

`material.authoring.add_panner` creates a `UMaterialExpressionPanner` with the
engine defaults, whose `SpeedX` / `SpeedY` fields default to **0**, so the resulting
node **does not animate** unless its `Speed` input is separately driven. That matches
the engine editor exactly (a hand-placed Panner is likewise static until you give it
speed), so the tool is engine-faithful — this is a **discoverability/docs** gap, not a
handler bug. The problem is that for the overwhelmingly common intent ("I want an
animated, scrolling panner") a bare `add_panner` yields a **static** result and nothing
in the call, the response, or the wiki page signals it. The only way to discover the
trap today is to read the engine source (`MaterialExpressionPanner.h`, where
`SpeedX`/`SpeedY` are `0`-default `UPROPERTY`s).

The Panner is the canonical "scroll the UVs over time" node — callers reach for
`add_panner` *specifically* to make a surface feel alive (energy panels, flowing
water, conveyor textures). A default-static panner therefore silently fails the exact
criterion it was added for, and the documented remedy (drive the already-listed `Speed`
pin, or `property.set` `SpeedX`/`SpeedY`) is cheap **once you know it is required** —
the missing piece is the warning that the default is static.

## Evidence (this task)

A glowing-sci-fi-wall-panels build (focus null, namespace material.authoring,
outcome tool_bug) authoring `/Game/SciFi/Materials/M_EnergyPanel` (Unlit) needed
"a subtle animated, scrolling energy pattern moving across the surface so the
panels feel alive." Prep's plan assumed a bare `add_panner` would supply the
animation. Reality broke that assumption: the Attempt's THINK trace was *"check
the Panner expression's default speed in the engine source, since add_panner
exposes no speed param"* → a Bash grep of `MaterialExpressionPanner.h` showing
`float SpeedX; float SpeedY;` → *"SpeedX/SpeedY default to 0, so a bare Panner
won't animate — I need to set the speed."* The Attempt then had to add a whole
extra `PanSpeed` vector parameter (`add_vector_parameter {0.1, 0.05, 0, 0}`) and
a `connect_nodes(... inputName:"Speed")` to drive the Panner's Speed input so the
panel animated as the goal required. CallAnalyzer note: *"an agent without
`C:\UE_5.7` source would have shipped a static (Speed=0) panner and silently
failed the animation criterion."*

## What it should do

**Fix (docs):** in `docs/wiki-src/material.authoring.md`, add a
`### material.authoring.add_panner` section stating plainly that a default Panner is
**static** (`SpeedX`/`SpeedY` default to 0, so a bare add_panner does not animate) and
that animation requires either driving the already-documented `Speed` input (from a
scalar/vector parameter or a `Constant`, via `connect_nodes(... inputName:"Speed")`)
or a `property.set` on `SpeedX`/`SpeedY` — turning the engine-source detour into a
one-line lookup. Also annotate the `Panner` row of the Common-target-pins table (which
renders on the namespace page, where the per-method H3 does not) so the static default
is visible at browse time.

**Considered and rejected — method param (gold-plating):** a `speedX`/`speedY` (or
`speed` `Vector2`) param on `add_panner` that seeds the expression defaults would make
the one-call animated case work, but it singles out `add_panner` among ~20 sibling
convenience adders (`add_noise`, `add_fresnel`, `add_rotator`, `add_voronoi`, …) that
all likewise just apply engine defaults. "Adders should expose params to override
expression defaults" is a broader design question, not a Panner-specific fix; for a
rare (material-authoring convenience) case the proportionate remedy is the docs note,
not expanding the convenience surface for one node.

**Workaround:** author a scalar/vector parameter (or `Constant`) and
`connect_nodes(... inputName:"Speed")` it into the Panner, or `property.set`
`SpeedX`/`SpeedY` on the Panner node after `add_panner`.

severity rationale: impact=soft-blocker (the animated common case silently ships
static; recoverable only via an engine-source dive + an extra param + an extra
connect) × reach=rare (material-authoring convenience method, not every-session)
-> Medium

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a glowing-sci-fi-panels material.authoring task (focus null, namespace material.authoring, outcome tool_bug; 28 calls, max among siblings) authoring `/Game/SciFi/Materials/M_EnergyPanel` (Unlit). The goal explicitly required "a subtle animated, scrolling energy pattern." Prep assumed a bare `add_panner` would animate; the Attempt discovered via an engine-source grep (`MaterialExpressionPanner.h` → `float SpeedX; float SpeedY;`, both 0-default) that a default Panner is static and that `add_panner` exposes no speed param, then worked around it by adding a `PanSpeed` vector parameter and wiring it into the Panner's `Speed` input. Without `C:\UE_5.7` source an agent would have shipped a static panner and silently failed the animation criterion. Distinct from `E-material-pin-input-type-undiscoverable` (which is about Panner *pin names* Coordinate/Time/Speed being undiscoverable on per-method pages) — this is about the Panner's *default behavior* (Speed=0 ⇒ static) and the *missing speed param* on the convenience adder. Propose: `add_panner` accept optional `speedX`/`speedY` (or `speed` Vector2) that set the expression defaults, OR at minimum document the static-default trap on `docs/wiki-src/material.authoring.md` (add_panner section). Dedup: ripgrep over OPEN + closed found no panner-speed/animate ticket. severity rationale: impact=soft-blocker × reach=rare -> Medium.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded to the docs-only scope: the `add_panner` handler is engine-faithful (a hand-placed Panner is also static until Speed is driven), so the "method param that seeds SpeedX/SpeedY" half was dropped as gold-plating (it would single out add_panner among ~20 sibling adders that all just apply engine defaults — a broader design question, not a Panner fix). Landed the proportionate docs fix in `docs/wiki-src/material.authoring.md`: (a) added a `### material.authoring.add_panner` H3 section stating the default is static (SpeedX/SpeedY=0) and giving the two remedies — drive the `Speed` input via `connect_nodes(... inputName:"Speed")` from a scalar/vector parameter or Constant, or `property.set` SpeedX/SpeedY; (b) annotated the `Panner` row of the Common-target-pins table (renders on the namespace page, where the H3 does not) with the static-default warning. Regression test `Source/PinWright/Private/Tests/Infra/TestMaterialAddPannerStaticDefaultDocs.cpp` renders the `material.authoring.add_panner` method page through the live `WikiHandler::RenderPage` and asserts the overlay-exclusive markers (SpeedX/SpeedY/static; connect_nodes + Speed; property.set) survive — reverting the H3 fails it. No handler/transport/dispatcher code touched.
- `#3-attempt-failed` `OPEN` developer — Auto-fix attempt reached BLOCKED-PERMISSION; reverted and NOT pushed (build/tests not green).
