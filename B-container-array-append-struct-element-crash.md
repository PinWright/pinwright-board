---
id: B-container-array-append-struct-element-crash
title: "container.array.append crashes the editor (EXCEPTION_ACCESS_VIOLATION) on a struct-element array — handler passes the object base, not the new element ptr, to ApplyJsonValueToProperty"
status: IN-REVIEW
severity: Critical
category: bug
tags: [container, array, append, crash, property-import, wrong-container-ptr]
encounters: 1
lastSeen: 2026-07-02T14:02:24.1156365+03:00
---

# container.array.append crashes the whole editor when the array's inner is a struct (e.g. `TArray<FDirectoryPath>`)

Appending a single element to a struct-typed array property crashes the editor
with an unhandled `EXCEPTION_ACCESS_VIOLATION` — the process dies immediately
(the very first `container.array.append` reset the RPC connection with
`WinError 10054`; every later probe got `WinError 10061` connection-refused),
so nothing landed and the editor never recovered.

Repro (verbatim, from the attempt):
- `property.get objectPath=/Script/Engine.Default__AssetManagerSettings propertyName=DirectoriesToExclude` → `[]` (array starts empty, confirmed).
- `container.array.append` with
  `{objectPath: "/Script/Engine.Default__AssetManagerSettings", propertyName: "DirectoriesToExclude", value: {Path: "/Game/Developers"}}`
  → **editor crash**. `DirectoriesToExclude` is a `TArray<FDirectoryPath>`; the
  value is the natural per-element shape (the wiki even names this settings
  object + property as a copy-paste target).

## Real crash (ground truth, from Saved/Crashes/…/EAContentExamples57.log)

```
LogWindows: Error: === Critical error: ===
LogWindows: Error: Fatal error!
LogWindows: Error: Unhandled Exception: EXCEPTION_ACCESS_VIOLATION writing address 0x00007ffe7891f528
LogWindows: Error: [Callstack] 0x... VCRUNTIME140.dll!UnknownFunction []
LogWindows: Error: [Callstack] 0x... UnrealEditor-PinWright.dll!FString::operator=() [C:\UE_5.7\Engine\Source\Runtime\Core\Public\Containers\UnrealString.h.inl:80]
LogWindows: Error: [Callstack] 0x... UnrealEditor-PinWright.dll!ApplyJsonValueToProperty() [PropertyImport.cpp:318]
LogWindows: Error: [Callstack] 0x... UnrealEditor-PinWright.dll!ApplyJsonValueToProperty() [PropertyImport.cpp:828]
LogWindows: Error: [Callstack] 0x... UnrealEditor-PinWright.dll!AutoHandler_334_() [UtilityPropertyHandler.cpp:1892]   (== container.array.append)
LogWindows: Error: [Callstack] 0x... UnrealEditor-PinWright.dll!FRpcDispatcher::DrainAutoRegistrations'...lambda [RpcDispatcher.cpp:296]
LogWindows: Error: [Callstack] 0x... UnrealEditor-PinWright.dll!FRpcDispatcher::ProcessRequest() [RpcDispatcher.cpp:479]
LogWindows: Error: [Callstack] 0x... UnrealEditor-PinWright.dll!FMcpTransport::Start'...lambda [McpTransport.cpp:566]
LogWindows: Error: [Callstack] 0x... UnrealEditor-HTTPServer.dll!FHttpConnection::ProcessRequest() ...
```

## Root cause (verified in plugin source — not response-derived)

`container.array.append` computes the correct new-element pointer but then hands
the WRONG container base to `ApplyJsonValueToProperty`.

`Plugins/PinWright/Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp:1886`:
```cpp
FScriptArrayHelper Helper(ArrayProp, ArrayProp->ContainerPtrToValuePtr<void>(TargetContainer));
const int32 NewIndex = Helper.AddValue();
void* ElemPtr = Helper.GetRawPtr(NewIndex);     // <-- correct element ptr, computed but ignored below
FProperty* Inner = ArrayProp->Inner;

FString ConversionError;
if (!ApplyJsonValueToProperty(TargetContainer, Inner, ValueField, ConversionError))   // line 1892: passes TargetContainer, NOT ElemPtr
```

