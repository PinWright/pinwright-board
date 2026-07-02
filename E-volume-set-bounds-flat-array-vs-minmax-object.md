---
id: E-volume-set-bounds-flat-array-vs-minmax-object
title: "volume.set_volume_bounds takes 'bounds' as a flat positional [minX,minY,minZ,maxX,maxY,maxZ] array, but its title says 'using min/max corners' and its echo returns a structured {min:[…],max:[…]} object — the input/output shapes disagree and a caller thinking in min/max corner objects must consult the wiki to learn the flat 6-value form"
status: OPEN
severity: Low
category: ergonomic
tags: [volume, set_volume_bounds, bounds, flat-array, min-max, input-shape, asymmetry, docs, discoverability]
encounters: 1
lastSeen: 2026-06-24T06:10:13Z
---

# `volume.set_volume_bounds` input is a flat 6-value array while its title and echo speak in `{min,max}` corner objects

`volume.set_volume_bounds` is registered with the title *"Set volume bounds
using min/max corners"* and a single `bounds` param typed `array` with the
description *"Array of 6 values [minX,minY,minZ,maxX,maxY,maxZ]"*
(`VolumeHandler.cpp:1401-1404`). The handler reads that flat array positionally
(`:1414-1420`). But the **response echo** emits `bounds` as a *structured*
object — `{min:[x,y,z], max:[x,y,z]}` (`:1466-1470`) — not the flat positional
array the request takes.

So the same method names "bounds" in two different shapes:
- **request:** a flat positional `[minX,minY,minZ,maxX,maxY,maxZ]` array;
- **echo:** a `{min:[…], max:[…]}` object.

A caller who reads the title ("using min/max corners") or the task naturally —
"set its bounds using min corner {x,y,z} and max corner {x,y,z}" — reaches for
two `{x,y,z}` corner objects (the same `{x,y,z}` shape every other volume verb
uses for `location`/`extent`), which is *not* what the param accepts. The flat
6-value positional form is discoverable only by reading the param description in
the wiki before calling. This is a small but real input/output-shape asymmetry:
the verb describes itself in min/max-corner terms, echoes a min/max object, yet
ingests a flat positional array.

## Why this is ergonomic (E-), not a tool bug (B-)

The call works correctly once the flat form is used — bounds are applied and the
echo is accurate. Nothing errors or silently no-ops; the only cost is the
pre-call wiki lookup needed to bridge "min/max corners" (title) → flat
`[minX,minY,minZ,maxX,maxY,maxZ]` (param) and to know the request shape differs
from the echoed `{min,max}` shape. Pure discoverability/consistency overhead, no
incorrect result — hence ergonomic.

## What it should do (any of)

- **Docs (named target):** add a `### volume.set_volume_bounds` section to the
  **`Plugins/PinWright/Docs/wiki-src/volume.md`** overlay (today a 3-line
  namespace blurb with no per-method sections — the same overlay
  `E-volume-set-extent-units-class-dependent-docs`,
  `E-volume-set-properties-class-mismatch-silent-noop`,
  `E-volume-type-filter-discovery`, `E-volume-create-name-vs-volumename`, and
  `E-volume-get-info-no-limit-spills` already want extended). State explicitly
  that `bounds` is a **flat 6-value array** `[minX,minY,minZ,maxX,maxY,maxZ]`
  (NOT two `{x,y,z}` corner objects despite the "min/max corners" title), and
  that the response echoes `bounds` as `{min:[x,y,z], max:[x,y,z]}` (a different
  shape than the request). One sentence removes the surprise.
- **Optionally (downstream, ergonomic):** accept the structured
  `{min:{x,y,z}, max:{x,y,z}}` (or `{min:[…],max:[…]}`) corner form in addition
  to the flat array, so the input matches the title and the echo — making the
  verb round-trip shapes. Docs alone is sufficient to close the friction.

## Friction evidence (this task — focus `volume.create_trigger_volume`, namespace `volume`, 18 calls, outcome `ergo`)

