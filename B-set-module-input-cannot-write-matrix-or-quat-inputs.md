---
id: B-set-module-input-cannot-write-matrix-or-quat-inputs
title: "niagara.set_module_input silently truncates a NiagaraMatrix to its first 4 floats and reports success — and that corrupting write is the ONLY way to create the override pin the working string form needs"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, set-module-input, matrix, quat, silent-truncation, data-loss, success-no-effect, shape-location, bootstrap, unsupported-input-value]
encounters: 1
lastSeen: 2026-09-05
---

# A 64-byte `NiagaraMatrix` input takes a 16-number array, keeps 4 of the 16, and answers `success: true`

`niagara.set_module_input` has no `NiagaraMatrix` (`/Script/Niagara.NiagaraMatrix`, 64 bytes,
four `Vector4f` rows) case. Three of the four spellings a caller would try are refused; the
fourth is **accepted and silently truncated to a `Vector4f`**, and it is the only one that
creates the override pin — so the one spelling that writes a whole matrix is unreachable
without first corrupting the pin.

This blocks `ShapeLocation`'s `Transform Method = Custom Matrix`, the engine's only sanctioned
way to decouple a spawn shape from the owner transform.

## Measured, live editor port 27145, UE 5.8, EAContentExamples58, 2026-09-05

Scratch emitter `/Game/FPS/VFX/Scratch_TicketRetry/E_TR_Curve` (an `asset.duplicate` of
`SimpleSpriteBurst`) + `niagara.add_module` `/Niagara/Modules/Spawn/Location/V2/ShapeLocation`
into `ParticleSpawnScript`, entryId `82FB05B84206E5D55B847184A792A3D4`, then
`niagara.set_static_switch {inputName:"Transform Method", value:"Custom Matrix"}` ->
`{"value":"NewEnumerator1","index":1,"displayName":"Custom Matrix"}`.

`niagara.inspect {includeStack:true}` reports the module's three affected inputs as
`Custom Transform Matrix` (`NiagaraMatrix`), `Rotation Matrix` (`NiagaraMatrix`) and
`Rotation Quaternion` (`Quat4f`), all `valueMode: "default"`.

| `value` sent to `Custom Transform Matrix` | result |
|---|---|
| `{Row0:{x,y,z,w}, Row1:..., Row2:..., Row3:...}` | `[UNSUPPORTED_INPUT_VALUE] Module input values must be a bool, number, vector object/array, color object, or an existing override pin default string.` |
| 16 comma-separated floats as one string | same `UNSUPPORTED_INPUT_VALUE` |
| the UE struct literal `(Row0=(X=1.0,...),Row1=...,Row2=...,Row3=...)` **on a pin with no override yet** | same `UNSUPPORTED_INPUT_VALUE` — proven separately on the untouched `Rotation Matrix` input |
| `[1,0,0,0, 0,1,0,0, 0,0,1,0, 0,0,0,1]` (16-number JSON array) | **`success: true`**, `"value":"(X=1.0,Y=0.0,Z=0.0,W=0.0)"` — **12 of the 16 components discarded, no warning** |
| the UE struct literal again, now that the pin HAS an override | `success: true`, `"value":"(Row0=(X=1.0,Y=0.0,Z=0.0,W=0.0),Row1=(X=0.0,Y=1.0,Z=0.0,W=0.0),Row2=(X=0.0,Y=0.0,Z=1.0,W=0.0),Row3=(X=0.0,Y=0.0,Z=0.0,W=1.0))"` — the full matrix |

