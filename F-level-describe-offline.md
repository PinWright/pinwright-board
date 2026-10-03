---
id: F-level-describe-offline
title: "No read-only verb exposes a level's actor labels or transforms without loading the world — under a held world lock, 24 actors had to be recovered by hand-parsing length-prefixed FStrings out of the .umap bytes"
status: DONE
severity: Medium
category: feature
tags: [level, actor, read-only, offline, umap, world-lock, asset-dump, level-describe, package-parse, weapons]
encounters: 1
costly: 1
lastSeen: 2026-09-03T00:00:00Z
---

# "What actors are in this map, and where?" has no answer that does not take the world

Every published route to a level's contents goes through the active editor world. When another
stream holds the world lock — the normal condition when several agents share one editor — the
question is unanswerable through the plugin, for a map that is sitting on disk in a readable format.

## The three routes, and why each is closed

**1. `asset.dump` on a `.umap` is a world-loading background job.** It does produce the answer
(`actors/manifest.json` plus per-actor JSON in the `pinwright.actor-describe.v1` shape), but it gets
there by loading the world — so it is exactly the thing an asset-only agent under a held lock must
not run, and it is asynchronous besides.

**2. `actor.*` needs the level to be the active world.** `actor.list`, `actor.describe`,
`actor.get_transform` all read the active editor world. There is no `levelPath` that redirects them
at an unloaded map.

**3. `system.inspect.*` needs the same.** Same precondition, same wall.

`level.get_info` / `get_actors` / `get_bounds` do take a `levelPath`, and they are the closest
existing shape — but they resolve it only against levels **already loaded into the active world**,
which is the subject of `E-level-getters-require-loaded-not-on-disk`.

## What that cost — measured

Recovering **24 actor labels and world locations** from
`X:\src\unreal\EAContentExamples58\Content\FPS\Test\T_Weapons.umap` required reading the package
file directly and **hand-parsing length-prefixed FStrings and transform triples out of the raw
bytes** — reconstructing, by hand and without a schema, data the plugin already knows how to emit in
a documented shape.

That is not a workaround in the rubric's sense (a documented alternative, or many extra calls). It
is a bespoke binary parse of an engine format, with no validation, that happens to have worked once.
Anything it got subtly wrong would have been invisible.

## What is asked for

A **`level.describe_offline`** (name illustrative) that reads a `.umap` from disk and returns its
actors without loading it or touching the active world:

- **Minimum:** per actor `{label, name, class, location, rotation, scale}` — the fields that make
  "did the layout land, and where is everything" answerable. This is what the hand-parse was after.
- **Useful next:** `folder`, `tags`, `guid`, and the actor count as a summary line; plus the
  external-actor references that One File Per Actor maps split out, listed rather than resolved
  (the `actors/manifest.json` shape already distinguishes embedded from external).
- **Contract that makes it worth having:** *never* loads the world, *never* changes the active level,
  and says so in its own documentation — so an asset-only agent can call it under a held lock as a
  matter of policy, not of hope. That guarantee is most of the value; a verb that "usually" avoids
  loading is not usable here.
- **Shape reuse:** emit `pinwright.actor-describe.v1`, the schema `asset.dump`'s `actors/*.json`
  already uses, so offline and loaded reads are diffable against each other without translation.

## Why this is not the same ask that was already declined

`E-level-getters-require-loaded-not-on-disk` (IN-REVIEW) explicitly ruled the offline path out of
scope — verbatim: *"auto-loading / peeking an unloaded on-disk map for a read getter (load-then-restore
or a registry peek …) is intentionally out of scope: it drags active-world mutation/restore risk into
a read-only path for a marginal payoff"*. **That reasoning is correct and this ticket does not
contest it.** It is an argument against making `level.get_info` load a map behind the caller's back —
against a *hidden* load inside an existing getter. It is not an argument against a separate,
explicitly-named verb that parses the package file and never loads anything: a disk parse cannot
mutate the world it never opens, so the risk that justified the exclusion does not exist on this
path. The payoff is also no longer marginal, since the alternative measured here was a hand-written
binary parse.

## Second-order value