`ApplyJsonValueToProperty(Container, Property, …)` treats its first arg as the
**container** for `Property` and dereferences it as `Property->ContainerPtrToValuePtr(Container)`
(effectively `Container + Property->Offset_Internal`). But `Inner` is the array's
inner property — its offset is 0 relative to an ARRAY ELEMENT, not relative to
the owning object/struct. Passing `TargetContainer` (the object base) makes the
import compute `TargetContainer + garbage/mismatched offset` and write there.

For a struct inner (`FDirectoryPath`), the recursion descends into its `Path`
sub-field:
`Plugins/PinWright/Source/PinWright/Private/Utils/PropertyImport.cpp:828`:
```cpp
if (!ApplyJsonValueToProperty(StructValuePtr, SubProp, Pair.Value, SubError))
```
and finally `Plugins/PinWright/Source/PinWright/Private/Utils/PropertyImport.cpp:318`:
```cpp
SP->SetPropertyValue_InContainer(TargetContainer, ValueField->AsString());
```
`SetPropertyValue_InContainer` does `FString::operator=` into a pointer that
resolves to unowned memory → the AV in `FString::operator=` seen in the
callstack.

The direct-assignment fallback in the same handler (lines 1896+) correctly uses
`ElemPtr` (`*reinterpret_cast<FString*>(ElemPtr) = …`) — but it is only reached
if `ApplyJsonValueToProperty` *returns false*. Here the bad-pointer call crashes
before it can return, so the fallback never runs, and for scalar inners the bad
call would silently corrupt memory instead of crashing.

## What it should do

Pass the newly-added element pointer, not the object base:
`ApplyJsonValueToProperty(ElemPtr, Inner, ValueField, …)` — OR give
`ApplyJsonValueToProperty` a "value ptr" entry that skips the
`ContainerPtrToValuePtr` step for an inner property (offset 0). The same
element-vs-container mistake should be audited in the sibling
`container.array.insert` / `container.array.set` handlers.

severity rationale: impact=crash × reach=every-session -> Critical
(`container.array.append` is a normal, wiki-advertised write verb; a
struct-element array is a common target, and the editor dies unrecoverably.)

## History
- `#2-fix-append-passes-element-ptr` `IN-REVIEW` developer — Fixed root cause: `container.array.append` now passes `ElemPtr` (the newly-added element), not `TargetContainer` (the object base), to `ApplyJsonValueToProperty` at `Plugins/PinWright/Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp:1892`, mirroring the already-correct sibling `container.array.set` (`:2197`) and `container.array.get` (`:2133`). Verified in current source: line 1892 still passed `TargetContainer` (last touched by the unrelated `E-property-get-deprecated-field-silent-null`), so this was live; `set`/`get` were fixed by `F-container-map-value-type-coverage` but that fix explicitly skipped `append`. Confirmed the crash mechanism matches the captured callstack (struct inner `FDirectoryPath` → `PropertyImport.cpp:815` `StructValuePtr == TargetContainer` since `Inner->Offset_Internal == 0` → `:828` recurse → `:318` `FString::operator=` at the object base → AV). Added a struct-inner regression test that appends `{Label,Weight}` to a reflected `TArray<FTestContainerPose>` and asserts the value lands in the new element (pre-fix: editor crash / value never lands). Files: `Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp`, `Source/PinWright/Private/Tests/Utility/TestContainerValueTypeCoverage.cpp`. Test: `PinWright.container.array.append.StructElementNoCrash`. (The pre-existing `container.array.append.ValidParamsNoCrash` targets `/Game/NonExistentObject` and bails at `OBJECT_NOT_FOUND` before the struct path, so it never caught this.)
- `#1-initial-repro` `OPEN` reporter — Editor crash. `container.array.append value={Path:/Game/Developers}` on `AssetManagerSettings.DirectoriesToExclude` (`TArray<FDirectoryPath>`) killed the editor with `EXCEPTION_ACCESS_VIOLATION writing` in `FString::operator=`, callstack `PropertyImport.cpp:318 <- :828 <- UtilityPropertyHandler.cpp:1892 (container.array.append)`. Root cause verified in source: handler passes `TargetContainer` (object base) instead of `ElemPtr` (new element) to `ApplyJsonValueToProperty` at `UtilityPropertyHandler.cpp:1892`, so the struct import writes an `FString` to a mismatched-offset pointer. Fatal crash dump captured this iteration.