**The truncation is on disk, not just in the echo.** A distinct probe on the second matrix input:
`niagara.set_module_input {entryId:"82FB05B84206E5D55B847184A792A3D4", inputName:"Rotation Matrix", value:[9,8,7,6,5,4,3,2,1,0,0,0,0,0,0,1]}`
-> `success:true`, `"value":"(X=9.0,Y=8.0,Z=7.0,W=6.0)"`.
`asset.save {force:true}` -> `saveState:"written"`, `sizeBytes:108059`;
`Content/FPS/VFX/Scratch_TicketRetry/E_TR_Curve.uasset` mtime `2026-09-05 20:55:47 +0300`.
`grep -a` on that file finds `(X=9.0,Y=8.0,Z=7.0,W=6.0)` and finds **no** `Y=4.0` and **no**
`X=5.0` anywhere in the package — the other twelve floats were never written.

## Why it matters

A refusal costs the caller a workaround. A `success: true` that keeps a quarter of the value
costs them the bug hunt, because every documented verification passes: the response echoes a
well-formed `Vector4f` literal, `niagara.compile` completes, and the emitter renders — from a
matrix whose bottom three rows are zero, which is a degenerate transform that collapses every
spawn position onto one axis. This is the `set_module_input` echo contract failing in the one
place its wiki page says to trust it: *"Confirm a literal edit from this result directly — no
second inspect needed."*

The bootstrap deadlock makes it worse rather than better. The struct-literal form **does** write
a correct 16-component matrix, so the capability exists — but it is gated on
*"an existing override pin default string"*, and the only way to get an override pin onto a
matrix input is the truncating array write. The sole route to a correct matrix runs through a
call that corrupts it first.

## Correction to the premise this ticket was filed under

**`Quat4f` is NOT affected.** The reporting brief guessed that `Rotation Quaternion` would fail
the same way. It does not:
`niagara.set_module_input {entryId:"82FB05B84206E5D55B847184A792A3D4", inputName:"Rotation Quaternion", value:{x:0,y:0,z:0.3826834,w:0.9238795}}`
-> `success:true`, `"value":"(X=0.0,Y=0.0,Z=0.382683,W=0.923879)"`, and `grep -a` finds that exact
string in the saved `.uasset`. A `Quat4f` is one `Vector4f` and the existing vector-object path
covers it. Only the 4-row `NiagaraMatrix` is broken. Do not widen the fix on the strength of the
original guess.

## Expected

1. A 16-number array (or a `{Row0..Row3}` object) on a `NiagaraMatrix` input writes all sixteen
   components, or is **refused**. Anything that keeps 4 of 16 and answers `success` is worse than
   no support at all. If the array path stays shared with vectors, it must reject an array whose
   length does not match the declared input's component count instead of taking a prefix.
2. The struct-literal string form works on an input with no override yet, so there is a bootstrap.
   Its current gate is what makes the correct route unreachable from a clean asset.
3. `UNSUPPORTED_INPUT_VALUE`'s message should name the input's declared type and the form that
   type takes, the way `MODULE_INPUT_NOT_FOUND` now lists a module's inputs. As written it lists
   the shapes the handler accepts and never says which one this input wants.

## Cross-ref

- `B-niagara-module-input-dotted-subinput-silent-noop` — the same verb's other
  affirmative-success-for-a-no-op, fixed by refusing. The same remedy applies here.
- `B-niagara-set-curve-keys-unreachable-module-input-di` `#10`/`#11` — the module-input
  addressing this verb shares; that fix creates an override pin for a default-valued input, which
  is the shape the matrix path needs for its bootstrap.

severity rationale: impact=silent data loss on a write verb whose response is documented as
authoritative, producing a degenerate transform that still compiles and renders x reach=every
`NiagaraMatrix` input, including `ShapeLocation`'s `Custom Matrix` mode, the engine's only
sanctioned shape/owner decoupling -> High

## Fix

Verdict: **PARTLY TRUE**. The Matrix truncation is source-confirmed: a fresh 16-number array
was converted by `JsonValueToPinDefaultString` into a four-component vector literal before the
override pin was created. The Quat values were preserved as four floats, but the fresh override
was also inferred as `Vec4`, and no quaternion normalization was applied.

`Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp` now validates fresh declared
`NiagaraMatrix` values as exactly 16 finite float-representable numbers in row-major order and
fresh declared `Quat4f` values as exactly four finite components, rejects a zero quaternion, and
normalizes an `FQuat4f`. It asks `UEdGraphSchema_Niagara::TryGetPinDefaultValueFromNiagaraVariable`
for the engine's pin-default encoding. UE 5.8's Matrix editor utility has no pin-default encoder,
so Matrix literals now refuse with `INVALID_INPUT_VALUE` before graph mutation; the fix makes no
claim that Matrix writes succeed on UE 5.8. For Quat, the handler writes the schema-encoded value,
decodes the actual override with `UEdGraphSchema_Niagara::PinToNiagaraVariable`, compares every
component, and echoes the decoded canonical string. Raw text equality is retained only for
untyped string paths.

Files changed:

- `Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp`
- `Source/PinWright/Private/Tests/Niagara/TestNiagaraSetModuleInputMatrixQuat.cpp`
- `Plugins/PinWright/Docs/wiki-src/niagara.md`

Regression tests:

- `PinWright.niagara.set_module_input.RefusesUnsupportedMatrixLiteral`
- `PinWright.niagara.set_module_input.WritesNormalizedQuatLiteral`
- `PinWright.niagara.set_module_input.RefusesWrongMatrixArity`

Deliberately not changed: no separate parser/bootstrap for a fresh UE struct-literal string or a
`{Row0..Row3}` Matrix object was added. On UE 5.8 the exact 16-number Matrix path is intentionally
unsupported and refuses before graph mutation; Quat remains supported through schema encode and
decoded component readback. No Unreal build, editor run, or automation execution was performed
under this source-only brief.

## History
- `#1-initial-repro` `OPEN` VFX — Filed while retrying six workaround tickets against the rebuilt plugin (PLAN rule 2) on the FPS VFX stream, EAContentExamples58, UE 5.8, live editor port 27145, 2026-09-05. Every call and every response above was executed and is quoted verbatim; the on-disk evidence is a `grep -a` over the saved `.uasset` at the mtime given. Scratch asset `/Game/FPS/VFX/Scratch_TicketRetry/E_TR_Curve` is deleted after the run, so the repro must be rebuilt from the `asset.duplicate` + `add_module` + `set_static_switch` steps above. Not source-confirmed: no read of `NiagaraEditHandler.cpp`'s value-parsing path was made; the "array path takes a prefix" reading is inferred from the echoed `Vector4f` literal plus the absent components on disk, not from the code. The brief this ticket was filed under asserted the object form and the 16-float string both fail — both confirmed — and predicted `Quat4f` fails too, which is **wrong** and is corrected above.
- `#2-typed-matrix-quat-write` `IN-REVIEW` developer — Verdict PARTLY TRUE. Changed `NiagaraEditHandler.cpp` to use declared Matrix/Quat types for fresh numeric literals, enforce finite exact arity, normalize non-zero `FQuat4f` values, verify override-pin type/default readback, and echo the read-back string; added `TestNiagaraSetModuleInputMatrixQuat.cpp` coverage for the UE 5.8 Matrix refusal, normalized non-identity Quat, and wrong Matrix arity. Deliberately left fresh struct-literal bootstrap and Matrix row-object parsing unchanged; no build, editor, or automation run was performed.
- `#3-schema-typed-readback-correction` `IN-REVIEW` developer — Corrected the rejected proof: Quat values are encoded through `UEdGraphSchema_Niagara::TryGetPinDefaultValueFromNiagaraVariable` and decoded through `PinToNiagaraVariable`, with component-wise readback verification; UE 5.8 Matrix values refuse pre-mutation because its type utility lacks pin-default encoding/decoding. Tests no longer derive expected text with `ToString`; the Quat fixture is non-identity so an identity decoder cannot pass. Added `Plugins/PinWright/Docs/wiki-src/niagara.md` to the changed-files list.
