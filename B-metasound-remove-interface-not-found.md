---
id: B-metasound-remove-interface-not-found
title: "remove_metasound_interface reports INTERFACE_NOT_FOUND for an attached-but-non-modifiable default interface (misleading error text)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [audio, metasound, authoring, interface, remove, error-message]
---

# remove_metasound_interface reports INTERFACE_NOT_FOUND for an attached-but-non-modifiable default interface (misleading error text)

`audio.authoring.remove_metasound_interface` fails to detach `UE.OutputFormat.Mono`
from a `UMetaSoundSource` and returns
`[INTERFACE_NOT_FOUND] Interface 'UE.OutputFormat.Mono' is not attached to this MetaSound`.
The interface IS in fact attached (it is the Source's mandatory default output
format), so the **error text is misleading**: it says "not attached" when the
real reason removal is refused is that the interface is **non-modifiable** for the
asset's UClass — a default Source output format is not user-detachable by design.

The "not attached" text contradicts the observable state (`describe_metasound`
shows the `UE.OutputFormat.Mono.Audio:0` Audio output vertex present), which drove
agents into path-form trial-and-error and source dives chasing a non-existent
keying bug.

## Root cause

Two findings, established against UE 5.7 engine source
(`MetasoundFrontendDocumentBuilder.cpp`, `MetasoundOutputFormatInterfaces.cpp`):

