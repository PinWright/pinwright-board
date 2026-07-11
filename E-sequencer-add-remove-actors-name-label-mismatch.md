---
id: E-sequencer-add-remove-actors-name-label-mismatch
title: "sequencer binding verbs split the actor-identity domain: add_actors/add_actor accept the internal object NAME (FindActorByName) but store + report + match bindings by the display LABEL, so get_bindings/remove_actors reject the very name string add_actors just accepted with 'Actor not found in sequence bindings'"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [sequencer, add_actors, add_actor, remove_actors, get_bindings, actorname, internal-name, display-label, cross-method-consistency, name-label-mismatch]
encounters: 1
lastSeen: 2026-07-11T05:25:51.6633990+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T14:23:44.5921829+03:00
---

# The sequencer binding family accepts the internal actor NAME on the add side but only the display LABEL on the read/remove side — the identical argument that binds an actor cannot unbind it

`sequencer.add_actors` (and its single-actor twin `sequencer.add_actor`) resolve
each entry in `actorNames` by the **internal object name** via
`FindActorByName`, then create the binding under the resolved actor's **display
label** (`AddPossessable(Found->GetActorLabel(), ...)`). But
`sequencer.get_bindings` reports the binding's `name` field as that stored
**label**, and `sequencer.remove_actors` matches `actorNames` **only** against
that stored label. So the exact name string that `add_actors` accepted
(`BP_Gears_146`) is rejected by `remove_actors` with
`"Actor not found in sequence bindings"` and `bindingsProcessed: 0` — the caller
must first read `get_bindings`, discover the labels, and re-issue `remove_actors`
with the labels (`BP_Gears`) to trim the cast.

Both methods' param help is **identical** ("`actorNames` — Array of actor name
strings"), so nothing signals that `add_actors` speaks the internal name while
`remove_actors`/`get_bindings` speak the label. The add -> remove round-trip
with the same argument list silently no-ops (the RPC returns a *success* envelope
with `bindingsProcessed: 0`; the per-item `success:false` + error are the only
signal), and the "Actor not found in sequence bindings" text reads as though the
actor is missing rather than "you passed the internal name; this verb matches the
label."

## What it should do

