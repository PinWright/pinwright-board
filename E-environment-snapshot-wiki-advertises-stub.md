---
id: E-environment-snapshot-wiki-advertises-stub
title: "environment namespace prelude + the export_snapshot/import_snapshot method pages advertise the snapshot round-trip with a required path param and no caveat that both verbs are NOT_IMPLEMENTED stubs"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, environment, snapshot, export-snapshot, import-snapshot, wiki, discoverability, stub]
---

# The `environment` wiki advertises the snapshot round-trip with no stub caveat

`environment.build.export_snapshot` and `environment.build.import_snapshot` are
NOT_IMPLEMENTED stubs (the C++ honesty fix is tracked under
`B-export-snapshot-empty-stub`, IN-REVIEW — both now fail loud with a clear
`[NOT_IMPLEMENTED]` + workaround hint instead of the old silent
`success:true`). The wiki, however, still steers a caller straight into them with
no warning:

- **Namespace prelude** (`docs/wiki-src/environment.md:3-5`): the overview opens
  "…terrain, sky, fog, lighting **snapshots**, and time-of-day state" and then
  says *"Use `call("environment.build")` for generated environment assets **or
  snapshots**."* — an unqualified capability claim for snapshots at the level a
  reader scans before drilling into any method.
- **Per-method pages** (auto-generated `environment.build.export_snapshot.md` /
  `environment.build.import_snapshot.md`): still read "Export/Import an
  environment snapshot to/from a JSON file" with a **required `path` param** and
  no hint the handler is a stub. (Noted verbatim as a residual sub-note in
  `B-export-snapshot-empty-stub` history `#3`, `#4`, and `#5`, each time
  deliberately "not filed separately" — this is that deferred docs-overlay
  companion.)

So discovery is *smooth and wrong*: the wiki has the param, the call's args are
right first try, and the caller only learns the verb does nothing at call time.
On a populated scene the import side is worse — there's no on-disk file to even
attempt, because export refused to write one.

## Evidence (this task's friction note, verbatim)

Focus `environment.build.bake_lightmap`; namespace `environment`; outcome
`tool_bug` (judge tracked `B-export-snapshot-empty-stub`). Steps 1-5 of a clean
mid-morning daylight rig (sky sphere, `set_time_of_day {time:10.5}`, directional
sun + sky light, `set_sun_intensity {8}` / `set_skylight_intensity {1.5}`,
`bake_lightmap {quality:"production"}`) all succeeded; step 6 — the documented
snapshot save — dead-ended:

> *"Discovery was smooth (wiki had all params, every call's args were right first
> try); the blocker is a real capability gap — export_snapshot AND import_snapshot
> are both unimplemented stubs that error instead of persisting/restoring, so the
> documented snapshot round-trip cannot complete **despite the method appearing in
> the wiki with a required path param**."*

The PROCESS angle: the wiki advertising the round-trip without a stub caveat is
what made the agent attempt it (and then attempt `import_snapshot` on a file that
was never written) — the 2 wasted `is_error` calls
(`export_snapshot {path:"Saved/EnvSnapshots/morning.json"}`,
`import_snapshot {path:"Saved/EnvSnapshots/morning.json"}`) are discoverability
overhead the docs could pre-empt.

## Why this is a distinct PROCESS angle (not a re-file of the B- ticket)

`B-export-snapshot-empty-stub` is the **capability/honesty** ticket: its action
is a pure C++ change (make both stubs fail loud). Its history explicitly and
repeatedly identifies the residual wiki gap and declines to file it:

- `#3`: *"the auto-generated wiki page `environment.build.export_snapshot.md`
  still reads 'Export an environment snapshot to a JSON file' with a required
  `path` param and no hint the handler is a NOT_IMPLEMENTED stub … Not filed
  separately … a wiki-src overlay note could pre-warn but is optional."*
- `#4`/`#5`: *"The wiki pages still advertise both verbs without a stub warning …
  same residual wiki sub-note … no new ticket."*

