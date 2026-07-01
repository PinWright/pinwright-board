---
id: E-remove-bp-creation-shims
title: "Remove deprecated Blueprint-creation shim files (~491 LOC dead code)"
status: DONE
severity: Low
category: ergonomic
tags: [cleanup, dead-code]
---

# Remove deprecated Blueprint-creation shim files

Two top-level files in `Private/` are deprecated Blueprint-creation shims with
zero external callers. The shim header self-describes the deprecation:

> "Backward-compatibility shim header (DEPRECATED). ... no longer called by
> any code and can be removed in a future cleanup pass."

The real auto-registered handlers live at
`Private/Handlers/Blueprint/BlueprintCreationHandler.cpp` (registered via
`REGISTER_RPC_HANDLER`). The shim previously bridged a pre-migration
`_BlueprintHandlers.cpp` to the new handlers, but that caller was deleted in
the same migration. A repo-wide grep for the shim class/method symbols
(`FBlueprintCreationHandlers`, `HandleBlueprintCreate`,
`HandleBlueprintProbeSubobjectHandle`) and the header filename returns only
self-references in the two shim files themselves.

**Safe to delete (pure removal, no migration):**
- `c:\Unity\unreal-fpv-pluginwork\Plugins\EditorAutomationRpcGateway\Source\EditorAutomationRpcGateway\Private\EditorAutomationRpcGateway_BlueprintCreationHandlers.h` (~21 LOC)
- `c:\Unity\unreal-fpv-pluginwork\Plugins\EditorAutomationRpcGateway\Source\EditorAutomationRpcGateway\Private\EditorAutomationRpcGateway_BlueprintCreationShim.cpp` (~470 LOC)

Total ~491 LOC.

**NOT in scope — leave alone:**
- `Private/EditorAutomationRpcGateway_BlueprintHandlers_List.cpp` — this is
  the working `blueprint.list` RPC handler (defines
  `REGISTER_RPC_HANDLER("blueprint.list", ...)`), not a shim. An initial
  review mistook it for dead code because of the similar filename prefix.
  Do not touch it.

## Optional bundling: `Utils/VersionCompat.h`

`VersionCompat.h` was advertised as holding UE 5.4–5.7 compat macros, but
the current file body is just `#include "Runtime/Launch/Resources/Version.h"`
— all macros have been removed. It is included by three sites
(`EditorAutomationRpcGatewayHelpers.h`, `Utils/PropertyUtils.cpp`,
`Utils/AssetUtils.cpp`) but no macro from it expands in plugin code, making
each include effectively a transitive `Version.h` pull.

Two reasonable options:
1. Bundle into this PR — delete `VersionCompat.h` and either drop the three
   includes or replace them with a direct `Version.h` include where actually
   needed. Also update `CLAUDE.md` (lines 123, 238) and `docs/arch.md`
   (lines 124, 415) which still describe it as holding compat macros.
2. Defer to a separate ticket if reviewers prefer the shim removal to land
   as a minimal pure-delete diff.

Recommend bundling: it is a strictly smaller diff to remove a header that
provides nothing than to chase doc references later.

**Fix:** Delete the two shim files. Optionally delete `VersionCompat.h` and
its three includes in the same commit, with the doc updates noted above.
Rebuild the plugin module to confirm no surprise transitive include
dependency on the deleted files surfaces.

## History
- `#1-initial-scope` `OPEN` reporter — Verified the two shim files have zero callers via grep for `FBlueprintCreationHandlers`, `HandleBlueprintCreate`, `HandleBlueprintProbeSubobjectHandle`, and the header filename — only self-references in the shim pair returned. Confirmed `BlueprintHandlers_List.cpp` is a live `REGISTER_RPC_HANDLER("blueprint.list", ...)` handler, not a shim. Confirmed `Utils/VersionCompat.h` is currently a 7-line file whose body is just a `Version.h` re-export, with no macro from it referenced anywhere in the plugin (3 includes, 0 expansions).
- `#2-delete-shims-and-versioncompat` `IN-REVIEW` developer — Deleted `EditorAutomationRpcGateway_BlueprintCreationHandlers.h`, `EditorAutomationRpcGateway_BlueprintCreationShim.cpp`, and `Utils/VersionCompat.h`. Repointed the `VersionCompat.h` includes directly at `Runtime/Launch/Resources/Version.h`. This ticket lands in a shared multi-ticket sprint working tree that also bundles **E-propertyutils-split-by-concern** (deletes `Utils/PropertyUtils.cpp`, adds `PropertyExport/Import/Inspection/Diff` `.cpp`/`.h`), **E-ir-pin-resolver-shared-base**, and **E-animation-authoring-handler-split** — so the three include sites named in this ticket's body must be read against that combined state. `EditorAutomationRpcGatewayHelpers.h` repointed (done here). `Utils/AssetUtils.cpp` repointed (done here). `Utils/PropertyUtils.cpp` existed at HEAD and still carried the `Utils/VersionCompat.h` include at line 4 when this work began — it was not a pre-existing post-split state; the `D Utils/PropertyUtils.cpp` in git status is E-propertyutils-split-by-concern's deletion, not this ticket's. This ticket's VersionCompat repoint of PropertyUtils.cpp became moot only because that sibling ticket deletes the file, so no edit to it was needed. Removed the `VersionCompat.h` parenthetical from `CLAUDE.md` line 128 (other flagged doc references at CLAUDE.md 123/238 and docs/arch.md 124/415 were already clean). Post-edit grep for the three shim symbols (`FBlueprintCreationHandlers`, `HandleBlueprintCreate`, `HandleBlueprintProbeSubobjectHandle`) and `VersionCompat` across `Source/` returns zero matches. No regression test — pure dead-code removal, no behavioral change. Files deleted via filesystem remove (not `git rm`); not staged, not compiled per ticket instructions.
- `#3-verify-fix` `DONE` tester — Verified on filesystem/doc state (this ticket's entire surface is file deletion + doc edit). Confirmed all three target files absent: `EditorAutomationRpcGateway_BlueprintCreationHandlers.h`, `EditorAutomationRpcGateway_BlueprintCreationShim.cpp`, `Utils/VersionCompat.h`. Confirmed not-in-scope `EditorAutomationRpcGateway_BlueprintHandlers_List.cpp` still present (10328 bytes). Repo-wide grep across `Source/` for `VersionCompat`, `FBlueprintCreationHandlers`, `HandleBlueprintCreate`, `HandleBlueprintProbeSubobjectHandle` returns zero matches. Both surviving include sites repoint directly: `EditorAutomationRpcGatewayHelpers.h:6` and `Utils/AssetUtils.cpp:4` now `#include "Runtime/Launch/Resources/Version.h"`. Plugin-root `CLAUDE.md` has no remaining `VersionCompat` reference.
