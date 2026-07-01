---
id: B-set-aa-invalid-method-silent-noop
title: "post_process.set_anti_aliasing silently no-ops (success:true, CVar untouched) on an unrecognized `method` token instead of rejecting it"
status: OPEN
severity: Medium
category: bug
tags: [post-process, set_anti_aliasing, cvar, silent-noop, validation, invalid-enum]
encounters: 1
lastSeen: 2026-07-01T09:20:18.7649290+03:00
---

# `post_process.set_anti_aliasing` fake-succeeds on an unrecognized `method`

`post_process.set_anti_aliasing` documents `method` as exactly `None | FXAA |
TAA | MSAA | TSR` and writes the `r.AntiAliasingMethod` CVar. For a **valid**
token it works and echoes it back (`{"success":true,"antiAliasingMethod":"TSR"}`).
For an **unrecognized** token it returns a clean `{"success":true}` while
touching **nothing** — the CVar is never written, no error is raised, and the
`antiAliasingMethod` echo field is simply absent. A caller who typos or guesses
a plausible-but-unsupported AA token (`TAAU`, `TemporalAA`, `SMAA`, `DLAA`,
`FSR`) gets a green success and reasonably concludes the AA method changed, when
it silently did not. This is the same misleading-success defect class already
filed on this board for `gas.configure_asc`
(`E-configure-asc-echoes-invalid-replication-mode`),
`water.set_water_body_underwater_post_process`
(`E-water-underwater-settings-silent-drop`), and the GAS/AI tag-write family:
a write RPC that drops its input must not report unqualified success.

The absent `antiAliasingMethod` field is the only in-band signal, and it is
undocumented as a validation marker — every other outcome (valid method, or a
`screenPercentage`-only call) also returns `success:true`, so absence is not a
reliable "it was rejected" tell.

## Verbatim repro (live, replayed against mcp__pinwright__call)

1. `post_process.set_anti_aliasing {"method":"TAAU"}`
   -> `{"success":true}`  (no `antiAliasingMethod`; `r.AntiAliasingMethod` unchanged)
2. `post_process.set_anti_aliasing {"method":"banana"}`
   -> `{"success":true}`  (same silent no-op)
3. Contrast — valid token:
   `post_process.set_anti_aliasing {"method":"TSR","screenPercentage":80}`
   -> `{"success":true,"antiAliasingMethod":"TSR","screenPercentage":80}`  (applied)

## Source confirmation

`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/PostProcessHandler.cpp`,
`post_process.set_anti_aliasing` handler:

- L331: `Resp->SetBoolField(TEXT("success"), true);` — success is set **unconditionally**, up front, before any token is examined.
- L337: `int32 MethodValue = -1;` — the resolver default; an unrecognized token never reassigns it (L339-343 only match the five valid tokens).
- L345: `if (MethodValue >= 0)` — the CVar write **and** the `antiAliasingMethod` echo (L347-352) are gated behind this. There is **no `else`** — an unrecognized token falls straight through with `MethodValue == -1`, so the block is skipped and the handler emits its already-`success:true` response having done nothing.

```cpp
331     Resp->SetBoolField(TEXT("success"), true);
...
337         int32 MethodValue = -1;
...
343         else if (Method.Equals(TEXT("TSR"), ESearchCase::IgnoreCase))  { MethodValue = 4; ResolvedMethod = TEXT("TSR"); }
344
345         if (MethodValue >= 0)
346         {
347             if (IConsoleVariable* CVar =
348                 IConsoleManager::Get().FindConsoleVariable(TEXT("r.AntiAliasingMethod")))
349             {
350                 CVar->Set(MethodValue, ECVF_SetByCode);
351             }
352             Resp->SetStringField(TEXT("antiAliasingMethod"), ResolvedMethod);
353         }   // no else: unrecognized token → silent no-op, success already true
```

## What it should do

Adopt the house validate-before-mutate / fail-loud convention already landed for
this exact class (`E-configure-asc-echoes-invalid-replication-mode` fix option
(a), the GAS/AI tag-write family): when `method` is present but resolves to
`MethodValue < 0`, reject with `SendError("INVALID_PARAMS", "Unrecognized
anti-aliasing method '<x>'; must be one of None|FXAA|TAA|MSAA|TSR.")` instead of
returning a clean success that changed nothing. (`method` is optional, so a call
with no `method` at all — e.g. `screenPercentage`-only — must still succeed.)

severity rationale: impact=silent false-success (High) x reach=rare (invalid-token edge path on a non-every-session setter) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode replay of `post_process.set_anti_aliasing` (finalize-the-look task). Confirmed live against `mcp__pinwright__call`: `{"method":"TAAU"}` and `{"method":"banana"}` both return `{"success":true}` with no `antiAliasingMethod` echo and `r.AntiAliasingMethod` untouched, while the valid `{"method":"TSR","screenPercentage":80}` applies and echoes. Source (`PostProcessHandler.cpp` L331 unconditional `success:true`, L337 `MethodValue=-1` default, L345 `if (MethodValue >= 0)` gate with no `else`) confirms an unrecognized token silently skips the CVar write. Same misleading-success class as `E-configure-asc-echoes-invalid-replication-mode` / `E-water-underwater-settings-silent-drop`; proposes the same validate-before-mutate `INVALID_PARAMS` rejection naming the allowed tokens (keeping the `method`-omitted path a success).
