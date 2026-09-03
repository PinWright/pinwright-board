---
id: E-niagara-create-emitter-description-says-referenced
title: "niagara.create_emitter's one-line description still sells the reference model the niagara page says the engine does not have"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, niagara, create_emitter, add_emitter, wiki-wrong, inheritance, follow-up]
encounters: 1
lastSeen: 2026-09-03T10:35:04Z
---

# The one leftover "referenced" in the emitter family, on the page that also says references do not exist

`B-niagara-add-emitter-snapshots-emitter-silently` established that a system never references an
emitter asset: `AddEmitterHandle` always duplicates, and the only question is whether the copy
keeps a **parent** link. That fix corrected `niagara.add_emitter`'s registration description in the
same wave — it now reads "Add one existing UNiagaraEmitter asset to a UNiagaraSystem **as an
inherited child of that asset**" (`NiagaraHandler.cpp:238`). Its sibling was missed.

`NiagaraHandler.cpp:965`:

```
REGISTER_RPC_HANDLER("niagara.create_emitter", "niagara",
    "Create an empty UNiagaraEmitter asset. Standalone emitters can be referenced by multiple
     systems via niagara.add_emitter.", ...)
```

`Docs/wiki-src/niagara.md:216`, the namespace page the same wave wrote:

> `niagara.add_emitter` takes an `emitterPath`, and the system ends up holding a **copy** of that
> emitter — **the engine has no mode in which a system references an emitter asset directly.**

A grep of the whole plugin for the reference model returns exactly these two lines, and they
contradict each other. This is the last one.

## Why the one-liner is the wrong place to leave it

The overlay corrects it two paragraphs down, so the *rendered method page* is self-repairing for
anyone who reads to the Notes. The registration string is not confined there — it is the text that
reaches `tools/list`, the `call("niagara")` namespace index, and
`Docs/rpc-method-reference.generated.md`, none of which carry the overlay. A caller skimming the
namespace index to pick a verb sees only "can be referenced by multiple systems".

The generated page shows both, in this order (`Saved/PinWright/wiki/niagara.create_emitter.md`):

```
Create an empty UNiagaraEmitter asset. Standalone emitters can be referenced by
multiple systems via niagara.add_emitter.
...
## Notes

Creates an empty standalone `UNiagaraEmitter`. Several systems can inherit from it through
`niagara.add_emitter` — each takes its own child copy that keeps a parent link back to this
asset, so an edit here reaches all of them once `niagara.refresh_emitter` merges it in.
```

"Reference" is the specific word the parent ticket blamed: `#2` there records that `emitterPath`
plus the old "Add one existing UNiagaraEmitter asset" sentence "both of which read as *reference*"
is why callers never went looking, and six stale handles shipped across one wave. Correcting the
verb that creates the relationship and leaving the verb that creates the asset asserting the
opposite is a half-landed fix, not a phrasing nit.

## Fix

One line. Replace the second sentence of the `create_emitter` registration description with the
inheritance wording the overlay already uses, e.g. "Several UNiagaraSystems can inherit from it via
niagara.add_emitter, each taking its own child copy that keeps a parent link back to this asset."
No behaviour change, no test change; `Docs/wiki-src/niagara.md`'s `### niagara.create_emitter`
section is already correct and needs no edit.

## Adjacent, checked, NOT a defect — `create_emitter` does not set `bIsInheritable` explicitly

Recorded here so it is not re-investigated. `niagara.create_emitter` does
`NewObject<UNiagaraEmitter>(...)` then `UNiagaraEmitterFactoryNew::InitializeEmitter(NiagaraEmitter,
false)` (`NiagaraHandler.cpp:1009,1017`) and never writes an inheritability flag. That is correct
and version-safe on every supported engine:

- **5.4–5.8** — `UNiagaraEmitter::bIsInheritable` carries an in-class initializer `= true`
  (5.8 `Niagara/Classes/NiagaraEmitter.h:793`; same line present at 5.4:723, 5.5:735, 5.6:741,
  5.7:751). `AddEmitterHandle` strips the parent only on `bIsInheritable == false`
  (5.8 `NiagaraSystem.cpp:3023`).
- **5.3** — no `bIsInheritable` exists; the rule is spelled `TemplateSpecification`, and it is set
  to `None` **twice** on this path without PinWright doing anything: by the `UNiagaraEmitter`
  constructor (`NiagaraEmitter.cpp:144`) and again inside `InitializeEmitter` itself
  (`NiagaraEmitterFactoryNew.cpp:231`). `AddEmitterHandle` there strips the parent only for
  `Template` / `Behavior` (`NiagaraSystem.cpp:2304`).
- The editor's own empty-emitter path is byte-identical to PinWright's — `FactoryCreateNew`'s
  else-branch is `NewObject<UNiagaraEmitter>` + `InitializeEmitter`, with no explicit flag
  (5.8 `NiagaraEmitterFactoryNew.cpp:189-191`). The explicit `bIsInheritable = true` at
  `:183` sits in the *copy-from-existing* branch, where the source could have carried `false`;
  PinWright has no such branch.

So there is no version hole to harden. Writing the flag explicitly would need a `#if` shim for a
field that does not exist on 5.3, and would re-introduce the duplicated inheritability rule that
`add_emitter`'s own comment (`NiagaraHandler.cpp:337-339`) deliberately refuses to keep: "the engine
spells the inheritability rule differently on different versions … a duplicated rule would drift
silently." No ticket filed for it.

## Cross-ref

- `B-niagara-add-emitter-snapshots-emitter-silently` — parent. This is the leftover of its
  "the reporter's 'the wiki says the opposite' is closed" claim.
- **Correction to that ticket's closing note, for whoever picks it up:** it says the recipe "Never
  share one emitter asset between two systems — editing it would change both" "lives in the host
  project's `CLAUDE.md`". It does not, and there is nothing there to fix. A grep of every checkout
  under `X:\src\unreal` (all six `EAContentExamples5x`, the four fuzz hosts, `unreal-fpv`,
  `unreal-fpv-dev`, `unreal-fpv-new`, `pinwright-ue`) finds that sentence in no tracked file. It
  exists only in a per-session Claude scratchpad,
  `%LOCALAPPDATA%\Temp\claude\X--src-unreal-EAContentExamples58\b495bc11-7b3d-4175-82bb-71f14975e6d1\scratchpad\NIAGARA_RECIPE.md:39-40`
  — an ephemeral hand-off document a VFX lead wrote for that wave's fixers, outside version control
  and already gone from any future session. Nothing outside this plugin needs a fix.

## History
- `#1-follow-up-from-add-emitter-fix` `OPEN` reporter — Follow-up research on the
  `B-niagara-add-emitter-snapshots-emitter-silently` fix wave, source-read only, no editor. Found
  that wave corrected `add_emitter`'s registration description but not `create_emitter`'s, leaving
  the plugin asserting the reference model at `NiagaraHandler.cpp:965` while
  `Docs/wiki-src/niagara.md:216` states the engine has no such mode; a plugin-wide grep for the
  reference model returns exactly those two lines. Low because the impact is pure docs friction on
  a rare verb (agents reach `create_emitter` far less than `asset.duplicate` of a stock template)
  and the method page's own Notes already correct it — the reach is the namespace index and
  `tools/list`, where the overlay does not follow. Also checked and cleared the adjacent
  "`bIsInheritable` is left implicit" concern against 5.3–5.8 engine source (section above); no
  ticket filed for it. Also corrected the parent's misattribution of the "never share one emitter"
  recipe to a host `CLAUDE.md` — it is a temp session scratchpad, not a repo file, so there is no
  host doc to route.
