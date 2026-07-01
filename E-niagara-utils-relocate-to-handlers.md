---
id: E-niagara-utils-relocate-to-handlers
title: "Relocate Niagara-only helpers from Utils/ to Handlers/Niagara/"
status: DONE
severity: Low
category: ergonomic
tags: [utils, niagara, code-organization]
---

# Relocate Niagara-only helpers from Utils/ to Handlers/Niagara/

`Utils/NiagaraGraphResetUtils.{h,cpp}` and `Utils/NiagaraInstanceUtils.{h,cpp}`
are domain-specific (Niagara-only) but live in the generic `Utils/` directory
alongside cross-cutting helpers like `JsonUtils`, `PathUtils`, `AssetUtils`,
`ClassUtils`, etc. Their entire surface is Niagara classes
(`UNiagaraGraph`, `UNiagaraNodeOutput`, `UNiagaraSystem`, `ENiagaraScriptUsage`).
Every caller lives in `Handlers/Niagara/` (or the Niagara test suite). The
files exist only because `FNiagaraStackGraphUtilities::ResetGraphForOutput`
and `FNiagaraEditorUtilities::KillSystemInstances` aren't `NIAGARAEDITOR_API`
exported, so they had to be vendored inline — that's an implementation
detail of the Niagara handlers, not a cross-cutting utility.

There is already precedent for in-domain helpers under `Handlers/Niagara/`:
`NiagaraJsonHelpers.{h,cpp}`, `NiagaraDecompileHelpers.h`,
`NiagaraParameterRenameUtils.{h,cpp}`, `NiagaraResetModuleInputHelpers.h`,
`NiagaraSetModuleScriptHelpers.h`, `NiagaraDumpBuilder.{h,cpp}`,
`NiagaraModelBuilder.{h,cpp}`, `NiagaraEditTypes.{h,cpp}`,
`NiagaraSystemViewModelCache.{h,cpp}`. The Utils/ pair is the odd one out.

## File inventory

Files to move:
- `Source/EditorAutomationRpcGateway/Private/Utils/NiagaraGraphResetUtils.h`
- `Source/EditorAutomationRpcGateway/Private/Utils/NiagaraGraphResetUtils.cpp`
- `Source/EditorAutomationRpcGateway/Private/Utils/NiagaraInstanceUtils.h`
- `Source/EditorAutomationRpcGateway/Private/Utils/NiagaraInstanceUtils.cpp`

Target location:
- `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraGraphResetUtils.{h,cpp}`
- `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraInstanceUtils.{h,cpp}`

Header surface is tiny (one namespace `EditorAutomationNiagara` with 3 free
functions total: `ResetGraphForOutput`, `FindParameterMapPin`,
`KillSystemInstances`).

## Callers

Five non-self include sites in `Handlers/Niagara/`:
- `Handlers/Niagara/NiagaraEditHandler.cpp` (both)
- `Handlers/Niagara/NiagaraAdvancedEditHandler.cpp` (both)
- `Handlers/Niagara/NiagaraCurveHandler.cpp` (instance only)
- `Handlers/Niagara/NiagaraHandler.cpp` (instance only)
- `Handlers/Niagara/NiagaraJsonHelpers.cpp` (instance only)

Plus one test:
- `Tests/Niagara/TestNIRFixtures.cpp` (instance only)

And the two self-includes in the `.cpp`s themselves. Total 8 `#include`
lines to retarget. All callers are Niagara-domain — no cross-domain caller
justifies keeping them in `Utils/`.

## Why move

- **Cohesion** — colocate helpers with their only consumers; matches the
  existing `Handlers/Niagara/` pattern for `NiagaraJsonHelpers` et al.
- **Reduces Utils/ sprawl** — Utils/ should be cross-cutting only;
  Niagara-only files there mislead browsers about scope.
- **Discoverability** — anyone working on a Niagara handler finds the
  vendored helpers in the same folder, not in a generic bucket.

## Risk

Low — pure relocation. Include paths flip from `"Utils/Niagara*.h"` to
`"Handlers/Niagara/Niagara*.h"` (or whatever the local convention dictates,
e.g. relative `"Niagara*.h"` from inside the same folder). No public API
changes, no signature changes, no behavior changes. PCH/Unity build is
unaffected. The `.Build.cs` does not reference `Utils/` directly
(`PrivateIncludePaths` only adds Niagara/Chooser engine internals).

