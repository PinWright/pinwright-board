---
id: B-nir-renderer-omits-default-valued-properties
title: "nir.txt renderer blocks omit properties sitting at the class default, so an absent bool reads as false when it is actually true — bSubImageBlend defaults to true and inverts"
status: IN-REVIEW
severity: Medium
category: bug
tags: [nir, asset-dump, niagara, renderer, defaults, silent-omission, boolean-inversion, sprite-renderer, subuv]
encounters: 1
lastSeen: 2026-09-03T00:45:00+05:00
---

# `nir.txt` omits default-valued renderer properties, and the reader cannot tell "absent" from "false"

## Symptom

A `renderer NiagaraSpriteRendererProperties @0 enabled { ... }` block in `nir.txt` lists
only the properties whose value **differs from the class default**. Properties at their
default are omitted entirely, with no marker saying so.

That is fine for a human skimming, and wrong for anything that parses it, because the
omission is indistinguishable from "false" for a bool. It bites hardest on
`UNiagaraSpriteRendererProperties::bSubImageBlend`, whose class default in UE 5.8 is
**`true`**:

| what `nir.txt` shows | actual value |
|---|---|
| `bSubImageBlend: false` present | `false` (explicitly overridden) |
| line absent entirely | **`true`** (class default) |

So the naive reading — "the line isn't there, so the flag isn't set" — returns the exact
opposite of the truth on the one renderer flag most likely to be audited across a package.

## Measurement

Two controls on emitters neither of which had this property written by the caller, both
read back with `property.get` on the renderer subobject:

```
E_FPS_MuzzleAR_Petals   nir: no bSubImageBlend line
  property.get bSubImageBlend -> true      # absent == true

E_ImpMetal_Sparks       nir: "bSubImageBlend: false"
  property.get bSubImageBlend -> false     # printed == explicitly false
```

`SubImageSize` behaves the same way: absent means the default `(1,1)`, and only a
non-default size is printed. That case happens to be harmless because the default is the
"unset" value a reader would guess, which is exactly why the `bSubImageBlend` inversion
goes unnoticed — one property in the same block confirms the reader's assumption while the
other silently contradicts it.

## Why it matters

This is a read-surface defect on the sidecar the docs steer bulk auditing toward.
`asset.dump_folder` + parse `nir.txt` is the recommended way to answer "which assets in
this subtree have property X", precisely because it avoids an RPC per asset. A boolean that
inverts on omission makes that answer wrong in the confident direction: the audit reports a
clean list of assets needing a fix, and the assets are already correct.

Concretely, in this session: an audit of 57 emitters under `/Game/FPS/VFX/Emitters`
reported 11 sprite renderers as `bSubImageBlend = false` and needing repair. All 11 were
already `true`. The wrong list was reported to the requesting agent as a work estimate
before `property.get` on two controls showed the parse was inverted. The cost was bounded
here only because the number looked implausible and got checked; a smaller discrepancy
would have produced eleven pointless remove/add/compile/save cycles on a shared editor, or
a false "already correct" verdict in the other direction.

## Expected

Any of these closes it; the first is the smallest:

- Print every reflected property in the renderer block, defaults included. The block is
  already ~10 lines; completeness costs little and removes the ambiguity entirely.
- Or keep the omission and emit a marker the parser can key on — a `defaults omitted:` list
  of names, or a `@default` suffix on printed values mirroring the `static ... @source
  override @default N` convention the **module** blocks already use. Module static-switch
  lines get `@source override @default 0.0`; renderer property lines get nothing, so the two
  halves of the same file disagree about how much a reader is told.
- Or document the rule in [`niagara.dump-files`](niagara.dump-files.md), which currently
  describes `nir.txt` as carrying "renderers" without saying the listing is filtered.

The third alone is weak: it leaves every existing parser wrong until its author re-reads the
page.

## Workaround

Do not infer a renderer property's value from its absence in `nir.txt`. For a bool, treat
absence as "unknown" and confirm with
`property.get {objectPath: "<emitter>.<emitter>:NiagaraSpriteRendererProperties_0",
propertyName: "..."}`, which returns the resolved value. When auditing in bulk, establish
the polarity first with a known-good and a known-bad control asset — one where the line is
printed and one where it is not — rather than assuming which way round it goes.

