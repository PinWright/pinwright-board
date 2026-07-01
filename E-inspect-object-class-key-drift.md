---
id: E-inspect-object-class-key-drift
title: "system.inspect.inspect_object emits the class under `className`, but its sibling readers (list_objects / find_by_class) and its own nested components[] use `class` — a naive reader who learned `class` misses it"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [inspection, response-key, naming-drift, class, className, docs]
---

# `system.inspect.inspect_object` top-level class key is `className`; its siblings (and its own components) use `class`

Within the **same `system.inspect` namespace and the same source file**
(`EnvironmentHandler.cpp`), the conceptual "what class is this?" output field is
spelled two different ways across read-only methods a caller naturally chains in
one audit:

| Method | Output key for the class | Source |
|--------|--------------------------|--------|
| `system.inspect.list_objects` | **`class`** (per-row) | `EnvironmentHandler.cpp:1454` |
| `system.inspect.find_by_class` | **`class`** (per-row) | `EnvironmentHandler.cpp:1510` |
| `system.inspect.find_by_tag` | **`class`** (per-row) | `EnvironmentHandler.cpp:1556` |
| `system.inspect.inspect_object` (**top-level result**) | **`className`** | `EnvironmentHandler.cpp:1718` |
| `system.inspect.inspect_object` (**nested `components[]`**) | **`class`** | `EnvironmentHandler.cpp:1770` |

(`inspect_object`'s non-actor `else` branch at `:1794`-`:1804` emits no class key
at all — only `isActor:false` plus an optional `transform`; the single top-level
`className` at `:1718` is shared by both the actor and non-actor paths. The
separate `className` emission at `:1597` belongs to a *different* method,
`system.inspect.inspect_class`, registered at `:1572` — not a branch of
`inspect_object`.)

So `inspect_object` is **internally inconsistent** (its top-level object says
`className` while the `components[]` it returns say `class`) *and* it disagrees
with its sibling enumerators (`list_objects`, `find_by_class`, `find_by_tag` all
say `class`). A caller who runs the
natural audit chain — `list_objects` / `find_by_class` (learns the key is
`class`), then `inspect_object` on one of the returned actors — will look for
`class` on the inspect result and find only `className`, or write a parser keyed
on one spelling and silently get `null`/`undefined` on the other.

This is **output-key naming drift**, a distinct axis from the input-param drift
the board already aggregates:
- **Distinct from `E-asset-path-vs-assetpath-list-drift`** (IN-REVIEW) — that is
  the *input param* `path` vs `assetPath` axis. This is the *response key*.
- **Distinct from `E-class-name-format-inconsistency`** (DONE) — that unified the
  accepted *input* class-name string *formats* (short vs `/Script/` path) across
  tools; it never touched which key the *output* carries the class under.
