---
id: E-eqs-context-class-no-project-discovery-path
title: "eqs.set_context_class: when NO project context exists, the blueprint.create authoring route is undocumented on the EQS surface"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [ai, eqs, env-query, authoring, context, wiki, docs]
---

# `eqs.set_context_class` has no documented authoring route when no project context exists

**Re-scoped (see History #1→#3).** The ticket's original ask — a
discovery cross-link from the EQS surface to `asset.list` for *finding*
an existing project context class — has already shipped (the IN-REVIEW
[`E-eqs-builtin-token-discovery`](E-eqs-builtin-token-discovery.md)
`#2` added it to both the rejection error and `docs/wiki-src/eqs.md`):

- `EQSHandler.cpp` `set_context_class` rejection now reads
  "…(or pass a full UEnvQueryContext class path; project-defined
  contexts can be located via `asset.list`)."
- `docs/wiki-src/eqs.md` `### eqs.set_context_class` now says
  "A project-defined context … is not a built-in — pass its full
  asset/class path, which you can locate with `asset.list`."

That closes the *discovery* friction. What remains open — surfaced by
two further independent repros (`#2`, `#3`) — is the **authoring**
case: when there is **nothing to discover** because the project ships
no `UEnvQueryContext` at all.

## The still-open gap: authoring when none exists

The user names "the Player context" — a player-pawn context that the
engine does NOT provide (built-ins are only
`querier|item|navigationdata|blueprintbase`; `blueprintbase` resolves
the *abstract* `/Script/AIModule.EnvQueryContext_BlueprintBase` base,
not a usable Player context). `eqs.set_context_class {contextClass:
"player"}` is correctly rejected; the agent then runs the now-documented
`asset.list {class: EnvQueryContext}` discovery route — which correctly
returns **zero** on a host with no project context. At that point the
documented cross-link is *necessary but not sufficient*: there is
nothing to list, so the agent must **author** a context Blueprint.

That authoring route exists — `blueprint.create {parent:
/Script/AIModule.EnvQueryContext_BlueprintBase}` produces a concrete
subclass whose generated `_C` class path is then passed to
`eqs.set_context_class` — but it is **undocumented on the EQS surface**.
Both repros inferred it (one first wiki-nav'd to `ai.add_eqs_context`,
which only says "Deprecated alias for eqs.set_context_class": there is
no EQS-context-CREATE verb, only an assign verb). Confirmed absent:
`grep` of `docs/wiki-src/eqs.md` for `blueprint.create` /
`EnvQueryContext_BlueprintBase` returns nothing.

## Why this is a separate ticket from `E-eqs-builtin-token-discovery`

- **Different source of truth.** Built-in tokens are answerable by
  enumerating the C++ `ContextClassMap()` / enriching the rejection
  error (which `E-eqs-builtin-token-discovery` did). A *project* or
  *authored* context class is **not in any C++ map** — that fix cannot
  surface it, and it does not touch the authoring fallback.
- **Different fix.** A one-sentence *authoring-route* append to the
  existing `### eqs.set_context_class` note — not token enumeration.

## What it should do

