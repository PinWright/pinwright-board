---
id: E-effect-create-niagara-systempath-discovery
title: "effect.create_* Niagara wrappers force a 4-call systemPath discovery dance with no default or documented example"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs]
---

# effect.create_* Niagara wrappers force a 4-call systemPath discovery dance with no default or documented example

The five `effect.create_*` Niagara wrappers — `create_volumetric_fog`,
`create_impact_effect`, `create_environment_effect`, `create_particle_trail`,
`create_niagara_ribbon` — are byte-identical thin shims around
`CreateNiagaraEffectHelper` (`Handlers/VFX/EffectHandler.cpp:1180-1240`). Each
declares `systemPath` as `RPC_PARAM_REQ` and passes an **empty** `DefaultSystemPath`
(`FString()`) into the helper, which then hard-rejects an empty path with
`INVALID_ARGUMENT` ("systemPath is required ... e.g. /Game/Effects/MySystem").

Two ergonomic problems compound:

1. **No default / fallback system.** The helper has a `DefaultSystemPath`
   parameter built for exactly this, but every wrapper passes empty. There is no
   fallback to a known-good engine/template Niagara system, so a caller who just
   wants "any sensible particle effect here" (a very common preview/blockout
   intent) must first go *find* a valid `NiagaraSystem` asset path on their own.

2. **No discovery affordance and no documented example.** Neither the param
   description (`"Asset path to the Niagara system"`) nor the `effect` wiki
   overlay (`docs/wiki-src/effect.md`) names a single concrete engine system path
   the caller could paste. The error message's `e.g. /Game/Effects/MySystem` is a
   *fictional* path, not a real asset — following it verbatim fails. So the
   caller is pushed into a manual asset-registry hunt.

In this task the user explicitly said "I don't have a specific one in mind — pick
a reasonable existing engine/template Niagara system." Satisfying that one
sentence cost the agent **4 discovery calls before either effect-create**:
`asset.find` (guessed, not-found → suggested alternatives), `asset.search_assets`
(classNames=[NiagaraSystem], limit 50), then two `asset.exists` probes
(SimpleExplosion, DefaultSystem) to validate candidates. Only then could
`create_volumetric_fog @origin` and `create_impact_effect [200,200,100]` run.
The named verb (`volumetric_fog`, `impact_effect`) contributes nothing to picking
a fitting system — it is purely a label on the spawned actor — which makes the
required `systemPath` feel surprising for methods that sound self-describing.

**Workaround:** Discover a valid `NiagaraSystem` path first (e.g.
`asset.search_assets {"classNames":["NiagaraSystem"]}`), then pass it as
`systemPath`.

**Fix (docs-first, downstream wiki process — not done here):** In
`docs/wiki-src/effect.md`, add a short `## Niagara effect systems` section under
the `create_*` methods that (a) states these wrappers require a real
`systemPath` and only set the actor label from the verb, and (b) names one or two
**concrete, real** engine/template Niagara system paths a caller can paste
without a registry hunt (verify they ship with a stock UE install before
listing). Also worth a follow-up *feature* angle: give `CreateNiagaraEffectHelper`
a real fallback — when `systemPath` is omitted, resolve a known engine default
(or the first `NiagaraSystem` in the registry) instead of erroring — so the
"just give me a reasonable effect" intent needs zero discovery calls. Per
`effect.md`'s own guidance these are preview helpers, so a sensible default fits
the namespace's purpose.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced as PROCESS friction in a preview-lighting fuzz task (seed `effect.create_dynamic_light`). User asked to "pick a reasonable existing engine/template Niagara system" for `effect.create_volumetric_fog` and `effect.create_impact_effect`; with no default and no documented example path, the agent spent 4 discovery calls (`asset.find` not-found, `asset.search_assets` classNames=[NiagaraSystem], 2× `asset.exists`) before either create. Both wrappers pass empty `DefaultSystemPath` at `EffectHandler.cpp:1200,1239`; the error's `e.g. /Game/Effects/MySystem` is a fictional path. Distinct from the judge-filed tool bug `B-actor-find-by-class-short-name-fails` (that is the find_by_class short-name defect; this is the systemPath discovery burden). Page to improve: `docs/wiki-src/effect.md`.
- `#2-discovery-affordance` `IN-REVIEW` developer — Closed the discovery gap with a docs+error fix (not the risky auto-default-fallback feature, which the adversarial lens flagged as non-deterministic/project-dependent — deliberately left as a future option). Verified a real stock NiagaraSystem ships and is loadable: `C:\UE_5.7\...\Plugins\FX\Niagara\Content\DefaultAssets\DefaultSystem.uasset` (class `/Script/Niagara.NiagaraSystem`), addressable as `/Niagara/DefaultAssets/DefaultSystem` (Niagara plugin mounts `Content/` at `/Niagara/`), which these wrappers can always load since they already require the Niagara plugin. Changes: (1) `Handlers/VFX/EffectHandler.cpp` — added `GStockNiagaraSystemPath` constant and rewrote the missing-systemPath `INVALID_ARGUMENT` message to drop the fictional `/Game/Effects/MySystem`, name the real stock path, and point at `asset.search_assets {"classNames":["NiagaraSystem"]}` for project-specific discovery; (2) `docs/wiki-src/effect.md` — added a `## Niagara effect systems` section stating the wrappers require a real `systemPath`, that the verb only labels the actor, naming the stock pasteable path, and giving the search/exists discovery recipe. Regression test: `Tests/Assets/TestVFXHandlers.cpp` `effect.create_niagara.MissingSystemPathErrorNamesRealAsset` invokes `create_volumetric_fog` (shared `CreateNiagaraEffectHelper`) with no `systemPath` via `InvokeHandlerWithCapture` and asserts the error is `INVALID_ARGUMENT`, no longer contains `/Game/Effects/MySystem`, and does contain `/Niagara/DefaultAssets/DefaultSystem` — fails if the message is reverted.
- `#3-evidence-environment-effect` `IN-REVIEW` reporter — Cross-task aggregation
  confirming the same discovery burden on a sibling wrapper, now post-fix. The
  torch-lit-dungeon preview-lighting struggle audit (namespace `effect`; outcome
  `clean`; no judge filing) called **`effect.create_environment_effect`** for
  "a little atmospheric haze near the corridor center." Same shape as `#1`: with no
  default and the `systemPath` declared required, the agent first ran
  `asset.search_assets {classNames:[NiagaraSystem], under:/Game/ExampleContent/Niagara}`
  to discover a valid system, picked `3DGasLitSmoke`, then ran the create — one
  extra discovery call before the effect. Milder than `#1`'s 4-call dance: the
  wiki page had **already flagged `systemPath` as required** (the agent read it
  during wiki-nav), so the agent went straight to a single targeted
  `asset.search_assets` rather than the `asset.find` not-found → search → 2×
  `asset.exists` probe sequence. That is the docs half of `#2` already paying off
  (the requirement is now discoverable up front); the residual is the *no-default*
  feature angle `#2` deliberately deferred — a "just give me reasonable haze here"
  intent still costs one registry hunt. Friction note (verbatim): *"environment
  effect needed a Niagara systemPath which the wiki flagged as required, so I
  discovered a valid one via asset.search_assets — smooth overall."* Confirms the
  systemPath burden spans `create_environment_effect` too (not just the `#1`
  `create_volumetric_fog`/`create_impact_effect` pair) and that, once the
  shipped docs/error fix lands, the friction degrades from a 4-call dance to a
  single search — the remaining cost is exactly the optional auto-default feature
  `#2` left as a future option.