An offline reader lets **asset-only agents prepare world work safely** — enumerate what a map
contains, plan edits, and validate a plan's assumptions — all before anyone takes the world lock.
Today that preparation either waits for the lock or does not happen, which is what pushes agents into
exactly the improvisation this ticket documents.

## Severity

**Medium** — the rubric's hard-blocker band, adjusted for reach. Through the published surface the
task is impossible while the lock is held: there is no documented workaround and no many-extra-calls
route, only an off-surface binary parse. Not High because it is not an every-session path — it bites
specifically when the world lock is contended, which is common in multi-agent sessions and absent in
single-agent ones.

## Related

- `E-level-getters-require-loaded-not-on-disk` (IN-REVIEW, Low) — the nearest existing surface, and
  the ticket that declared the offline peek out of scope for those three getters. Its fix
  (`LEVEL_NOT_LOADED` instead of a misleading `LEVEL_NOT_FOUND`) makes the wall *honest*; this ticket
  asks for a door. They are complements, not alternatives, and the new verb is the natural thing for
  that improved error message to point at.
- `B-level-load-dirty-world-memory-leak-fatal`, `B-level-load-dirty-world-fatal` — the cost of the
  load this verb avoids, and part of why "just load it" is not a neutral suggestion.
- `E-actor-list-no-limit-spills`, `E-actor-describe-no-header-only-read` (IN-REVIEW) — the loaded-world
  actor reads, and their response-size ergonomics. Whatever projection they settle on should apply
  here too; a 24-actor map is small, a real level is not.
- `B-dump-folder-includes-level-subobjects`, `B-asset-dump-folder-no-completion-signal` — the
  `asset.dump` route's own problems, which is the other reason it is a poor substitute.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 2. No read-only verb exposes a level's actor labels or transforms without loading the world: `asset.dump` on a `.umap` is a world-loading background job (it does emit `actors/manifest.json` plus per-actor `pinwright.actor-describe.v1` files, but only by loading the world), and `actor.*` / `system.inspect.*` all require the level to be the active world; `level.get_info` / `get_actors` / `get_bounds` take a `levelPath` but resolve it only against levels already loaded, per `E-level-getters-require-loaded-not-on-disk`. With the world lock held by another stream, recovering 24 actor labels and world locations from `X:\src\unreal\EAContentExamples58\Content\FPS\Test\T_Weapons.umap` required reading the package file and hand-parsing length-prefixed FStrings and transform triples out of the raw bytes — a bespoke, unvalidated binary parse of an engine format, reconstructing data the plugin already emits in a documented shape. Ask: a `level.describe_offline`-style verb that reads a `.umap` from disk without loading it — minimum `{label, name, class, location, rotation, scale}` per actor, then `folder`/`tags`/`guid` and external-actor references listed rather than resolved, emitting the existing `pinwright.actor-describe.v1` schema so offline and loaded reads are diffable, and carrying a documented guarantee that it never loads the world or changes the active level (that guarantee is most of the value — a verb that only usually avoids loading is unusable under a lock). Explicitly not the ask `E-level-getters-require-loaded-not-on-disk` declined: that ticket ruled out a *hidden* load-then-restore or registry peek inside an existing getter because it "drags active-world mutation/restore risk into a read-only path" — correct, and untouched here, since a disk parse cannot mutate a world it never opens; the payoff is also no longer marginal now that the measured alternative is a hand-written binary parse. Second-order value: asset-only agents could enumerate and validate a map before anyone takes the world lock, which is the preparation that today either waits or turns into the improvisation above. Severity Medium on the hard-blocker band (no documented workaround, no many-calls route, only an off-surface parse), not High because the block is specific to a contended world lock rather than every session.