**Fix:** Append to the existing `### eqs.set_context_class` note in
`docs/wiki-src/eqs.md`, after the "locate with `asset.list`" sentence:
"if none exists, author one with `blueprint.create {parent:
/Script/AIModule.EnvQueryContext_BlueprintBase}` and pass its generated
`_C` class path." This closes the loop for the case where discovery
returns nothing, so the agent does not have to infer the create route.
The discovery cross-link itself is already present (do not re-add it).

**Workaround:** if `asset.list {class: EnvQueryContext}` returns
nothing, `blueprint.create` a subclass of
`/Script/AIModule.EnvQueryContext_BlueprintBase` and pass its `_C` path.

## History
- `#4-reword-and-fix-authoring-route` `IN-REVIEW` developer — Re-scoped + fixed. The ticket's original ask (an `asset.list` discovery cross-link from the EQS surface) had already shipped via the IN-REVIEW `E-eqs-builtin-token-discovery #2` (present at `EQSHandler.cpp:563` and `docs/wiki-src/eqs.md:15`), so the title/body/"What it should do" were reworded to the genuinely-still-open delta surfaced by `#2`/`#3`: when NO project context exists, the `blueprint.create {parent: EnvQueryContext_BlueprintBase}` AUTHORING fallback is undocumented on the EQS surface (grep-confirmed absent from `docs/wiki-src/eqs.md`; the discovery cross-link returns 0 on a host shipping no project context). Fix: appended an authoring-route sentence to the existing `### eqs.set_context_class` H3 overlay note in `Docs/wiki-src/eqs.md` — author via `blueprint.create {parent: /Script/AIModule.EnvQueryContext_BlueprintBase}`, pass the generated `_C` class path, and noting `blueprintbase` resolves only the abstract base and that `ai.add_eqs_context` is a deprecated alias (no create verb). No production C++ behavior change (the rejection enrichment + discovery link from the sibling ticket are left intact). Regression test: `FEqsContextAuthoringDocTest` (`PinWright.infra.wiki_handler.MethodPage.EqsContextAuthoringRoute`) in `Source/PinWright/Private/Tests/Infra/TestEqsContextAuthoringDocs.cpp` renders the `eqs.set_context_class` page through the live `WikiHandler::RenderPage` and asserts the overlay-exclusive authoring markers (`blueprint.create`, `EnvQueryContext_BlueprintBase`, `_C`, `ai.add_eqs_context` + `deprecated alias`) survive — it fails if the append is reverted. Files: `Docs/wiki-src/eqs.md`, `Source/PinWright/Private/Tests/Infra/TestEqsContextAuthoringDocs.cpp`.
- `#3-third-repro-findcovernearplayer-author-context` `OPEN` reporter — Third independent EQS-authoring task reproducing the same "named context is not a built-in, and none exists in the project, so author one via `blueprint.create`" pattern, confirming it is task-independent. Seed `eqs.set_context_class`; authored `/Game/AI/EQS/EQS_FindCoverNearPlayer` (ActorsOfClass generator, Distance/score + Trace/filter tests; the user named "the Player context" — the player pawn — for both the generator `SearchCenter` and the Trace `Context`). Identical sequence to `#2`: `eqs.set_context_class {prop:SearchCenter, contextClass:"player"}` correctly rejected with the now-enriched error (`[INVALID_ARGUMENT] Unsupported EQS context class: player. Valid built-in names: blueprintbase, item, navigationdata, querier (or pass a full UEnvQueryContext class path; project-defined contexts can be located via asset.list).`) → the agent ran the documented discovery route `asset.list {class:EnvQueryContext, /Game, recursive}` which **correctly returned 0** (this throwaway host ships no project context) → so it had to **author** one: `blueprint.create {name:EQS_Context_Player, parent:/Script/AIModule.EnvQueryContext_BlueprintBase}`, then passed the generated `EQS_Context_Player_C` class path to `eqs.set_context_class` on both the generator `SearchCenter` and the Trace `Context` (both `ok:true`, round-trip-verified via `property.get`). Reinforces `#2`'s conclusion: the `asset.list` discovery cross-link (now in `docs/wiki-src/eqs.md`) is necessary-but-not-sufficient when nothing exists to discover, and the `blueprint.create {parent: EnvQueryContext_BlueprintBase}` authoring route remains **undocumented on the EQS surface** — same proposed addition (append to the `set_context_class` note: *"if none exists, author one with `blueprint.create {parent: EnvQueryContext_BlueprintBase}` and pass its `_C` class path"*). Friction note (verbatim): "there is NO built-in 'Player'/player-pawn EQS context (wiki + asset.list + engine source all confirm only querier/item/navigationdata/blueprintbase), so contextClass:'player' returned INVALID_ARGUMENT and I had to create an EnvQueryContext_BlueprintBase Blueprint via blueprint.create to obtain a non-default Player context to assign by path." Outcome clean (the single context `is_error` was this expected `player`-token rejection, recovered via the create route). Pure process friction surviving a "done" outcome.
- `#2-no-context-exists-author-via-blueprint-create` `OPEN` reporter — Second independent EQS-authoring task (`/Game/AI/EQS/FindShootingPositions`: SimpleGrid generator on Querier, Distance/score + Trace/filter tests; outcome clean), extending this ticket with a sub-case `#1` does not cover: **when NO project `EnvQueryContext` exists at all**. The user named "the Player context" (step 6); `eqs.set_context_class {contextClass:"player"}` correctly rejected it — and the error is now *enriched* (the IN-REVIEW `E-eqs-builtin-token-discovery #2` landed): `[INVALID_ARGUMENT] Unsupported EQS context class: player. Valid built-in names: blueprintbase, item, navigationdata, querier (or pass a full UEnvQueryContext class path; project-defined contexts can be located via asset.list).` The agent followed exactly the documented route this ticket proposes — `asset.list {/Game, recursive, class:EnvQueryContextBlueprint}` — but it correctly returned **none** (this throwaway host ships no project context). So the discovery cross-link (already in `docs/wiki-src/eqs.md`'s `### eqs.set_context_class` section: *"locate with asset.list"*) is necessary but **not sufficient**: there was nothing to discover, so the agent had to **author** a context Blueprint — `blueprint.create {parent: EnvQueryContext_BlueprintBase, name: EQC_Player}` — then pass its generated `_C` class path (`EQC_Player_C`) to `eqs.set_context_class` on both the Trace and Distance tests. Notably the agent first wiki-nav'd to `ai.add_eqs_context.md` hoping for a context-create verb, but that page says only *"Deprecated alias for eqs.set_context_class"* (it assigns, does not create) — so there is **no EQS-context-create verb**, and the `blueprint.create {parent: EnvQueryContext_BlueprintBase}` route is undocumented on the EQS surface. Proposed addition to the same `docs/wiki-src/eqs.md` `set_context_class` note: after the "locate with asset.list" sentence, add *"if none exists, author one with `blueprint.create {parent: EnvQueryContext_BlueprintBase}` and pass its `_C` class path"* — so the agent doesn't have to infer the creation route. Friction note (verbatim): "no built-in 'Player' EQS context exists (engine ships only querier/item/navigationdata/blueprintbase), so step 6 required creating a project EnvQueryContext_BlueprintBase Blueprint (EQC_Player) via blueprint.create and assigning its _C class path … the task's named 'Player' context is not a thing the MCP provides." Outcome clean otherwise (the one context `is_error` was this expected `player`-token rejection; recovered via the create route). Pure process friction surviving a "done" outcome.
- `#1-initial-audit` `OPEN` reporter — Surfaced by the same successful EQS-authoring task (`/Game/AI/EQS/FindFiringPosition`) that produced [`E-eqs-builtin-token-discovery`](E-eqs-builtin-token-discovery.md), but a distinct PROCESS angle the judge's ticket does not cover: the user's "Player context" is a PROJECT-defined `UEnvQueryContext` subclass (`BP_EQC_PlayerLocation`), not a built-in token, so enumerating the C++ `ContextClassMap()` / enriching the `Unsupported EQS context class` error cannot surface it. With no documented project-context-discovery route from the EQS surface, the agent spent two exploratory `asset.dump` calls (`EQS_StateTreeAI_PointsAround` then `BP_EQC_PlayerLocation`) reverse-engineering the class from a sibling asset before its first `eqs.set_context_class` mutation. The capability to do this directly already exists — `asset.list { filter.class: EnvQueryContext }` — but is never cross-linked from `docs/wiki-src/eqs.md` (a 2-sentence prelude with no per-method sections). Proposed fix is a usage cross-link, not token enumeration.