1. **The repro interface is attached-but-immutable, not absent.** For a
   `UMetaSoundSource`, every output-format interface (including `UE.OutputFormat.Mono`)
   is registered with `bIsModifiable = false`
   (`MetasoundOutputFormatInterfaces.cpp:138` — `{UMetaSoundSource, false, bIsSourceDefault}`).
   `RemoveInterface(FName)` therefore returns **false** at
   `MetasoundFrontendDocumentBuilder.cpp:5431` ("is not set to be modifiable for
   given UClass") — *after* confirming the interface IS present in
   `Document.Interfaces` (it passes the `Contains` check at :5416). The handler's
   `if (!bRemoved)` branch (`MetaSoundInterfaceHandler.cpp:259`) then re-emits the
   same misleading `INTERFACE_NOT_FOUND` "is not attached" text.

2. **The original "version-keyed vs name-keyed" diagnosis was wrong** (recorded for
   the record — do NOT implement it). `IsInterfaceDeclared(FName)`,
   `IsInterfaceDeclared(FMetasoundFrontendVersion)`, `RemoveInterface(FName)` and the
   handler's pre-check ALL resolve `FindInterfaceWithHighestVersion` and gate on the
   identical `Document.Interfaces.Contains(<highest version>)` set
   (`MetasoundFrontendDocumentBuilder.cpp:4463-4478, 5414-5416`). The pre-check can
   never disagree with `RemoveInterface`'s own gate, so the FName-overload
   "alternative" is a no-op. **Dropping the pre-check is a regression**:
   `RemoveInterface(FName)` returns **true** (`:5419`) for a genuinely-absent
   *modifiable* interface ("skipping remove request"), so removing the pre-check
   would make absent-interface removals report fake success and would break the
   existing `FRemoveMetaSoundInterfaceRejectsAbsentRegisteredInterfaceTest`.

So this is not a logic bug in the gate — the gate is correct. It is a **diagnostic
message bug**: when `RemoveInterface` fails for an interface that IS declared, the
handler must report the real reason (non-modifiable / mandatory default interface)
instead of the false "is not attached".

**Fix:** keep the version-keyed `IsInterfaceDeclared` pre-check (it correctly
rejects genuinely-absent interfaces — the negative test depends on it). In the
`if (!bRemoved)` branch, the interface WAS declared (it passed the pre-check), so
the only remaining engine failure reason is non-modifiability for the asset's
UClass: emit a distinct, accurate error (e.g. `INTERFACE_NOT_REMOVABLE`) stating
the interface is a mandatory/default interface for this asset class and cannot be
detached — rather than the false `INTERFACE_NOT_FOUND` "is not attached". Optionally
detect this up front via the interface's `UClassOptions` (`bIsModifiable == false`
for the asset's class) to give the accurate message before mutating. The genuine
attach → detach round-trip for a *modifiable* interface already works (the existing
add round-trip test plus the non-modifiable analysis confirm the gate is sound).

This shipped with the `add/remove/list` feature
([`F-metasound-no-interface-attach`](F-metasound-no-interface-attach.md), DONE),
whose verification (#8) only exercised the **add** round-trip — the **remove** path
error text was never exercised live, so the misleading message went undetected.

## Repro (verbatim, replayed via mcp__editor-automation__call)

1. `audio.authoring.create_metasound` `{name: "MS_OracleReplay", path: "/Game/Audio/MetaSounds"}`
   → `{assetPath: "/Game/Audio/MetaSounds/MS_OracleReplay", ..., assetClass: "MetaSoundSource"}` (a `UMetaSoundSource`).
2. `audio.authoring.add_metasound_interface` `{... interfaceName: "UE.OutputFormat.Mono"}`
   → success (the Mono output format is the Source's default — it is present after).
3. `audio.authoring.describe_metasound` → `rootGraph.interface.outputs` contains
   `{name: "UE.OutputFormat.Mono.Audio:0", typeName: "Audio"}` — interface IS attached.
4. `audio.authoring.remove_metasound_interface` `{... interfaceName: "UE.OutputFormat.Mono"}`
   → **`[INTERFACE_NOT_FOUND] Interface 'UE.OutputFormat.Mono' is not attached to this MetaSound`**
   — MISLEADING: it IS attached; it is just a non-modifiable default for `UMetaSoundSource`.
5. Fully-qualified `.MS_OracleReplay` path → identical (rules out path resolution).

Note: the repro picked the one interface class that cannot be removed from a Source
(a default output format), so it does NOT establish that detach is broken for a
*modifiable* interface — only that the rejection text is wrong.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed via mcp__editor-automation__call: created MS_OracleReplay, attached `UE.OutputFormat.Mono` (add returned success, describe shows the `UE.OutputFormat.Mono.Audio:0` Audio output vertex), then `remove_metasound_interface` returned `[INTERFACE_NOT_FOUND] Interface 'UE.OutputFormat.Mono' is not attached to this MetaSound` for the genuinely-attached interface — both on the bare assetPath and the fully-qualified `.MS_OracleReplay` form (rules out path resolution). Root cause in `MetaSoundInterfaceHandler.cpp` remove handler (~line 246): a version-keyed `Builder.IsInterfaceDeclared(FMetasoundFrontendVersion)` pre-check rejects the interface even though `add` attaches name-keyed via `AddInterface(FName)` and the real removal `RemoveInterface(FName)` is name-keyed too. Fix: drop the version-keyed pre-check (the name-keyed `RemoveInterface(FName)` return value already covers genuine absence) or switch to the name-only `IsInterfaceDeclared(FName)` overload. Sibling DONE feature ticket F-metasound-no-interface-attach verified only the add round-trip, never remove.
- `#2-process-confusing-error-drives-trial-and-error` `OPEN` reporter — Independent reproduction from a struggle-audited `audio.authoring.list_metasound_interfaces` task (vehicle-engine-loop scenario, asset `MS_EngineLoop` at `/Game/Audio/MetaSounds`; steps 1-5 all passed first try — `UE.OutputFormat.Mono` attached, describe showed the `UE.OutputFormat.Mono.Audio:0` Audio vertex plus the custom `RPM`/`Throttle` Float inputs). **PROCESS cost beyond the bug itself:** because the `[INTERFACE_NOT_FOUND] … is not attached` error flatly contradicts `describe_metasound` (which proves the vertex IS present), the agent could not trust the error and burned three failed `remove_metasound_interface` calls plus an intervening `describe_metasound` doing path-form trial-and-error — bare assetPath, re-describe to re-confirm attachment, bare assetPath again, then fully-qualified `MS_EngineLoop.MS_EngineLoop` — exactly the path-resolution hypothesis the `#6` failure mode of `F-metasound-no-interface-attach` had primed (the agent reasonably suspected the bare-vs-qualified path trap, since add accepts the bare form). Only after all path forms failed identically did the agent fall back to reading the plugin handler (`MetaSoundInterfaceHandler.cpp`) and engine (`MetasoundFrontendDocumentBuilder.cpp/.h`) source to diagnose the version-keyed vs name-keyed mismatch — a source-dive a single-line RPC author should never need. Self-report (verbatim): "the error message ('not attached') flatly contradicts describe_metasound … a discoverability gap"; "I did NOT fall back to python.execute (that would mask the bug)." Confirms #1's root cause and severity; no separate ticket filed — the confusing-error and path-form-trial-and-error friction are both intrinsic to this bug and resolve when the version-keyed pre-check is dropped. Total: 3 failed removes + 1 corroborating describe wasted on a single intent the success check required.
- `#3-reworded-and-fixed` `IN-REVIEW` developer — REWORDED then fixed. Both the correctness and adversarial validity lenses (re-verified by me against UE 5.7 engine source) disproved the original "version-keyed vs name-keyed" root cause and BOTH proposed fixes: `IsInterfaceDeclared(FName)`/`(version)`, the handler pre-check, and `RemoveInterface(FName)` all resolve `FindInterfaceWithHighestVersion` and gate on the identical `Document.Interfaces.Contains(<highest version>)` set (`MetasoundFrontendDocumentBuilder.cpp:4463-4478, 5414-5416`), so the FName-overload alternative is a no-op; and dropping the pre-check is a REGRESSION because `RemoveInterface(FName)` returns **true** (`:5419`) for a genuinely-absent *modifiable* interface (fake success) and would break the existing `FRemoveMetaSoundInterfaceRejectsAbsentRegisteredInterfaceTest`. The real defect is misleading error TEXT: for the repro asset (`MS_OracleReplay`, a `UMetaSoundSource`), `UE.OutputFormat.Mono` is the Source's mandatory NON-modifiable default output format (`MetasoundOutputFormatInterfaces.cpp:138` registers `{UMetaSoundSource, bIsModifiable=false}`), so it is attached-but-immutable; `RemoveInterface` returns false at `:5431` ("not set to be modifiable") AFTER confirming it IS present, and the handler re-emitted the false "is not attached". Reworded title/body/Fix/severity (High→Medium, category bug, tag remove→error-message) to target the message, not a keying bug. FIX (`MetaSoundInterfaceHandler.cpp` remove handler): kept the `IsInterfaceDeclared` pre-check (genuine-absence → `INTERFACE_NOT_FOUND`, the negative test depends on it); added `IsInterfaceNonModifiableForAsset(Interface, MetaSound)` helper that reads the interface's `UClassOptions` for the asset's UClass; when the interface IS declared but non-modifiable for the asset class, send the new accurate `INTERFACE_NOT_REMOVABLE` ("is attached but is a mandatory/default interface … and cannot be detached") instead of the false "not attached"; the post-`RemoveInterface` `!bRemoved` branch now also reports `INTERFACE_NOT_REMOVABLE` (interface was proven declared) under the search-engine path, falling back to `INTERFACE_NOT_FOUND` only without the search engine. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Audio/MetaSound/MetaSoundInterfaceHandler.cpp`. TEST: added `FRemoveMetaSoundInterfaceReportsNonModifiableDefaultTest` in `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestMetaSoundInterfaceOps.cpp` — builds a transient `UMetaSoundSource` with `UE.OutputFormat.Mono` declared, invokes the production `audio.authoring.remove_metasound_interface` handler, and asserts the error is `INTERFACE_NOT_REMOVABLE` and explicitly NOT `INTERFACE_NOT_FOUND` (fails if the misleading-text fix is reverted). The existing negative test (absent registered interface → `INTERFACE_NOT_FOUND`) is unaffected — the pre-check is preserved. Not compiled/tested here (later phase).
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
