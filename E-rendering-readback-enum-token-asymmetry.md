---
id: E-rendering-readback-enum-token-asymmetry
title: "rendering.get_project_settings reads enum fields back as raw identifiers (DetailTracing/AAM_TSR), not the friendly tokens you set — verify dead-ends in an engine-header dive"
status: OPEN
severity: Low
category: ergonomic
tags: [rendering, project-settings, get_project_settings, set_lumen_method, enum, readback, verify, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# rendering.get_project_settings reads enum fields back as raw identifiers, not the friendly tokens you set

The `rendering.*` write verbs accept and document **friendly enum tokens** on
input — `softwareRTMode: Detail | Global`, `reflectionMethod: None | Lumen |
ScreenSpace` (with `RT`/`SSR` aliases), `set_dynamic_gi_method method: LumenGI |
…` — but `rendering.get_project_settings` reads those same fields back as the
**raw serialized enum identifiers**: `LumenSoftwareTracingMode` reads as
`DetailTracing` (not `Detail`), `DefaultFeatureAntiAliasing` reads as `AAM_TSR`
(not `TSR`), `Reflections`/`DynamicGlobalIllumination` read as `Lumen`. So a
caller running the natural "set it, then read it back and confirm it matches"
loop gets two strings that **don't lexically match** what they set, with no
in-band mapping. The values are in fact correct — `DetailTracing` *is* `Detail`,
`AAM_TSR` *is* `TSR` — but nothing in the response or the wiki says so, so the
caller cannot tell a correct round-trip from a silent mismatch without leaving
the tool.

## Distinct from the existing rendering tickets on this method

- `B-rendering-lumen-software-mode-token-alias` (IN-REVIEW) is the **input**
  side: `set_lumen_method` *rejected* the documented `Detail`/`Global` tokens.
  That fix makes the friendly token **accepted on write**. This ticket is the
  **read** side: even with the input alias fixed (this task confirms `Detail`
  was accepted), `get_project_settings` still returns the *raw* identifier, so
  the write/read token spaces remain asymmetric and a verify step can't confirm
  equality in-band.
- `E-rendering-unknown-property-no-hint` (OPEN) is about `set_project_settings`
  rejecting an unknown *property name*. Orthogonal — this is about the *value*
  token form on a successful read.

## What it should do

Cheapest, docs-only (this ticket's recommended form): the
`docs/wiki-src/rendering.md` overlay already lists the friendly **input** tokens
under `### rendering.set_lumen_method` and `### rendering.set_dynamic_gi_method`,
but says nothing about what `get_project_settings` reads them back **as**. Add a
short "read-back token forms" note mapping the input tokens to the raw enum
identifiers the read returns, e.g.:

| field | set with | reads back as |
|-------|----------|---------------|
| `LumenSoftwareTracingMode` | `Detail` / `Global` | `DetailTracing` / `GlobalTracing` |
| `DefaultFeatureAntiAliasing` | `TSR` (`AAM_TSR`) | `AAM_TSR` |
| `Reflections` / `DynamicGlobalIllumination` | `Lumen` (`RT` alias) | `Lumen` |

That single overlay table lets a caller self-confirm a verify read without an
engine-header dive. (Optional, richer alternative: have `get_project_settings`
also emit a friendly-token projection alongside the raw value for known enum
fields — more code, not required to close the friction.)

## Evidence

From this task's friction note (`rendering` namespace, persist-Lumen-settings
end-to-end task, 15 calls, outcome clean): "the input softwareRTMode token
'Detail' round-trips/reads back as the full enum name 'DetailTracing' (and
DefaultFeatureAntiAliasing reads back as 'AAM_TSR' rather than 'TSR'), so I
**cross-checked RendererSettings.h** to confirm those serialized forms are the
requested Detail/TSR values rather than a mismatch." The task's step 5 was
explicitly "read the renderer settings back one more time and confirm … all
reflect what I just set" — a confirm-what-I-set intent that the read-back token
asymmetry forced off-tool to resolve. Verify-side reads in the call log:
`get_project_settings filter=lumen`, `filter=AntiAliasing`,
`filter=DynamicGlobalIllumination`, `filter=Reflections` (4 read-backs), each
returning a raw identifier the caller had to reconcile against the friendly
token it wrote.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the `rendering`
  persist-Lumen-settings fuzz task (15 calls, outcome clean, no tool error).
  `rendering.set_*` verbs accept/document friendly enum tokens (`Detail`, `TSR`,
  `RT`/`SSR`) but `rendering.get_project_settings` reads the same fields back as
  raw enum identifiers (`DetailTracing`, `AAM_TSR`, `Lumen`) with no in-band
  mapping, so the task's step-5 "confirm what I set" read-back forced an off-tool
  `RendererSettings.h` cross-check to prove round-trip equality rather than a
  mismatch. Distinct from `B-rendering-lumen-software-mode-token-alias`
  (input-side rejection, IN-REVIEW) and `E-rendering-unknown-property-no-hint`
  (unknown-property name hint, OPEN). Recommends a docs-only fix: a "read-back
  token forms" table in `docs/wiki-src/rendering.md` mapping input tokens to the
  raw identifiers the read returns; optional richer alternative is a
  friendly-token projection in the `get_project_settings` response.