- **Not the same as `B-inspect-object-omits-component-properties`** (DONE) — that
  added the missing `properties` field to `inspect_object`; it left the
  `className`-vs-`class` key spelling untouched (the verified output in that
  ticket's `#5` history still shows the top-level `className`).

## What it should do

Make `inspect_object` carry the class under the same canonical key as its
siblings and its own `components[]`:
- Emit the leaf class under **`class`** at the top level of `inspect_object`
  (matching `list_objects` / `find_by_class` / `find_by_tag` and its own nested
  `components[]`), while keeping `className` as a back-compat alias carrying the
  identical value so parsers keyed on either spelling keep working. `classPath`
  stays as-is for the full `/Script/...` path.
- **Docs (`docs/wiki-src/system.inspect.md` overlay, `### system.inspect.inspect_object`
  section):** state explicitly that the top-level class is carried under `class`
  (with `className` as the legacy alias and `classPath` as the full path), so a
  reader isn't misled by the prior generic "class" wording into thinking the
  top-level key was only `className`.

**Fix:** In `EnvironmentHandler.cpp` `system.inspect.inspect_object`, emit the
leaf class name under both `class` (canonical, shared with the components[] and
sibling enumerators) and `className` (back-compat alias) at the top level; the
non-breaking additive alias was chosen over renaming because no test asserts the
`className` key for `inspect_object` and it lets parsers keyed on either spelling
work. Updated the overlay `### system.inspect.inspect_object` section to document
the `class`/`className`/`classPath` triplet.

This was pure guessability / parser-mismatch overhead; the task that surfaced it
completed clean (no error, no retry) because the agent eyeballed the spilled
response and noticed the key by hand. The cost was the silent-miss risk for any
caller that keyed a parser on the wrong spelling.

## Evidence

From the struggle audit of a read-only scene-audit task (focus
`system.inspect.get_scene_stats`, namespace `system.inspect`, outcome **clean** —
all 9 calls `ok`/non-error). The story walked the textbook inspection chain on
the Content Examples `ExampleProjectWelcome`/`PersistentLevel` level (227 actors):
`get_scene_stats` → `list_objects` (full enum, spilled to file) →
`find_by_class {className:"StaticMeshActor"}` → `inspect_object` on
`StaticMeshActor_1` (spilled to file) → `get_scene_stats` (confirm). The agent
read both spilled responses read-only and observed the discrepancy directly.

Friction note, verbatim: *"none — wiki pages on disk gave every param up front;
two responses spilled to HttpResponses files (expected for a 227-actor level /
100-property actor) and I parsed them read-only; one minor note: inspect_object
returns the class under key `className` not `class`, which a naive reader could
miss."* So the self-reported outcome was clean, but the agent flagged the exact
output-key drift this ticket tracks — `inspect_object` → `className`, while the
`list_objects`/`find_by_class` it had just used (and the wiki prose) say `class`.

(The two file-spills in the same task are owned elsewhere and not re-filed here:
`list_objects` spilling is `E-inspect-list-objects-no-limit-spills` (IN-REVIEW);
the `inspect_object` 100-property payload is the intended output of
`B-inspect-object-omits-component-properties` (DONE). Only the `class`/`className`
key drift is unowned.)

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a read-only scene-audit task (focus `system.inspect.get_scene_stats`, namespace `system.inspect`, outcome **clean** — all 9 calls `ok`/non-error). The agent ran the natural inspection chain (`list_objects` → `find_by_class StaticMeshActor` → `inspect_object` on `StaticMeshActor_1`) on the Content Examples `ExampleProjectWelcome` 227-actor level and flagged that `inspect_object` returns the class under key `className` not `class`, "which a naive reader could miss." Confirmed in source (`EnvironmentHandler.cpp`, same file/namespace): `list_objects` (:1454), `find_by_class` (:1510, :1556), and `inspect_object`'s own nested `components[]` (:1770) all emit `class`; only `inspect_object`'s top-level result (:1718) and its non-actor branch (:1597) emit `className`. So the method is internally inconsistent (top-level `className` vs nested `components[].class`) and disagrees with its two sibling enumerators. The wiki overlay (`docs/wiki-src/system.inspect.md`) documents `list_objects` rows as `{name, path, class}` (lines 7, 32, 36) but describes `inspect_object` with a generic "class" (lines 16-22), so the prose teaches the `class` spelling the top-level key does not use. Output-key drift, a distinct axis from the input-param drift family (`E-asset-path-vs-assetpath-list-drift` is `path`/`assetPath` input; `E-class-name-format-inconsistency` DONE unified input class-name *formats*; `B-inspect-object-omits-component-properties` DONE added `properties` but left the key spelling). Friction note (verbatim): *"one minor note: inspect_object returns the class under key `className` not `class`, which a naive reader could miss."* Recovered without error (the agent read the spilled response by hand); the cost is the silent-miss risk for a parser keyed on the wrong spelling. Proposed: emit `class` at the top level of `inspect_object` (or keep `className` + a `class` alias for a transition release), unify with the nested `components[]`, and until then spell out the `className` top-level key in the `### system.inspect.inspect_object` overlay section. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX found no ticket owning the `class`/`className` *output-key* drift in `system.inspect` (the two file-spills in this task are owned by `E-inspect-list-objects-no-limit-spills` / `B-inspect-object-omits-component-properties` and are not re-filed).
- `#2-retriage` `OPEN` triage — Low→Medium: silent class/className output-key drift across chained inspect verbs can null a parser keyed on the wrong spelling, on the every-session inspect path.
- `#3-fix` `IN-REVIEW` developer — Unified the class output-key on the additive, non-breaking path recommended by all three lenses: `system.inspect.inspect_object` now emits the class short-name at the top level under **both** `class` (matching its siblings `list_objects`/`find_by_class`/`find_by_tag` and its own nested `components[].class`) **and** `className` (kept for back-compat), both sourced from one `ClassShortName` local. A parser keyed on `class` (learned from the audit chain it disagreed with) now reads it here; existing `className` parsers are untouched. The set is before the actor/non-actor split (`EnvironmentHandler.cpp:1718`), so both target kinds carry the alias. Source: `Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp` (top-level emission). Docs: `Docs/wiki-src/system.inspect.md` `### system.inspect.inspect_object` section now states the class short-name is emitted under both `class` and `className` (path under `classPath`), removing the generic-"class"/`className` mismatch the ticket flagged. Regression test: `Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp` `PinWright.system.inspect.inspect_object.EmitsClassKey` — spawns an actor + adds a USceneComponent, inspects both (exercising the actor and non-actor paths) and asserts the top-level `class` is present, equals the class short-name, and agrees with `className`; fires if the `class` alias is reverted. Notes on the two cited line-label slips (verified against current source, thesis intact, no body rewrite needed): `:1556` is `find_by_tag` (also emits `class`, point stands), and inspect_object has no separate non-actor `className` line — `:1718` sets the top-level `className` unconditionally for both actor and non-actor targets (so the alias covers both).
- `#4-reword-and-fix` `IN-REVIEW` developer — (Independent host; integrated alongside `#3-fix` — same additive fix, no behavioral divergence.) Reworded then fixed. **Reword:** corrected two source mis-citations the original carried — `:1597` is `system.inspect.inspect_class` (registered `:1572`), NOT `inspect_object`'s non-actor branch (that `else` branch at `:1794`-`:1804` emits no class key at all; `inspect_object`'s only class key is the shared top-level `className` at `:1718`), and the `:1556` row is `find_by_tag` (registered `:1525`), not `find_by_class` (whose per-row `class` is `:1510` only). Split the table to attribute `:1556` to `find_by_tag` and added the non-actor-branch / inspect_class clarifier; severity kept Medium (Low naming impact + Reach +1 for the every-session inspect path, per the README rubric). **Fix (code):** `EnvironmentHandler.cpp` `system.inspect.inspect_object` now emits the leaf class under both `class` (canonical, matching its own nested `components[].class` at `:1770` and the `list_objects`/`find_by_class`/`find_by_tag` rows) and `className` (back-compat alias, identical value) at the top level. Additive/non-breaking — chosen over a rename because the only `className` response assertion in the suite is on `environment.spawn`, not `inspect_object`. **Docs:** updated the `### system.inspect.inspect_object` overlay section in `docs/wiki-src/system.inspect.md` to document the `class`/`className`/`classPath` triplet. **Test:** added `PinWright.system.inspect.inspect_object.EmitsClassKey` in `Tests/World/TestEnvironmentHandlers.cpp` — spawns an actor, invokes the real handler via `InvokeHandlerWithCapture`, and asserts the top-level `class` key is present and equals the actor's leaf class name, and that the `className` alias carries the same value (fails if `class` is dropped or diverges). Files: `Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp`, `Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp`, `docs/wiki-src/system.inspect.md`.