Mirror the fix already adopted for `editor.focus_actor` (see
`E-focus-actor-rejects-internal-name-label-only`, which routed `actorName`
through a shared resolver that accepts label OR internal name OR object path):
have `sequencer.remove_actors` (and, for symmetry, the binding-name matching in
any sibling) resolve each `actorNames` entry through the same
name-or-label resolution `add_actors` already uses, so the identifier that binds
an actor also unbinds it. At minimum, make the mismatch discoverable — document
on the overlay that `add_actors` takes the internal object name while
`get_bindings`/`remove_actors` are keyed by the display label, and rewrite the
`"Actor not found in sequence bindings"` error to name which identifier kind was
expected (e.g. "no binding with label '<X>' — remove_actors matches the display
label reported by get_bindings, not the internal actor name add_actors accepts").

## Affected methods (shared root cause — the label-keyed binding namespace)

- `sequencer.add_actors` — input resolved by internal name (`FindActorByName`), binding stored under `GetActorLabel()`
- `sequencer.add_actor` — same single-actor code path
- `sequencer.remove_actors` — matches `actorNames` against the stored label only (where the symptom surfaces)
- `sequencer.get_bindings` — reports the binding `name` field as the stored label only
- `sequencer.add_camera` — likewise binds under `GetActorLabel()`, so the binding namespace is uniformly label-keyed

## Verbatim repro (replay-confirmed live at HEAD)

1. `sequencer.create {name: SEQ_ReplayNameLabel, path: /Game/Cinematics}` -> ok
2. `actor.list` confirms the level actor: `label: "BP_Gears"`, `name: "BP_Gears_146"` (and `label: "UELogo"`, `name: "StaticMeshActor_1"`)
3. `sequencer.add_actors {path: /Game/Cinematics/SEQ_ReplayNameLabel, actorNames: ["BP_Gears_146", "StaticMeshActor_1"]}` (the internal NAMES)
   -> `{"results":[{"name":"BP_Gears_146","success":true,"bindingGuid":"331D5AB0..."},{"name":"StaticMeshActor_1","success":true,"bindingGuid":"ECD428D6..."}]}`
4. `sequencer.get_bindings {path: ...}`
   -> `{"bindings":[{"id":"331D5AB0...","name":"BP_Gears"},{"id":"ECD428D6...","name":"UELogo"}]}`  (names come back as the LABELS)
5. `sequencer.remove_actors {path: ..., actorNames: ["BP_Gears_146", "StaticMeshActor_1"]}`  (the SAME strings that bound them)
   -> `{"removedActors":[{"name":"BP_Gears_146","success":false,"error":"Actor not found in sequence bindings"},{"name":"StaticMeshActor_1","success":false,"error":"Actor not found in sequence bindings"}],"bindingsProcessed":0}`
6. `sequencer.remove_actors {path: ..., actorNames: ["BP_Gears", "UELogo"]}`  (the LABELS from get_bindings)
   -> `{"removedActors":[{"name":"BP_Gears","success":true,"status":"Actor removed"},{"name":"UELogo","success":true,"status":"Actor removed"}],"bindingsProcessed":2}`

The `add_actors` call in step 3 and the failing `remove_actors` in step 5 pass
byte-identical `actorNames`; only the identity domain differs.

## Guilty source (verbatim)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp`

`sequencer.add_actors` resolves by internal name, stores by label:
```cpp
:879   AActor* Found = Ctx.GetSubsystem()->FindActorByName(Name);
...
:893   FGuid BindingGuid = MovieScene->AddPossessable(
:894       Found->GetActorLabel(), Found->GetClass());
```

`sequencer.remove_actors` matches only the stored binding name (= the label):
```cpp
:1074  FString BindingName;
:1075  if (FMovieScenePossessable* Possessable =
:1076          MovieScene->FindPossessable(Binding.GetObjectGuid()))
:1077      BindingName = Possessable->GetName();
...
:1082  if (BindingName.Equals(Name, ESearchCase::IgnoreCase))
...
:1099  Item->SetStringField(TEXT("error"), TEXT("Actor not found in sequence bindings"));
```

`sequencer.get_bindings` publishes that same label under `name` (`:1166-1173`,
`BindingName = Possessable->GetName();` -> `Bobj->SetStringField("name", BindingName)`),
so the whole read/remove side of the family is label-keyed while the add side is
name-keyed.

## Distinction from sibling name/label tickets (not dupes — different method + root cause)

- `E-focus-actor-rejects-internal-name-label-only` (IN-REVIEW) — `editor.focus_actor` is label-only; opposite direction and a different namespace/handler (`ViewportHandler.cpp`).
- `E-networking-actorname-internal-name-only` (IN-REVIEW) — networking verbs are internal-name-only; different namespace/handler.
- `E-inspect-find-by-tag-internal-name-not-label`, `E-actor-name-resolution-label-collision` — actor.* input-side label/name discoverability; not the sequencer binding namespace.
- `E-sequencer-add-camera-no-actor-path`, `F-sequencer-binding-convert`, `F-sequencer-evaluate-readback` — other sequencer gaps; none touch the add-name-vs-remove-label split.

severity rationale: impact=blocker-with-workaround (add -> remove round-trip with the identical argument silently no-ops at bindingsProcessed:0; recovery needs an extra get_bindings read to learn the labels) x reach=common (sequencer cast setup/trim is a normal cinematics workflow) -> Medium

## History
- `#2-go-fix-remove-actors-name-label` `IN-REVIEW` developer — GO (shipped, compiled clean, red test green). Fixed `sequencer.remove_actors` to resolve each `actorNames` entry through the SAME name-or-label-or-path resolver `add_actors` uses (`Ctx.GetSubsystem()->FindActorByName`) and also match on the resolved actor's display label — the identity bindings are stored under (`AddPossessable(GetActorLabel())`, `SequenceHandler.cpp:936-937`). The match is additive: the direct arg-vs-binding-name compare is kept as a fallback so spawnable bindings and orphaned/renamed-actor bindings still remove by their stored name; the per-item "not found" error was rewritten to name both identifier kinds and point at `get_bindings`. File changed: `Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp` (remove_actors match loop, ~:1117-1174). Regression test (adopted red test, in tree): `PinWright.Sequencer.RemoveActors.RoundTripsInternalNameAddActorsAccepts` in `Source/PinWright/Private/Tests/Sequencer/TestSequencerRemoveActorsNameLabel.cpp` — reporter observed it FAIL pre-fix; post-fix it now runs `Result={Success}` (red->green). `get_bindings`/`add_camera` left unchanged by design (they store/report the label; no broken round-trip). Severity Medium / category ergonomic unchanged. Lease retained for the commit phase.
- `#3-test-phase-fix` `IN-REVIEW` developer — Adversarial-review fixes carried to verified-green (compile clean; full suite 3655/3655, 0 fail, 0 crash). Final shipped mechanism in `sequencer.remove_actors` (`SequenceHandler.cpp` ~:1117-1205) is now a TWO-PASS binding match that supersedes #2's single additive OR-loop: Pass 1 removes the binding whose stored name equals the caller's verbatim `actorNames` string (exact self-match wins); Pass 2 falls back to the resolved actor's display label (via the same `FindActorByName` resolver `add_actors` uses) only when Pass 1 found nothing — so an internal-name/label collision across two bindings deletes the caller-named binding deterministically, not whichever iterated first. Binding-name extraction hoisted into a `GetBindingName` lambda; exactly one binding removed via a collected `TargetGuid`; the per-item not-found error rewritten to name both identifier kinds and point at `get_bindings`. Regression test unchanged and green: `PinWright.Sequencer.RemoveActors.RoundTripsInternalNameAddActorsAccepts` (asserted, Result={Success}). Reviewer concern on the fuzzy-`Contains` fallback in this destructive verb was reviewed and left as-is: `McpActorUtils::FindActorByName` (`ActorUtils.cpp:80-101`) is ambiguity-safe (returns null on >1 fuzzy label match) and is the identical resolver `add_actors` uses, so the add/remove round-trip stays symmetric per this ticket's mandate. No board status change (stays IN-REVIEW).
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live at HEAD (SEED-mode `sequencer.remove_actors` cinematic blockout task). `add_actors(["BP_Gears_146","StaticMeshActor_1"])` succeeded resolving the internal names, `get_bindings` returned those bindings under the LABELS `BP_Gears`/`UELogo`, `remove_actors` with the same name strings returned per-item `success:false "Actor not found in sequence bindings"` + `bindingsProcessed:0`, and only `remove_actors(["BP_Gears","UELogo"])` (the labels) removed them. Root cause: `add_actors`/`add_actor` resolve by `FindActorByName` but `AddPossessable(GetActorLabel(),...)` (`SequenceHandler.cpp:879,893-894`), while `remove_actors` matches `Possessable->GetName()` = the stored label (`:1074-1082`) and `get_bindings` reports the same label (`:1166-1173`) — the add side is name-keyed, the read/remove side label-keyed, docs identical for both. Dedup: ripgrep OPEN+closed (qmd unavailable) — no existing ticket covers the sequencer binding name/label split; distinct from the focus_actor/networking/find_by_tag name-label tickets (different namespaces + resolution rules) and from the other sequencer tickets (add_camera path, binding convert, evaluate readback). New symptom-family within sequencer: the label-keyed binding namespace vs the name-keyed add verbs.
