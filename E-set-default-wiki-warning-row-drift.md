---
id: E-set-default-wiki-warning-row-drift
title: "blueprint.set_default wiki documents a `warning` response field the handler never emits"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, set-default, docs, doc-drift, wiki, response-shape]
encounters: 1
---

# `blueprint.set_default` wiki documents a `warning` response field the handler never emits

The `### blueprint.set_default` overlay in `docs/wiki-src/blueprint.md` has a
"#### Post-compile CDO readback" section whose response-field table lists a
`warning` row:

> `warning` — Present if the value could not be read back — indicates the value
> may not have survived compilation (e.g. the property was removed or the BP is
> in error state).

and a follow-up sentence:

> Always inspect `warning` in the response. Its presence means the write may
> have been silently discarded.

The handler emits no such field. In `Source/PinWright/Private/Handlers/Blueprint/BlueprintPropertyHandler.cpp`
the `blueprint.set_default` block (lines 391-543) builds its response at lines
534-540 and sets only `propertyName` (535), `blueprintPath` (536), `saved`
(537), and conditionally `value` (538-539, only when the readback
`ExportPropertyToJsonValue` returns a valid value — line 532), plus
`AddAssetVerification` (540). There is no `warning` key anywhere in the handler.
The only `warning` field in this file belongs to the sibling
`blueprint.add_variable` handler (line 243), a different method.

Worse than a phantom field, the row's premise is inverted by the handler's
current design. After today's persistence rewrite (see
`B-blueprint-set-default-not-persisted`, IN-REVIEW) a readback/re-apply failure
is a **hard error**, not a survivable soft warning: a post-compile property
re-resolve failure returns `PROPERTY_NOT_FOUND` (line 514) and a re-apply
failure returns `CONVERSION_FAILED` (line 521). The one soft path — an
export/readback that yields an invalid `FJsonValue` — simply **omits** the
`value` field (line 538); it still emits no `warning`. So an agent reading the
docs would (a) build handling for a field that can never appear and (b) assume
readback failure is survivable when it actually errors out.

The `saved` row in the same table *is* accurate — it was added today alongside
the persistence fix — so the section is not wholesale wrong, only the `warning`
row and its "Always inspect `warning`" follow-up are drift. The generated wiki
page derives from this overlay at editor launch, so fixing the overlay fixes the
generated page too.

**Fix:** delete the `warning` table row and the "Always inspect `warning` in the
response…" sentence from the `### blueprint.set_default` overlay in
`docs/wiki-src/blueprint.md`. Do NOT implement a soft-warning code path — that
would contradict the rewrite's intentional fail-loud design (`PROPERTY_NOT_FOUND`
/ `CONVERSION_FAILED` on failure). Leave the `value` and `saved` rows intact.

## History
- `#1-doc-drift-confirmed` `OPEN` reporter — Verified against source: `blueprint.set_default` (`BlueprintPropertyHandler.cpp:391-543`) sets only `propertyName`/`blueprintPath`/`saved`/`value` (lines 534-540) plus asset-verification; no `warning` key. Re-resolve failure → `PROPERTY_NOT_FOUND` (514), re-apply failure → `CONVERSION_FAILED` (521), invalid readback → `value` omitted (538) with no warning. The wiki's `warning` row + "Always inspect `warning`" line (`docs/wiki-src/blueprint.md` "#### Post-compile CDO readback") document a never-emitted field and wrongly frame a hard-error path as survivable. The `warning` field exists only on the sibling `add_variable` handler (line 243). Fix = delete the `warning` row and its follow-up sentence; keep the accurate `value`/`saved` rows. Not a duplicate of `E-set-default-none-clear-conversion-failed` (null-sentinel clear) or `B-blueprint-set-default-not-persisted` (the persistence rewrite this drift trails).
