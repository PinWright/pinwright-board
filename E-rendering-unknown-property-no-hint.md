---
id: E-rendering-unknown-property-no-hint
title: "rendering.set_project_settings unknown_property is a dead-end — no near-match hint, no pointer to get_project_settings"
status: OPEN
severity: Low
category: ergonomic
tags: [rendering, project-settings, set_project_settings, error-messages, error-hint, discovery, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# rendering.set_project_settings unknown_property dead-ends the caller instead of steering it

When `rendering.set_project_settings` cannot resolve a UPROPERTY name, it
returns a flat dead-end rejection:

> `{ rejected: [{ name: "bSupportGPUSkinCache", reason: "unknown_property" }] }`

The reason is accurate but offers **no recovery path**: it does not echo any
near-match UPROPERTY name, and it does not point at the sibling RPC
(`rendering.get_project_settings`) that already enumerates every valid
`CPF_Config | CPF_GlobalConfig` field on `URendererSettings`. The data needed to
suggest the right name is right there in the same handler's read path, but the
write path withholds it. Faced with that wall, the caller has no in-band signal
of the correct spelling and must leave the MCP entirely to recover.

## Distinct from the judge-filed bug on this task

The per-finding judge filed `B-rendering-lumen-software-mode-token-alias` for a
*different* friction point in the same task (the `softwareRTMode` enum-token
alias gap). This ticket is the **error-ergonomics** angle on the second
friction point: even with a perfectly correct `unknown_property` validation, a
caller who passes a legacy / renamed / typo'd field name should get a hint, not
a wall. Here the property genuinely does not exist on UE 5.7's
`URendererSettings` (the field was renamed `bSupportGPUSkinCache` →
`bSupportSkinCacheShaders`), so the rejection itself is correct — the friction
is purely that the dead-end forced an off-tool recovery.

## What it should do

On an `unknown_property` rejection, enrich the rejection with a recovery hint,
e.g.:

> `{ name: "bSupportGPUSkinCache", reason: "unknown_property",
>   didYouMean: ["bSupportSkinCacheShaders"],
>   hint: "call rendering.get_project_settings to list valid property names" }`

Cheapest useful form: the handler already walks
`URendererSettings::PropertyIterator()` for `get_project_settings` — run the
rejected name through the same enumerated set and surface the top 1–3 near
matches (case-insensitive substring / edit-distance on the UPROPERTY name) plus
the pointer to `get_project_settings`. Even with zero near matches, the pointer
to the read RPC alone would have kept the recovery in-band. This is the same
error-hint shape the board has already adopted: `E-make-struct-error-hint`
(DONE — `make<>` suggests the native `Make*` function), `E-bpir-createwidget-pin-hint`
(DONE — pin error hints the correct pin), and `E-add-metasound-node-error-no-hint`
(OPEN — `NODE_CLASS_NOT_FOUND` should list near matches + point at the search RPC).

## Docs angle

Complementary, cheaper still: the `docs/wiki-src/rendering.md` overlay already
documents the `unknown_property` rejection (the `### rendering.set_project_settings`
section) but lists no renamed/legacy field aliases. A short "common renamed
fields" note there (e.g. `bSupportGPUSkinCache` → `bSupportSkinCacheShaders`
in UE 5.6+, matching the page's existing `RayTraced`-token and
`finalGatherQuality`-CVar-only caveats) would let a caller self-correct from the
wiki without an engine-header dive. The wiki edit is a downstream wiki process,
not part of this ticket.

## Evidence

From this task's friction note (`rendering` namespace, persist-Lumen-settings
task): "bSupportGPUSkinCache was rejected as unknown_property because in UE 5.7
the actual URendererSettings field is bSupportSkinCacheShaders (CVar
r.SkinCache.CompileShaders); I confirmed this in the engine header and applied
the correct property." Call log: `rendering.set_project_settings
{bVirtualTextures, bSupportGPUSkinCache}` rejected `bSupportGPUSkinCache`
(`unknown_property`), then a corrected `rendering.set_project_settings
{bSupportSkinCacheShaders}` succeeded — a misuse-then-correct pair where the
correction required an **off-tool engine-header lookup** that an in-band
near-match hint (or a wiki alias note) would have eliminated.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the `rendering`
  persist-Lumen-settings fuzz task: `rendering.set_project_settings` rejected
  `bSupportGPUSkinCache` with a bare `unknown_property` (no near-match list, no
  pointer to `get_project_settings`), forcing an off-tool engine-header dive to
  discover the renamed UE 5.7 field `bSupportSkinCacheShaders` before the
  corrected write succeeded. Distinct from the judge-filed enum-token bug
  `B-rendering-lumen-software-mode-token-alias` on the same task. Follows the
  established error-hint precedent: `E-make-struct-error-hint`,
  `E-bpir-createwidget-pin-hint` (both DONE), `E-add-metasound-node-error-no-hint`
  (OPEN). Cheapest fix reuses the existing `get_project_settings` property walk
  for near-matches; complementary docs fix names `docs/wiki-src/rendering.md`
  for a "common renamed fields" alias note.
