---
id: E-rendering-set-settings-enum-token-discovery
title: "rendering.set_project_settings enum-valued UPROPERTYs have no in-band valid-token list — forces an engine-header dive before the write"
status: OPEN
severity: Low
category: ergonomic
tags: [enum-token-discovery, rendering, project-settings, set_project_settings, enum, discovery, docs]
encounters: 1
lastSeen: 2026-07-01T23:27:37.8179159+03:00
---

# `rendering.set_project_settings` enum-value tokens are undiscoverable in band

`rendering.set_project_settings` accepts an arbitrary `{UPROPERTY: value}` map
over `URendererSettings`, but for **enum-valued** UPROPERTYs there is no in-band
way to discover the accepted value tokens. To set `LumenRayLightingMode` a caller
must already know its enum spells `SurfaceCache | HitLightingForReflections |
HitLighting` — none of which is documented on the wiki. Faced with that, the
attempt Glob'd + Grep'd + Read the installed engine's
`C:/UE_5.7/.../RendererSettings.h` header to confirm the token `HitLighting`
**before** issuing the write, exactly the off-tool engine-source dive the hard
constraints call a last resort.

The write itself succeeded (the token was correct on the first try because the
agent pre-read the header), so this is pure discoverability friction, not a
failure — but it is friction that lands on every enum property reachable through
`set_project_settings` that isn't one of the few knobs the sibling `set_*` verbs
happen to document.

## Distinct cell in the rendering enum-token matrix

This method already has three enum/token tickets, and this is a **fourth,
orthogonal** axis:

| axis | ticket |
|------|--------|
| unknown *property NAME* rejected, no near-match hint | `E-rendering-unknown-property-no-hint` (OPEN) |
| enum *value* read back as raw identifier (`DetailTracing`) vs friendly input | `E-rendering-readback-enum-token-asymmetry` (OPEN) |
| a *documented* param's friendly token wrongly rejected | `B-rendering-lumen-software-mode-token-alias` (IN-REVIEW) |
| **enum *value* INPUT tokens undiscoverable for arbitrary `set_project_settings` UPROPERTYs** | **this ticket** |

The read-back ticket's proposed token table covers `LumenSoftwareTracingMode`,
`DefaultFeatureAntiAliasing`, and `Reflections`/`DynamicGlobalIllumination` — it
does **not** cover `LumenRayLightingMode`, nor the open-ended enum surface that
the arbitrary-map `set_project_settings` exposes. The unknown-property ticket is
about the map's *key* (property name); this is about the *value* token for a
valid key. Same root pattern as `E-eqs-builtin-token-discovery` (IN-REVIEW —
token strings undiscoverable, errors echo only the bad input, no enumeration
surface), one namespace over.

## What it should do

Either (both are cheap, and complementary):

- **(a) docs:** on the `docs/wiki-src/rendering.md` overlay's
  `### rendering.set_project_settings` section, list the common enum-valued
  UPROPERTYs and their accepted tokens (at minimum `LumenRayLightingMode:
  SurfaceCache | HitLightingForReflections | HitLighting`), mirroring the token
  lists the page already gives for `set_lumen_method`/`set_dynamic_gi_method`. A
  caller could then self-serve the token without leaving the tool. (The downstream
  wiki edit is a separate process, not part of this ticket.)
- **(b) error-hint:** when an enum value fails coercion, have the
  `rejected[].reason` enumerate the accepted tokens for that enum (the handler
  already knows the `UEnum`), so a wrong first guess self-corrects in band instead
  of forcing an engine-header dive. This is the same error-hint shape the board
  has adopted elsewhere (`E-rendering-unknown-property-no-hint`,
  `E-add-metasound-node-error-no-hint`, `E-eqs-builtin-token-discovery`).

## Evidence

Surfaced from a Lumen cinematic-setup task (`rendering` namespace, focus
`rendering.get_project_settings`, outcome clean — every call `ok`/non-error, 8
real `mcp__pinwright__call` invocations, zero rejections). The CallAnalyzer
flagged a `wiki-nav` inefficiency on `rendering.set_project_settings`, and the
attempt's own friction note states it verbatim: *"the wiki documents softwareRTMode
tokens but not LumenRayLightingMode's valid enum values, so I confirmed HitLighting
against the engine RendererSettings.h header before the set_project_settings write."*
The call log shows the successful `rendering.set_project_settings
{ReflectionCaptureResolution: 512, LumenRayLightingMode: HitLighting, save: true}`
was preceded by a `Glob → Grep → Read` of `RendererSettings.h` to source the
token — an off-tool step an in-band token list or a token-enumerating rejection
would have eliminated.

severity rationale: impact=docs/discoverability (Low) × reach=rare (rendering
config writes, and enum-property writes via `set_project_settings`, are not an
every-session path) -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the `rendering`
  Lumen cinematic-setup fuzz task (focus `rendering.get_project_settings`,
  outcome clean, 8 calls, zero rejections). `rendering.set_project_settings`
  takes an arbitrary `{UPROPERTY: value}` map but exposes no in-band list of valid
  enum-value tokens; to write `LumenRayLightingMode=HitLighting` the agent had to
  `Glob`/`Grep`/`Read` the installed engine's `RendererSettings.h` to confirm the
  token before the (successful) write — the last-resort engine-source dive the
  hard constraints flag. Distinct fourth axis from `E-rendering-unknown-property-no-hint`
  (unknown property NAME), `E-rendering-readback-enum-token-asymmetry` (read-back
  value form), and `B-rendering-lumen-software-mode-token-alias` (documented param
  token rejected); same root pattern as `E-eqs-builtin-token-discovery`. Proposed:
  (a) list common enum UPROPERTYs + tokens on the `docs/wiki-src/rendering.md`
  overlay, and/or (b) have the enum-coercion rejection enumerate the accepted
  tokens. Not re-filing the judge's `E-rendering-write-verb-persist-key-drift`
  (persist-key drift) which fully covers that separate friction point.