The tutorial-trigger blockout task ran clean (every call `ok`, no retries, no
errors). One of the two friction notes was this input shape, verbatim:
*"set_volume_bounds also takes a flat [minX,minY,minZ,maxX,maxY,maxZ] array
rather than min/max objects, which the wiki clarified before I called it."* The
call-log shows the author front-loaded a `set_volume_bounds` wiki-nav read
(`args_summary:"wiki-nav (read page)"`) before the real
`volume.set_volume_bounds {SecretAlcove bounds[-900,500,-50,-700,700,150]}`
call — i.e. the flat-array form had to be confirmed from the wiki first, which
is exactly the discoverability overhead this ticket records. Source-confirmed:
flat-array request `VolumeHandler.cpp:1401-1420`, structured `{min,max}` echo
`:1466-1470`.

## Distinct from

- `B-blocking-volume-no-brush-geometry` (IN-REVIEW) — a *behavioral* bug:
  `set_volume_bounds` (and create/extent) on a BlockingVolume produce no brush
  geometry / a stale echo. This ticket is purely the *request input shape* vs
  the title/echo (no behavioral claim) — it stands even after the brush-geometry
  bug is fixed.
- `E-volume-set-extent-units-class-dependent-docs` (OPEN) — `set_volume_extent`'s
  `extent` *units* (a different method/param). This is `set_volume_bounds`'s
  *shape*, not units.
- `E-volume-set-properties-class-mismatch-silent-noop` (OPEN, the judge's filing
  for this task) — `set_volume_properties` silently dropping class-mismatched
  props. Different method, different friction (silent no-op vs input shape).
- `E-chooser-set-cell-value-shape-undocumented` (OPEN) — same *flavor* (an
  undocumented per-call value shape forcing a source/wiki dive) in the chooser
  namespace; this is the volume-namespace instance for `set_volume_bounds`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the tutorial-trigger blockout task (focus `volume.create_trigger_volume`, namespace `volume`, 18 calls, outcome `ergo`; judge filed `E-volume-set-properties-class-mismatch-silent-noop` for the `set_volume_properties` no-op). Distinct PROCESS angle: `volume.set_volume_bounds` ingests `bounds` as a flat positional `[minX,minY,minZ,maxX,maxY,maxZ]` array (`VolumeHandler.cpp:1404`, read positionally `:1414-1420`) while its registered title says "Set volume bounds using min/max corners" and its echo returns a structured `{min:[x,y,z],max:[x,y,z]}` object (`:1466-1470`) — input/output shapes disagree, and a caller thinking in `{x,y,z}` corner objects (the shape every other volume verb uses for location/extent) must read the wiki to learn the flat 6-value form. Friction note verbatim: "set_volume_bounds also takes a flat [minX,minY,minZ,maxX,maxY,maxZ] array rather than min/max objects, which the wiki clarified before I called it." Call-log confirms a dedicated `set_volume_bounds` wiki-nav read preceded the real call. No call errored — pure discoverability/consistency overhead. Dedup: ripgrep over OPEN+closed board (`set_volume_bounds|min.?max|flat array|six-value`) — nearest neighbors `B-blocking-volume-no-brush-geometry` (behavioral brush-geometry bug, not input shape), `E-volume-set-extent-units-class-dependent-docs` (different method's units), `E-volume-set-properties-class-mismatch-silent-noop` (different method's no-op), `E-chooser-set-cell-value-shape-undocumented` (same flavor, chooser namespace) — all distinct (see "Distinct from"). Classified ERGONOMIC + `docs`: works correctly, the gap is the title/echo-vs-input shape mismatch and its undocumented flat form. Fix (downstream docs): add a `### volume.set_volume_bounds` section to `Plugins/PinWright/Docs/wiki-src/volume.md` stating the flat 6-value array input and the `{min,max}` echo; optionally accept structured corner objects too. Same overlay `E-volume-set-extent-units-class-dependent-docs`/`E-volume-set-properties-class-mismatch-silent-noop`/`E-volume-type-filter-discovery`/`E-volume-create-name-vs-volumename`/`E-volume-get-info-no-limit-spills` already target.