This is that companion. The runtime error is honest, but the docs still steer the
caller in. Same shape as the accepted overlay precedents
`E-texture-create-wiki-advertises-stub`,
`E-level-structure-wp-wiki-advertises-dead-end`, and
`E-set-transition-rules-wiki-overstates-rule-authoring`: the wiki overview
over-advertises a capability the runtime does not deliver; the fix is a
`docs/wiki-src/` overlay edit.

Orthogonal to the other environment tickets: `E-set-time-of-day-no-suntime-readback`
(TOD readback property name), `E-environment-build-create-no-name-param` (naming
created actors). Neither touches the snapshot advertisement.

## Fix (wiki overlay — `docs/wiki-src/environment.md`)

1. **Prelude (`:3-5`):** drop the unqualified "or snapshots" claim from the
   `call("environment.build")` line and from the "…lighting snapshots…" list, or
   qualify it inline ("snapshot export/import is not yet implemented").
2. Add a namespace-page-visible `## ` section (below the prelude so it renders on
   the namespace page, not the root index) stating that
   `environment.build.export_snapshot` / `import_snapshot` are **NOT_IMPLEMENTED**
   (export writes nothing; import applies nothing), and pointing at the documented
   workaround the runtime error already gives: capture the look via `actor.list` +
   per-actor `property.get`, and rebuild on restore with the typed spawn verbs
   (`environment.build.create_*`, `environment.spawn_*`) + `property.set`.
3. Cross-reference `B-export-snapshot-empty-stub` for the capability status. (The
   per-method auto pages inherit their summary from the handler registration; if
   the `B-` fix's registration summaries already begin "Returns NOT_IMPLEMENTED…"
   the per-method pages self-document — verify and, if so, scope this to the
   prelude/namespace section only. The wiki edit itself is the downstream process,
   not this ticket.)

## History
- `#1-initial-audit` `OPEN` reporter — Struggle (process) audit of the clean
  mid-morning daylight-rig task (focus `environment.build.bake_lightmap`,
  namespace `environment`, outcome `tool_bug`; judge tracked
  `B-export-snapshot-empty-stub`, whose history `#5` already absorbed this task's
  both-direction repro). Distinct PROCESS angle: the `environment` wiki advertises
  the snapshot round-trip with no stub caveat — the prelude
  (`docs/wiki-src/environment.md:3-5`) lists "lighting snapshots" and says
  `call("environment.build")` is "for generated environment assets or snapshots",
  and the auto method pages `environment.build.export_snapshot.md` /
  `import_snapshot.md` carry a required `path` param with no NOT_IMPLEMENTED hint.
  Friction note (verbatim): *"Discovery was smooth (wiki had all params, every
  call's args were right first try); the blocker is a real capability gap …
  despite the method appearing in the wiki with a required path param."* 2 wasted
  `is_error` calls (`export_snapshot` + `import_snapshot`, both
  `{path:"Saved/EnvSnapshots/morning.json"}`). This is the docs-overlay companion
  that `B-export-snapshot-empty-stub` history `#3`/`#4`/`#5` repeatedly identified
  and declined to file ("Not filed separately … no new ticket"). Dedup (ripgrep
  over OPEN+closed; qmd unavailable): no existing `environment*snapshot*wiki` E-
  ticket; not covered by `B-export-snapshot-empty-stub` (C++ honesty fix),
  `E-set-time-of-day-no-suntime-readback` (TOD readback), or
  `E-environment-build-create-no-name-param` (actor naming). Same accepted shape as
  `E-texture-create-wiki-advertises-stub` /
  `E-level-structure-wp-wiki-advertises-dead-end`. Names the page to edit:
  `docs/wiki-src/environment.md`.