severity rationale: impact=silently inverted boolean on the recommended bulk-read path,
producing confidently wrong audit results in either direction x reach=every `nir.txt`
renderer block, i.e. every Niagara emitter in any dumped subtree, but only for properties
whose class default is not the "unset-looking" value -> Medium.

## Fix

Confirmed against source before changing anything. `NIRDecompiler::BuildReflectedFields`
(`Source/PinWright/Private/NIR/NIRDecompiler.cpp:555`) passes the class CDO as the `Default`
argument, and `FIrTextUtils::AppendReflectedFields`
(`Source/PinWright/Private/IrCore/IrTextUtils.cpp`) `continue`d on
`Property->Identical(instance, default)`. `UNiagaraSpriteRendererProperties::bSubImageBlend`
is initialised `true` in its constructor (`NiagaraSpriteRendererProperties.cpp:57`, UE 5.8),
so the reporter's measured polarity is right and the class-default inference holds. The same
inversion applies, on `UNiagaraSpriteRendererProperties` alone, to `bCastShadows`
(`uint8 : 1 = 1`), `bSortOnlyWhenTranslucent` (`true` in the ctor), `bIncludeInHitProxy`
(`= 1`), the base class's `bIsEnabled` and `bAllowInCullProxies` (both `true` in
`UNiagaraRendererProperties()`), `SubImageSize` ((1,1)), `PivotInUVSpace` ((0.5,0.5)),
`MaxCameraDistance` (1000), `PixelCoverageBlend` (1.0), and the ~28
`FNiagaraVariableAttributeBinding` renderer bindings — every one of which was invisible at
default. The reporter also asked about `bEnableCameraDistanceCulling` and the sort flags:
those are safe. `bEnableCameraDistanceCulling` has no initialiser (0), `SortMode` defaults to
`None` (0), and `SortPrecision` / `Alignment` / `FacingMode` / `MotionVectorSetting` /
`PixelCoverageMode` / `GpuTranslucentLatency` all default to their enum's 0 entry, so a reader
guessing "absent means the zero entry" was already right on them and still is.
`bGpuLowLatencyTranslucency` is not on the properties asset at all — it is a runtime field on
`FNiagaraRendererSprites`/`Meshes`, derived from `GpuTranslucentLatency`, and never appears in
`nir.txt`.

