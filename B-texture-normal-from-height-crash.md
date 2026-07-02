---
id: B-texture-normal-from-height-crash
title: "texture.create_normal_from_height crashes the editor (null deref of unchecked LockReadOnly mip pointer)"
status: IN-REVIEW
severity: Critical
category: bug
tags: [texture, crash, null-deref, normal-map, bulkdata, lockreadonly]
encounters: 2
lastSeen: 2026-07-01T09:04:47.4423899+03:00
---

# texture.create_normal_from_height null-derefs a freshly-built source mip and crashes the editor

`texture.create_normal_from_height` loads the source height texture, null-checks
the `UTexture2D*`, then locks `Mips[0].BulkData` for read and immediately indexes
the returned pointer **without checking it for null**. For a source texture whose
platform mip has just been built/compressed (e.g. created via
`texture.create_pattern_texture`, then encoded `TFO_AutoDXT`), `LockReadOnly()`
returns `nullptr` and the very first pixel read dereferences address `0x0`,
taking down the whole editor with an `EXCEPTION_ACCESS_VIOLATION`. The RPC never
returns a JSON error — the connection is reset (WinError 10054) mid-call and every
subsequent call is refused (WinError 10061).

severity rationale: impact=crash × reach=every-session -> Critical

## Why it matters

Generating a normal map from a height texture is a primary, documented texture
workflow. The input here is entirely valid — a real `UTexture2D` that loads fine
and passes the handler's own null check at line 664 — yet the call hard-crashes
the editor and loses all unsaved state. A crash on a valid documented path is the
worst possible failure mode (no error to catch, no recovery).

## Root cause (source)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureHandler.cpp`,
`ExecuteTextureAction`, the `create_normal_from_height` branch (sub-action matched
at line 624). The source texture pointer IS validated:

```cpp
// line 663-667
UTexture2D* HeightMap = Cast<UTexture2D>(StaticLoadObject(UTexture2D::StaticClass(), nullptr, *SourceTexture));
if (!HeightMap)
{
    TEXTURE_ERROR_RESPONSE(FString::Printf(TEXT("Failed to load height map: %s"), *SourceTexture));
}
```

but the locked mip pointer is NOT — the guilty lines (719-725):

```cpp
// Lock source texture for reading
FTexture2DMipMap& HeightMip = HeightMap->GetPlatformData()->Mips[0];
const uint8* HeightPixels = static_cast<const uint8*>(HeightMip.BulkData.LockReadOnly());

for (int32 i = 0; i < Width * Height; i++)
{
    // BGRA format: index 0=B, 1=G, 2=R, 3=A
    uint8 B = HeightPixels[i * 4 + 0];   // <-- crash: HeightPixels == nullptr
```

`LockReadOnly()` returns `nullptr` because the freshly built `Mips[0].BulkData`
for a compressed/streamed platform texture is not CPU-resident, but line 725
dereferences it unconditionally. (Secondary issue: the loop also assumes a raw
8-bit BGRA layout, which is wrong for a DXT-compressed platform mip even when the
data IS resident — but the immediate fatal bug is the missing null check on the
`LockReadOnly` result.) `GetPlatformData()` itself is also unchecked.

## Crash evidence (verbatim)

`Saved/Crashes/UECC-Windows-BC8389E54D7D589AFF91F9A00F49EF66_0000/CrashContext.runtime-xml`:

```
ErrorMessage: Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x0000000000000000
CrashType:    Crash
GameName:     UE-EAContentExamples57
```

Callstack (top frames, verbatim):