- `#2-additional-golden-morning-preset` `OPEN` reporter — Additional evidence: the wiki-advertises-stub friction reproduced again on an independent task (a "golden morning" environment preset: sky sphere + `set_time_of_day {hour:7.5}` + `set_sun_intensity {6}` + `set_skylight_intensity {1.5}` + `create_fog_volume {x:0,y:0,z:200}` — all of which succeeded — then export → midday-variant tweak → import-to-restore → roundtrip-export). The generated wiki pages still steer the caller straight in: replay-confirmed `environment.build.export_snapshot.md` reads exactly "Export an environment snapshot to a JSON file" and `environment.build.import_snapshot.md` reads "Import an environment snapshot from a JSON file", each with a single required `path` param and **no** NOT_IMPLEMENTED caveat; the namespace page `environment.md` still lists both in the `## Methods` index and the prelude still says `call("environment.build")` is "for generated environment assets or snapshots". The agent's args were right first try (smooth-and-wrong discovery) and it only learned the verbs are stubs at call time, burning 2 `is_error` calls (`export_snapshot` + `import_snapshot`, both `{path:"Saved/EnvSnapshots/golden_morning.json"}`) and dead-ending steps 6/8/9 (export/import/roundtrip). Oracle replay-confirmed both runtime errors verbatim via `mcp__editor-automation__call`: `export_snapshot` → `[NOT_IMPLEMENTED] environment.build.export_snapshot is not implemented: it does not enumerate or serialize the level's environment actors, so the snapshot cannot restore the scene. To persist a look, capture the environment with actor.list + per-actor property.get and rebuild on restore with the typed spawn verbs (environment.build.create_*, environment.spawn_*) + property.set.`, `import_snapshot` → `[NOT_IMPLEMENTED] environment.build.import_snapshot is not implemented: it parses the file but applies nothing to the world (no spawn, no property set), so it cannot restore an environment. Rebuild the look manually with the typed spawn verbs (environment.build.create_*, environment.spawn_*) + property.set.` (the underlying `B-export-snapshot-empty-stub` honest-failure fix is still behaving as designed — clean errors, no file written). MCP stayed fully reachable; this remains the docs-overlay companion, still OPEN/unfixed. No new defect filed.
- `#3-wiki-and-summary-fix` `IN-REVIEW` developer — Fixed the discoverability gap on BOTH surfaces the ticket names. Verification of Fix step #3's self-documentation hypothesis resolved AGAINST it: the per-method auto pages render the registration summary verbatim (`WikiHandler.cpp` `RenderMethodPage` emits `Reg.Summary`), and the two registrations carried the bare "Export/Import an environment snapshot to/from a JSON file" with a required `path` param and no caveat — so the prelude overlay alone would have left the per-method pages bare (matching the accepted `E-texture-create-wiki-advertises-stub` shape, whose `TextureHandler.cpp` create-verb summaries DO begin "Returns NOT_IMPLEMENTED: ..." AND carry a `## Create coverage` overlay). Changes: (1) `Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp` — prefixed both `export_snapshot` (`:124`) and `import_snapshot` (`:161`) registration summaries with "Returns NOT_IMPLEMENTED: ..." (so each per-method page self-documents the stub) and noted the `path` param is unused; the runtime `SendError("NOT_IMPLEMENTED", ...)` bodies were already honest and unchanged. (2) `docs/wiki-src/environment.md` — dropped the unqualified "lighting snapshots" / `call("environment.build")` "or snapshots" claims from the prelude and added a namespace-page-visible `## Snapshot round-trip is not implemented` section that states both verbs are NOT_IMPLEMENTED, warns the required `path` is not a persist promise, and points at the `actor.list` + per-actor `property.get` capture / typed-spawn-verbs + `property.set` rebuild workaround, cross-referencing `B-export-snapshot-empty-stub`. Regression test: `Source/PinWright/Private/Tests/Infra/TestWikiHandler.cpp` `FWikiHandlerEnvironmentSnapshotDocumentsStubsTest` (`PinWright.infra.wiki_handler.MethodPage.EnvironmentSnapshotDocumentsStubs`) renders both per-method pages via `WikiHandler::RenderPage` and asserts each carries "NOT_IMPLEMENTED" (pins the summary prefix), then renders the `environment` namespace page and asserts the overlay-exclusive "Snapshot round-trip is not implemented" / "Do not read the prelude" / `actor.list`+`property.get` markers (pins the overlay section) — fails if either the summary prefix or the overlay section is reverted. Not compiled here (a later phase compiles + runs).