**Design: the zero-default rule, not "print everything".** Omission stays — it matches the
rest of the IR family (MGIR/AGIR/BTIR/SCIR/PCGIR all call the same helper with a CDO) — but
what may be omitted is narrowed: a default-valued property is dropped **only** when that
default is the type's zero value (false / 0 / empty / null / identity). A default that is not
the zero value prints, tagged ` @default`. Absence then means exactly one thing, and it is the
thing every reader already assumes, so existing naive parsers become correct rather than merely
warned. Rejected alternatives: printing every reflected property (drops the IR's value as a
readable summary and diverges from the family); a `defaults omitted: [names]` marker (still
requires a per-class default table to get the value, which is what the reader actually wants);
documentation alone (the reporter's own objection — every existing parser stays wrong).

Two secondary decisions worth reviewing: (1) the ` @default` tag is kept rather than emitting a
bare line, because plain always-emit would destroy the authored-vs-inherited distinction that
omission used to encode, and the suffix matches NIR's existing `@scope` / `@source override
@default <v>` / `@(x, y)` convention. (2) A line forced out by the rule is formatted against a
**nullptr** default so the whole value prints; formatting it against the archetype yields the
empty diff `"()"` for structs. Overridden lines keep the archetype diff so struct sub-fields
still suppress per `B-decompile-struct-subfield-dropped`.

Output grows: renderer blocks go from ~6 lines to ~30-40, mostly renderer bindings. Measured
against the host project's committed dump mirror that is 355 renderer blocks over a 37.9 MB
`nir.txt` corpus, i.e. roughly +3%. The next `asset.dump_folder` sweep will produce a large but
expected diff in any committed mirror.

Files changed (all under `Plugins/PinWright/`):
- `Source/PinWright/Public/IrCore/IrTextUtils.h` — new `FReflectedFieldEmitOptions::bEmitNonZeroDefaults`, default **false** so no other IR's output moves.
- `Source/PinWright/Private/IrCore/IrTextUtils.cpp` — anonymous-namespace `IsAtTypeZeroValue` (engine `FDefaultConstructedPropertyElement`; `Identical(A, nullptr)` is NOT a substitute because `UScriptStruct::CompareScriptStruct` answers "not identical" for a null comparand), and the `AppendReflectedFields` loop honouring the flag.
- `Source/PinWright/Private/NIR/NIRDecompiler.cpp` — `BuildReflectedFields` sets the flag; covers `renderer` and `simStage` blocks.
- `Source/PinWright/Private/Tests/Niagara/TestNIRDecompiler.cpp` — new `PinWright.niagara.decompile_nir.RendererDefaultValuedProperty`.
- `docs/wiki-src/niagara.nir.md` — new "Reflected blocks: the zero-default rule" section with the three-state table.
- `docs/wiki-src/niagara.dump-files.md` — the `nir.txt` bullet now says the listing is filtered (the reporter's third expectation).
- `docs/ir-authoring.md` — family-wide statement of the rule plus adoption status (NIR only; adopting it in a compile-input IR needs the parser to strip ` @default` first).

Not compiled and not run — a separate compile pass follows.

**Reviewer verification.** Run `PinWright.niagara.decompile_nir.RendererDefaultValuedProperty`;
it builds a transient emitter, adds a `UNiagaraSpriteRendererProperties` with `bSubImageBlend`
left at its default, and asserts `bSubImageBlend: true @default`,
`SubImageSize: "(X=1.000000,Y=1.000000)" @default`, an unmarked `MacroUVRadius: 12.5`
override, and that the zero-defaulted `MinFacingCameraBlendDistance` is still absent. Then
re-dump one real sprite emitter and confirm on the live asset that the two controls named in
the Measurement section above now read the same through `nir.txt` as through `property.get`.
Also confirm no other IR's text moved: `PinWright.core.ir_text.reflected_property.*`,
`MGIR.*`, `AGIR.*`, `BTIR.*`, `SCIR.*`, `PCGIR.*` should be unaffected because the flag
defaults off (`DefaultIdenticalValue` in the IrCore fixture is `int32 = 0`, a zero-valued
default, so its existing omission assertion holds either way).

## History
- `#1-filed` `OPEN` reporter — Found auditing `SubImageSize` / `bSubImageBlend` across 57 emitters under `/Game/FPS/VFX/Emitters` on EAContentExamples58 (UE 5.8, shared editor, port 27145) while repairing flipbook renderers left at 1x1. The parse was a plain `'bSubImageBlend: true' in body` test over the `renderer ... { }` block, which never matched because the string never appears — the true case is expressed by absence. Corrected polarity is measured, not inferred: the two `property.get` controls quoted above are on emitters whose renderer I never wrote to, one printing the line and one omitting it, returning `false` and `true` respectively. The class-default reading (`bSubImageBlend` defaults to `true`) follows from that pair plus the omission rule; I did **not** read `NiagaraSpriteRendererProperties.h` to confirm the UPROPERTY initialiser, so a verifier should, and should also check whether other renderer bools (`bEnableCameraDistanceCulling`, `bGpuLowLatencyTranslucency`, sort flags) have non-false defaults and therefore invert the same way. `SubImageSize` was verified to follow the identical omit-at-default rule, defaulting to `(1,1)`.
- `#2-zero-default-rule` `IN-REVIEW` developer — Verified TRUE by reading source: `NIRDecompiler::BuildReflectedFields` passes the CDO and `FIrTextUtils::AppendReflectedFields` skipped every default-identical property; `UNiagaraSpriteRendererProperties::bSubImageBlend` is `true` in the constructor (`NiagaraSpriteRendererProperties.cpp:57`, UE 5.8), confirming the reporter's measured polarity and the class-default reading they explicitly left for a verifier. Also checked the other renderer bools they flagged: `bCastShadows` and `bSortOnlyWhenTranslucent` invert the same way; `bEnableCameraDistanceCulling` and `SortMode` default to zero and never did. Fixed by narrowing what omission is allowed to cover rather than by printing everything — `FReflectedFieldEmitOptions::bEmitNonZeroDefaults` (off for every IR but NIR) omits a default-valued property only when that default IS the type's zero value, and prints the rest with a ` @default` tag. Absence now has one meaning. See the `## Fix` section for the rejected alternatives, the two secondary decisions, the ~+3% dump-size measurement, and reviewer steps.