```
UnrealEditor_PinWright!ExecuteTextureAction() [...\Handlers\Material\TextureHandler.cpp:725]
UnrealEditor_PinWright!RunTextureAction() [...\Handlers\Material\TextureHandler.cpp:2699]
UnrealEditor_PinWright!AutoHandler_310_() [...\Handlers\Material\TextureHandler.cpp:2772]
UnrealEditor_PinWright!`FRpcDispatcher::DrainAutoRegistrations'::`6'::<lambda_1>::operator()() [...\Dispatch\RpcDispatcher.cpp:296]
UnrealEditor_PinWright!FRpcDispatcher::ProcessRequest() [...\Dispatch\RpcDispatcher.cpp:479]
UnrealEditor_PinWright!`FMcpTransport::Start'::`2'::<lambda_1>::operator()() [...\Transport\McpTransport.cpp:566]
UnrealEditor_HTTPServer!FHttpConnection::ProcessRequest() [...\HTTPServer\Private\HttpConnection.cpp:213]
UnrealEditor_HTTPServer!FHttpListener::Tick() [...\HTTPServer\Private\HttpListener.cpp:163]
UnrealEditor_Core!FTSTicker::Tick() [...\Core\Private\Containers\Ticker.cpp:121]
UnrealEditor!FEngineLoop::Tick() [...\Launch\Private\LaunchEngineLoop.cpp:6073]
```

`AutoHandler_310_` is the `texture.create_normal_from_height` registration
(`REGISTER_TEXTURE_ACTION_HANDLER("texture.create_normal_from_height", "create_normal_from_height", ...)` at TextureHandler.cpp:2772).

## Repro

1. `texture.create_pattern_texture {name: "T_Panel_Height", path: "/Game/Textures/Panel", pattern: "Brick", width: 1024, height: 1024}` -> success, asset built `TFO_AutoDXT` (per editor log `LogTexture: Building textures: /Game/Textures/Panel/T_Panel_Height (TFO_AutoDXT, 1024x1024 ...)`).
2. `texture.create_normal_from_height {sourceTexture: "/Game/Textures/Panel/T_Panel_Height", name: "T_Panel_Normal", algorithm: "Sobel"}` -> connection reset (`[WinError 10054]`), editor process gone; every subsequent call -> `[WinError 10061]` connection refused. Canary `call()` after a 30s backoff still refused — editor did not recover.

Trigger context from the attempt (self_report): "Created and saved 5 source textures (base noise, brick height, AO/rough/metal); the editor crashed during the parallel create_normal_from_height + channel_pack calls and never recovered, so I could not build the normal map or ORM mask..." friction: "a single batch of two valid MCP calls (texture.create_normal_from_height and/or texture.channel_pack) crash-killed the editor (WinError 10054 connection-reset then 10061 refused on every retry); unrecoverable MCP-induced editor crash."

## Fix

Null-check the lock result before indexing it, and unlock cleanly on the error
path:

```cpp
FTexturePlatformData* PD = HeightMap->GetPlatformData();
if (!PD || PD->Mips.Num() == 0)
{
    TEXTURE_ERROR_RESPONSE(TEXT("Source height map has no platform mip data"));
}
FTexture2DMipMap& HeightMip = PD->Mips[0];
const uint8* HeightPixels = static_cast<const uint8*>(HeightMip.BulkData.LockReadOnly());
if (!HeightPixels)
{
    HeightMip.BulkData.Unlock();
    TEXTURE_ERROR_RESPONSE(TEXT("Could not read source height pixels (mip not CPU-resident; supply an uncompressed source or import from a height file)"));
}
```

A robust implementation should read from `Source` (the editor `FTextureSource`,
which is always uncompressed and CPU-resident) rather than the built platform mip,
and respect the actual source format instead of assuming 8-bit BGRA.

## History
- `#1-initial-repro` `OPEN` reporter — Editor crashed during the attempt's `texture.create_normal_from_height {sourceTexture:/Game/Textures/Panel/T_Panel_Height, name:T_Panel_Normal, algorithm:Sobel}` call (the crash IS the repro; no MCP replay possible — editor was down and a 30s-backoff canary `call()` stayed refused). Crash dump `UECC-Windows-BC8389E54D7D589AFF91F9A00F49EF66_0000` shows `EXCEPTION_ACCESS_VIOLATION reading address 0x0` at `ExecuteTextureAction` TextureHandler.cpp:725, dereferencing `HeightPixels` from an unchecked `HeightMip.BulkData.LockReadOnly()` (line 720) that returned null for the freshly-built `TFO_AutoDXT` source mip. Source confirmed verbatim under Plugins/PinWright/Source. No prior board ticket covers this crash (the only adjacent texture ticket, B-texture-create-placeholder-fake-success, is a different fake-success bug in create_texture_array/cube/volume).
- `#2-additional-gradient-source` `OPEN` reporter — Additional evidence (broader scope): reproduced again at HEAD with the source built via a DIFFERENT verb — `texture.create_gradient_texture {name: T_Rock_HeightGradient, Linear 90deg black->white, 1024x1024}` (prior repro used `create_pattern_texture`), then `texture.create_normal_from_height {src: T_Rock_HeightGradient, name: T_Rock_Normal, algorithm: Sobel, strength: 1.5}` → connection reset (`[WinError 10054]`) mid-call, editor gone, two confirm probes refused (`[WinError 10061]`); 20s+10s backoff canary `call()` stayed refused (editor did not recover). This confirms the crash is not specific to a compressed pattern mip — ANY freshly-built source (gradient too, `TFO_AutoDXT` per editor log `Building textures: /Game/Textures/StylizedRock/T_Rock_HeightGradient (TFO_AutoDXT, 1024x1024 ...)`) whose `LockReadOnly()` returns null triggers the same unchecked deref. Fresh crash dump `UECC-Windows-25B6507741A44A873776C89670D35E94_0000`: `Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x0000000000000000` at `ExecuteTextureAction` TextureHandler.cpp:725 → `RunTextureAction` :2693 → `AutoHandler_310_` :2766 (line numbers drifted slightly from the #1 dump but the crash site TextureHandler.cpp:725 is identical). Four prior creates in the same attempt (noise/gradient/pattern/radial-gradient) all succeeded; only create_normal_from_height crashed. Attempt friction: "texture.create_normal_from_height reproducibly took the editor down (WinError 10054 then 10061 on two confirm probes) right after four clean creates — a real robustness bug, not a discoverability issue."
- `#3-source-read-fix` `IN-REVIEW` developer — Root-cause fix: read the height pixels from the editor `FTextureSource` (always CPU-resident and uncompressed) instead of the built platform mip. In `Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureHandler.cpp` (`create_normal_from_height` branch) replaced the unchecked `GetPlatformData()->Mips[0].BulkData.LockReadOnly()` deref (former crash site) with: a `Source.IsValid()` + source-format guard (accepts `TSF_BGRA8`/`TSF_G8`, else a clean error instead of misreading compressed/float bytes), `Source`-derived Width/Height, `Source.LockMipReadOnly(0)` with a null-check, and per-format (BGRA8/G8) channel decode; unlock via `Source.UnlockMip(0)`. This removes every `GetPlatformData()` dereference from the branch, so the null/GPU-only platform mip that crashed the editor is no longer reachable, and the normal map is now built from the true source pixels. Regression test added: `PinWright.texture.create_normal_from_height.ReadsFromEditorSource` (`Plugins/PinWright/Source/PinWright/Private/Tests/Assets/TestCreateNormalFromHeightSourceRead.cpp`) — builds an in-code source whose editor `Source` carries a horizontal gradient while its platform mip is left resident-but-flat, drives the production handler via `InvokeHandlerWithCapture`, and asserts the output normal map's X (R) channel varies widely; reverting to the platform-mip read collapses the output to flat (spread 0) and fails the assertion without crashing the suite. Not compiled/tested here (later phase).