## Fix

1. `git mv` the four files into `Handlers/Niagara/`.
2. Update the two self-includes inside the moved `.cpp`s.
3. Update the six caller-side includes (five handlers + one test).
4. Decide on the in-handler include style — either `"Handlers/Niagara/NiagaraGraphResetUtils.h"`
   (matches existing absolute-relative-to-Private style used elsewhere) or
   `"NiagaraGraphResetUtils.h"` (sibling-relative, matches some local
   patterns). `NiagaraJsonHelpers.cpp` itself uses
   `"Utils/NiagaraInstanceUtils.h"` so the absolute-relative style is the
   prevailing convention — go with that.

Suggested bundling with other Utils/ relocations (file separately):
- `AssetDumpBuilder` / `AssetDumpWriter` → `Handlers/Asset/`
- `ActorDescribeBuilder` → `Handlers/Actor/`

This entry covers only the Niagara scope.

## History
- `#1-initial-scope` `OPEN` reporter — `Utils/NiagaraGraphResetUtils.{h,cpp}` and `Utils/NiagaraInstanceUtils.{h,cpp}` are Niagara-only vendored substitutes for un-exported engine helpers; all 6 caller files live under `Handlers/Niagara/` (5 handlers + 1 test). Precedent for in-domain helpers already exists in that folder (`NiagaraJsonHelpers`, `NiagaraDecompileHelpers`, etc.). Propose `git mv` to `Handlers/Niagara/` + update 8 include sites. Pure relocation, no behavior change.
- `#2-relocated` `IN-REVIEW` developer — `git mv`'d all four files (`NiagaraGraphResetUtils.{h,cpp}`, `NiagaraInstanceUtils.{h,cpp}`) from `Utils/` to `Handlers/Niagara/`. Retargeted all 10 `#include` sites to the absolute-relative `"Handlers/Niagara/..."` style: 2 self-includes in the moved `.cpp`s plus 8 caller sites (`NiagaraEditHandler.cpp`, `NiagaraAdvancedEditHandler.cpp`, `NiagaraCurveHandler.cpp`, `NiagaraHandler.cpp`, `NiagaraJsonHelpers.cpp`, `Tests/Niagara/TestNIRFixtures.cpp`). No behavior change; ergonomic ticket, no regression test.
- `#3-review-scope-check` `IN-REVIEW` developer — Re-reviewed against the `[spec]` drive-by complaint. Confirmed this ticket's edits are confined to exactly the 11 in-scope paths: the 4 relocated `Handlers/Niagara/Niagara{GraphReset,Instance}Utils.{h,cpp}` files (relocation + self-include flip), the 6 caller files (`NiagaraEditHandler.cpp`, `NiagaraAdvancedEditHandler.cpp`, `NiagaraCurveHandler.cpp`, `NiagaraHandler.cpp`, `NiagaraJsonHelpers.cpp`, `Tests/Niagara/TestNIRFixtures.cpp`), and this board file. Grep confirms zero remaining `"Utils/Niagara*Utils.h"` references and all 10 include sites point at `Handlers/Niagara/`. The ~100 other modified files in the working tree (AGIR, WikiHandler, Bpir, PropertyUtils, etc.) are sibling-ticket work in a shared tree — NOT touched by this ticket, and not safe to revert. The drive-by complaint is a commit-staging concern: the committer must stage only these 11 paths into this ticket's commit. No code change required to resolve.
- `#4-verify-fix` `DONE` tester — Verified via file-system state (relocation ticket, surface is file/doc only). Glob confirms all 4 files (`NiagaraGraphResetUtils.{h,cpp}`, `NiagaraInstanceUtils.{h,cpp}`) now live under `Handlers/Niagara/` and zero remain under `Utils/`. Grep confirms 0 stale `Utils/Niagara*Utils.h` includes in code (remaining matches are board-doc text only) and all 10 include sites resolve to `"Handlers/Niagara/..."`: 2 self-includes + 8 callers (`NiagaraEditHandler` both, `NiagaraAdvancedEditHandler` both, `NiagaraCurveHandler`/`NiagaraHandler`/`NiagaraJsonHelpers`/`TestNIRFixtures` instance-only). `.Build.cs` has no `PrivateIncludePaths` ref to `Utils/`, matching the risk note.