- `#2-offline-umap-reader` `IN-REVIEW` developer — Added `level.describe_offline {levelPath}` (new file `Source/PinWright/Private/Handlers/Level/LevelDescribeOfflineHandler.cpp`). It opens the `.umap` with a plain file reader, reads `FPackageFileSummary` + name/import/export tables, finds the `World`/`PersistentLevel` export, parses the level's `Actors` list (self-checked: every entry must be an export of that level, else `PARSE_FAILED`), then walks each actor's tagged-property stream (both the 5.4+ complete-type-name tag format and the older one) for `ActorLabel`/`FolderPath`/`Tags`/`ActorGuid`/`RootComponent`, and composes the world transform through the root component's `AttachParent` chain inside the package (engine `CalcNewComponentToWorld_GeneralCase` rules incl. absolute flags). Unsaved transform fields come from the native class default's component template (FindObject only — exact); for Blueprint classes they fall back to the component class default and the actor reports `transformExact:false` + `transformCaveats` (also for socket attachments and out-of-package parents). Emits `pinwright.actor-describe.v1` per actor (`storage:"embedded"`, no properties/components) plus `levelPath`, `packageFile`, `worldPath`, `usesExternalActors`, `embeddedActorCount`, `externalActorReferenceCount`, `externalActors[]` (external-actor packages listed from the asset registry, not resolved). Structural no-load guarantee: no LoadObject/LoadPackage/CreatePackage in the verb. Errors: `ASSET_NOT_FOUND`, `NOT_A_MAP`, `PARSE_FAILED` (cooked/unversioned/text/malformed — never a partial list). Docs: `docs/wiki-src/level.md` (`### level.describe_offline` + Cross-cluster pointer), `CHANGELOG.md`. Tests (`Source/PinWright/Private/Tests/World/TestLevelDescribeOffline.cpp`, saved scratch map with a labelled/foldered/tagged/scaled parent and an attached child): `PinWright.level.describe_offline.MatchesLoadedActorDescribe` (offline fields equal `ActorDescribeBuilder::BuildActorJson` of the live actors, child world transform composed), `PinWright.level.describe_offline.NeverLoadsTheMapOrTouchesTheEditorWorld` (byte copy of the .umap under a never-loaded name: `FindPackage` null before and after, editor world unchanged, labels still read), `PinWright.level.describe_offline.RefusesMissingAndNonMapPackages`. Not done: the `LEVEL_NOT_LOADED` message on `level.get_*` still points at `asset.dump`, not at this verb (left untouched to keep this change in new files); no `limit`/`fields` projection yet.
- `#3-malformed-input-hardening` `IN-REVIEW` developer — Review round: a malformed `.umap` could kill the editor. Now the file reader is opened `FILEREAD_Silent` (an EOF read sets the error flag instead of logging an Error), the summary's name/import/export offsets are range-checked against the file size before any `Seek` (the file reader `checkf`s on an out-of-range seek), the name-table read is checked before the import seek, `ReadTags` rejects `SerialSize < 0`, the World-outer lookup uses `IsValidIndex`, type-name `InnerCount > 64` is rejected, and `ActorLabel` goes through a length check against the tag size before `FString` allocates. Added `source {fileMd5, unsavedChanges}` (shared `AssetDumpBuilder::BuildSourceStampJson`) with a warning when the map is loaded dirty, and a warning when the level uses actor-folder objects (folder reads `None`). Docs: `level.md` corrected (a text-format map gives `ASSET_NOT_FOUND`), README verb-table row. New test `PinWright.level.describe_offline.MalformedFilesAreRefusedNotCrashed` (garbage file, header cut inside the name table, export table past EOF, export table at EOF → each `PARSE_FAILED`); `MatchesLoadedActorDescribe` now also asserts the clean/dirty provenance. Not done: seeking via `ScriptSerializationStartOffset` (would need a 5.3 guard; the parse already starts at `SerialOffset`, which is equivalent for these classes).
- `#4-review-fixes` `IN-REVIEW` developer — Re-review round: three crafted-file crash paths closed, all now `PARSE_FAILED`. (1) The export data-range guard compared `SerialOffset + SerialSize` after adding them, so an offset near INT64_MAX wrapped negative and reached the file reader's past-EOF `checkf` in `Seek`; it now checks `SerialOffset <= FileSize` and `SerialSize <= FileSize - SerialOffset` without the sum. (2) Name-table entries are read by `ReadNameEntry`: the length is pre-checked (non-zero, within `NAME_SIZE`) before `FNameEntrySerialized`, the read error is checked before `FName(Entry)` is built, and the entry must end in its NUL, so `FName` never `Strlen`s an unfilled or unterminated buffer into its max-length `checkf` (the engine's "String is too long" Error log is also never reached). (3) An FName reference with a negative number is refused in `FNameTableArchive` (`_-N` never re-parses, so a path built from it could pass `NAME_SIZE` inside `FindObject`). Also: an actor-list read that runs past EOF is refused instead of returning zero actors; `RootComponent` / `AttachParent` references that are mis-sized or outside the import/export tables are refused instead of reported as an exact identity transform or "outside this package"; a Blueprint-class actor with no saved `Tags` reports `tagsExact:false` (new per-actor bool; a native class with no saved `Tags` now reads its class default tags exactly); the `Tags` loop bound counts the 4-byte count prefix. Tests: `PinWright.level.describe_offline.MalformedFilesAreRefusedNotCrashed` gains `_NameLengthCorrupt`, `_NameUnterminated`, `_ExportRangeOverflows`, `_NegativeNameNumber` (table offsets intact, so each reaches the code it targets) and an imports-before-exports precondition; `MatchesLoadedActorDescribe` asserts `tagsExact:true` for the native fixture actors. Files: `LevelDescribeOfflineHandler.cpp`, `TestLevelDescribeOffline.cpp`, `docs/wiki-src/level.md` (`tagsExact`, a Tags bullet), `CHANGELOG.md` (entry extended). Not covered by a test: the actor-list EOF and the out-of-range `RootComponent` / `AttachParent` refusals (crafting them needs per-property byte offsets in export data); the Blueprint `tagsExact:false` path (needs a Blueprint actor fixture). fastcheck OK on both files; not built or run.
- `#5-review-fixes` `IN-REVIEW` developer — Re-review #2: a `Tags` value whose count is negative or does not equal 4 + 8*count bytes, an `ActorGuid` not 16 bytes and a `FolderPath` not 8 bytes are now `PARSE_FAILED` instead of a partial or zero reading; `ReadTriple` refuses NaN/Inf (the JSON builder would have written 0). Class defaults are now found with `FindObjectFast<UPackage>` + `FindObjectFast<UClass>` on the class import's FNames (`FindLoadedClass`), so no path string is parsed (removes `StaticFindObject`'s T3D-import load branch and the long-path surface). Docs / CHANGELOG / registration summary: `tagsExact:false` also covers a class not loaded in this editor; the hardening claim is scoped to the tables and property data. Test safety: `MalformedFilesAreRefusedNotCrashed` now writes its crafted `.umap` files under `/Temp/PinWright/TestTemp/...` (Saved/), not `/Game/PinWrightTests`: the asset registry watches and searches only non-read-only roots and `ScanPathsSynchronous` refuses `/Temp`, so its `FPackageReader` (same unterminated-name exposure) can never read them mid-run or at the next start; the dir is removed on scope exit. Not built or run; fastcheck OK.
- `#6-verified-linux` `DONE` tester — Fix commit b5b0bd56. Passed non-skipped in run3/full: `PinWright.level.describe_offline.MatchesLoadedActorDescribe`, `.NeverLoadsTheMapOrTouchesTheEditorWorld`, `.RefusesMissingAndNonMapPackages` and `.MalformedFilesAreRefusedNotCrashed`. Minimum ask: label, name, class, location, rotation and scale equal `ActorDescribeBuilder` output for live actors, including an attached child's composed world transform (`MatchesLoadedActorDescribe`). Useful-next ask: folder, tags, guid and counts (`embeddedActorCount`, `usesExternalActors`, `externalActorReferenceCount`) are asserted in the same tests. Contract: `FindPackage` is null before and after on a never-loaded copy, the editor world is unchanged, and the guarantee is documented in `docs/wiki-src/level.md` ("never loads the map ... never changes the", source read). Shape reuse: emits `pinwright.actor-describe.v1`. Hardening: crafted files give PARSE_FAILED, not a crash. Coverage limits: the OFPA `externalActors[]` listing is tested only in the zero case (no OFPA fixture). The Blueprint `tagsExact:false` path, the actor-list EOF refusal and the out-of-range RootComponent/AttachParent refusals are untested. There is no `limit`/`fields` projection. None of these is a minimum or contract item.
