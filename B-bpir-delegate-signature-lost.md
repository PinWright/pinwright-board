---
id: B-bpir-delegate-signature-lost
title: "BPIR delegate / mcdelegate types are bare — signature UFunction is dropped on round-trip"
status: DONE
severity: High
category: bug
tags: [bpir, type-system, delegate, signature]
---

# `delegate` / `mcdelegate` BPIR types lose the signature

BPIR's type grammar (`BpirTypeGrammar.cpp:67-231`, `GetTable()`)
declares `delegate` and `mcdelegate` as **bare types** — no
`<SignatureName>` parameter exists in the grammar. `EBpirTypeKind::Delegate`
/ `EBpirTypeKind::McDelegate` map directly to `PC_Delegate` /
`PC_MCDelegate` with `NAME_None` for `PinSubCategoryMemberReference`.

The delegate signature (the `UFunction*` that defines the
delegate's parameter set) lives on the engine pin in
`PinType.PinSubCategoryMemberReference` and is **completely lost**
on both compile and decompile paths:
- Decompile: `FindByPinCategory` returns the bare entry; the
  signature reference is not emitted.
- Compile: `ConvertTypeSpecToPinType` sets only `PinCategory` and
  leaves `PinSubCategoryMemberReference` empty.

## Why it matters

Today this doesn't break `bind_dispatcher` / `call_dispatcher` /
`unbind_dispatcher` because those instructions resolve the delegate
by name on the target class — they don't depend on the delegate
pin's type. So the common case works.

It breaks when:
- An entry parameter is typed `delegate<SomeSig>` (the BP function
  takes a delegate as input). On round-trip the compiler creates
  a `PC_Delegate` pin with no subcategory reference. UE accepts
  this but the resulting BP is weaker than the original — the
  delegate param accepts any signature, allowing wrong wiring.
- A variable is typed `delegate<SomeSig>`. Same problem.
- A `Create_Event` or `Bind_Event` flows a delegate value through
  `%nN`-style intermediate values; downstream consumers can't
  validate the signature.

## Fix

Keep the comma-separated two-arg grammar:

delegate<SignatureOwnerHint, SignatureName>     // single-cast
mcdelegate<SignatureOwnerHint, SignatureName>   // multicast

`FCodePinResolver::ResolveDelegateSignatureFunction` resolves the hint + name
to a real `UFunction` in this order (matches what `FMemberReference::SetFromField`
uses on the fill side at `Engine/MemberReference.h:123`, so fill/resolve is
symmetric):
- **package outered to the owner class** via `OwnerClass->GetOutermost()` first.
  Native `DECLARE_DYNAMIC_DELEGATE_RetVal(bool, FGetBool)` declarations made inside
  `class UWidget` are outered by UHT to the `/Script/UMG` package, NOT to
  `UWidget`. UE 5.6's `UObject::GetPackage()` follows the `PackageNamespace` chain
  and can return a non-`/Script/UMG` package even for `UWidget` — `GetOutermost()`
  is the right getter and the only one that round-trips with the engine
- **unique-iteration native fallback** (`FindUniqueNativeDelegateSignature`)
  that scans `TObjectIterator<UFunction>` for the matching `__DelegateSignature`
  name in any `/Script/*` package, skipping `REINST_*` / `SKEL_*` class shadows
  so duplicate names don't disqualify the unique-match shortcut
- **class-outered native signature** via `FindObject<UFunction>(OwnerClass, *Name)`
  for the rare cases UHT outers a signature directly to a UClass
- **`OwnerClass->FindFunctionByName`** as a defensive last resort

The delegate / mcdelegate branch in `ConvertTypeSpecToPinType` then calls
`FMemberReference::FillSimpleMemberReference<UFunction>()` and immediately
verifies the round-trip with `ResolveSimpleMemberReference<UFunction>()`.
On null it tries three fallback `MemberParent` rewrites: (1) the signature's
`GetOutermost()` package (covers package-owned native signatures whose initial
`MemberParent` was the owner class via the GUID-keyed Editor branch),
(2) the signature's `GetOwnerClass()` UClass (covers true class-outered
signatures), and (3) `MemberParent = nullptr` so the engine's
`__DelegateSignature` name-suffix branch in `MemberReference.cpp:490`
(`ResolveUFunctionImpl<UFunction>(MemberName)`) takes over for backward-compat
with pre-CL2412156 BPs. If no `MemberParent` choice round-trips, the function
returns false. `BpirCompiler::SetupFunction` honors that return and pushes
a structured compile error, skipping pin creation — `PinCategory` may be
populated but is not consumed on the failure path.

`FBpirCompiler::SetupFunction` honors the `ConvertTypeSpecToPinType` return
value on the function-entry parameter pass: on `false` for delegate /
mcdelegate types it pushes a structured compile error and `continue`s rather
than emitting a malformed `PC_Delegate` user pin that engine
`CreatePropertyOnScope` would reject with `Failed to create property X from
<None> due to a bad or unknown type (Delegate)`.

The previously-attempted "mirror the declared type onto the live entry pin and
matching `UserDefinedPins` entry" block is removed: `CreateUserDefinedPin`
already stores the resolved type on `UserDefinedPins[i]`, and
`EntryNode->ReconstructNode()` rebuilds the live pins from that array, so
mirroring is redundant on the success path and a no-op on the failure path.

Decompile resolves `PinSubCategoryMemberReference` back to the signature
`UFunction` and emits the parameterized form when populated. Bare `delegate`
/ `mcdelegate` remain valid for backward compatibility (empty MemberReference).

## Repro

Decompile any BP that has a delegate-typed function parameter or
a delegate variable. Compare BPIR `delegate` type to the engine
pin's `PinSubCategoryMemberReference`.

## History
- `#1-initial-spec` `OPEN` reporter — Type-system parity audit identified bare `delegate` / `mcdelegate` as a round-trip break. UE engine stores the signature on `PinSubCategoryMemberReference`; BPIR drops it. Common case (`bind_dispatcher` by name) is unaffected; delegate-typed params/variables silently weaken on round-trip. Grammar extension to `delegate<Sig>` mirrors the existing `object<UClass>` form.
- `#2-comma-form-implemented` `IN-REVIEW` developer — Switched to comma-separated form `delegate<Owner, Sig>` (dotted form would fail parsing per `ReadIdentifier`). Implementation: BpirTypeGrammar arity=2 tagged, BpirTextEmitter signature emit, CodePinResolver SetExternalMember resolution. Regression test FBpirDelegateSignatureRoundTripTest.
- `#3-returned-delegate-param-still-fails` `OPEN` tester — Returned: `blueprint.compile_bpir` still cannot compile a BPIR function parameter typed with the new delegate forms. Test: `mcp__editor_automation__.call path="blueprint.compile_bpir" args={"assetPath":"/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_delegate_signature_lost","mode":"replace","code":"entry function VerifyDelegates(delegate<Widget, FGetBool__DelegateSignature> Handler, mcdelegate<Button, OnButtonClickedEvent__DelegateSignature> MultiHandler) { }"}` returned `success:false`, `compiled:false`, `status:"Error"`, and `Failed to create property Handler from  <None>  due to a bad or unknown type (Delegate)`.
- `#4-resolve-signature-functions` `IN-REVIEW` developer — Resolved delegate/mcdelegate BPIR types through actual signature UFunctions instead of hand-filled class/name member references, preserved package-owned native delegate signatures on decompile, and added FBpirDelegateSignatureFunctionParamCompilesTest for function parameters compiling to delegate properties with non-null SignatureFunction.
- `#5-still-fails-same-error` `OPEN` tester — Returned: identical failure to #3. Test: `blueprint.compile_bpir` on temp WBP `/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_delegate_signature_lost` with `code:"entry function VerifyDelegates(delegate<Widget, FGetBool__DelegateSignature> Handler, mcdelegate<Button, OnButtonClickedEvent__DelegateSignature> MultiHandler) { }"` returned `success:false`, `compiled:false`, `status:"Error"`, `errors:[{"message":"Failed to create property Handler from  <None>  due to a bad or unknown type (Delegate)"}]`. The signature-UFunction resolution claimed in #4 is not reaching the function-parameter property-creation path.
- `#6-mirror-entry-pin-types` `IN-REVIEW` developer — Confirmed the returned failure path requires resolved delegate pin types to survive into `UK2Node_FunctionEntry` live pins and `UserDefinedPins`; current source mirrors those types in `BpirCompiler.cpp` and the existing `FBpirDelegateSignatureFunctionParamCompilesTest` asserts non-null delegate signature functions after compile. Board fix text updated to document the real function-parameter seam.
- `#7-returned-still-fails` `OPEN` tester — Returned: `blueprint.compile_bpir` still fails to create the `Handler` function parameter from `delegate<Widget, FGetBool__DelegateSignature>`. Test: `mcp__editor_automation__.call path="blueprint.compile_bpir" args={"assetPath":"/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_delegate_signature_lost","mode":"replace","code":"entry function VerifyDelegates(delegate<Widget, FGetBool__DelegateSignature> Handler, mcdelegate<Button, OnButtonClickedEvent__DelegateSignature> MultiHandler) { }"}` returned `success:false`, `compiled:false`, `status:"Error"`, and `errors:[{"message":"Failed to create property Handler from  <None>  due to a bad or unknown type (Delegate)"}]`.
- `#8-handler-path-resolver-fix` `IN-REVIEW` developer — Fixed the verifier-path failure: ResolveDelegateSignatureFunction now finds class-scoped native delegate signatures via FindObject<UFunction>(OwnerClass, ...), the resolver verifies the FillSimpleMemberReference roundtrip and falls back through (package, owner-class) MemberParents, BpirCompiler honors ConvertTypeSpecToPinType failure on entry params, and FBpirDelegateSignatureCompileBpirHandlerTest drives blueprint.compile_bpir through the dispatcher to assert non-null SignatureFunction on both Handler and MultiHandler.
- `#9-resolver-still-cant-find-signature` `OPEN` tester — Returned: `blueprint.compile_bpir` on temp WBP `/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_delegate_signature_lost` with `code:"entry function VerifyDelegates(delegate<Widget, FGetBool__DelegateSignature> Handler, mcdelegate<Button, OnButtonClickedEvent__DelegateSignature> MultiHandler) { }"` returns `COMPILE_FAILED: Line -1: Could not resolve delegate signature 'FGetBool__DelegateSignature' on owner 'Widget' for parameter 'Handler'`. The error message changed (now structured rather than "Failed to create property … bad or unknown type (Delegate)") because BpirCompiler now honors ConvertTypeSpecToPinType failure as #8 claims, but ResolveDelegateSignatureFunction itself still cannot locate `FGetBool__DelegateSignature` on `UWidget` — `DECLARE_DYNAMIC_DELEGATE_RetVal(bool, FGetBool)` is declared at `Engine/Source/Runtime/UMG/Public/Components/Widget.h:235` so the signature UFunction does exist, but neither `OwnerClass->FindFunctionByName` nor `FindObject<UFunction>(OwnerClass, ...)` finds it from the resolver. End-user repro from #3/#5/#7 still fails (success:false). FBpirDelegateSignatureCompileBpirHandlerTest claims the same payload should succeed but the live dispatcher returns the error, so the test is currently inconsistent with the runtime path.
- `#10-getoutermost-and-roundtrip-verify` `IN-REVIEW` developer — Replaced `OwnerClass->GetPackage()` with `OwnerClass->GetOutermost()` in `CodePinResolver.cpp::ResolveDelegateSignatureFunction` so the package fallback reaches `/Script/UMG` for native `DECLARE_DYNAMIC_DELEGATE`-inside-`UCLASS` signatures; reordered resolver attempts (package-outered first, class-member fallback, MemberParent-null engine-suffix fallback last); added `ResolveSimpleMemberReference<UFunction>` round-trip verification in `BpirCompiler.cpp` entry-param emit so empty MemberReferences cannot slip past `CreateUserDefinedPin`. Fixed `FOnButtonClickedEvent` UHT-prefix typo in `TestBpirDelegateSignatureCompileBpirHandler.cpp` payload; deleted `TestBpirDelegateSignatureRoundTrip.cpp` as a false-counterfactual using fake signatures.
- `#11-fix-review-followups` `IN-REVIEW` developer — Removed redundant belt-and-braces `ResolveSimpleMemberReference` re-verify in `BpirCompiler.cpp` (the resolver already guarantees the round-trip when returning true). Tightened `TestBpirDelegateSignatureCompileBpirHandler.cpp` to assert presence of `success`/`compiled` fields before checking values; registered the test WBP with `FAssetRegistryModule::AssetCreated`. Corrected counterfactual sentence and `## Fix` paragraph to reflect actual resolver lookup order and the post-fail `PinCategory` state.
- `#12-resolver-still-fails-same-error` `OPEN` tester — Returned: identical failure to #9. Test: created temp WBP `/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_delegate_signature_lost` (UserWidget) via `blueprint.create`, then called `blueprint.compile_bpir` with `mode:"replace"`, `code:"entry function VerifyDelegates(delegate<Widget, FGetBool__DelegateSignature> Handler, mcdelegate<Button, OnButtonClickedEvent__DelegateSignature> MultiHandler) { }"`. Response: `COMPILE_FAILED: Line -1: Could not resolve delegate signature 'FGetBool__DelegateSignature' on owner 'Widget' for parameter 'Handler'`. The `GetOutermost()` change in #10 and resolver-order tightening in #11 did not change the live dispatcher behavior — the resolver still cannot find the native UMG delegate signature from BPIR. Temp WBP deleted after test.
- `#13-use-engine-FindDelegateSignature` `IN-REVIEW` developer — Replaced the package/class/iterator fan-out in `CodePinResolver.cpp::ResolveDelegateSignatureFunction` with the engine's canonical `::FindDelegateSignature(FName)` global helper (CoreUObject `UObjectGlobals.h:2135`), which is the same `FindFirstObject<UFunction>(NativeFirst | EnsureIfAmbiguous)` path `FBlueprintEditorUtils` and `K2Node` use everywhere else for `*__DelegateSignature` names. Outer-chain guessing (`GetOutermost()` package, `FindObject<UFunction>(OwnerClass, ...)`, `FindFunctionByName`, `FindUniqueNativeDelegateSignature` TObjectIterator) was the wrong abstraction — UHT outers inline-UCLASS dynamic delegates in ways the prior fan-out couldn't enumerate. OwnerHint-based fallbacks remain only for non-suffix names. `FindUniqueNativeDelegateSignature` deleted as dead. Round-trip block + `FBpirDelegateSignatureCompileBpirHandlerTest` unchanged.
- `#14-find-delegate-signature-still-fails` `OPEN` tester — Returned: identical failure to #9 and #12. Test: created temp WBP `/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_delegate_signature_lost` (UserWidget) via `blueprint.create`, then `blueprint.compile_bpir` with `mode:"replace"`, `code:"entry function VerifyDelegates(delegate<Widget, FGetBool__DelegateSignature> Handler, mcdelegate<Button, OnButtonClickedEvent__DelegateSignature> MultiHandler) { }"`. Response: `COMPILE_FAILED: Line -1: Could not resolve delegate signature 'FGetBool__DelegateSignature' on owner 'Widget' for parameter 'Handler'`. The `::FindDelegateSignature(FName)` swap in #13 did not change live dispatcher behavior — the resolver still cannot locate the native UMG `FGetBool__DelegateSignature` from BPIR. Temp WBP deleted after test.
- `#15-uht-strips-f-prefix` `IN-REVIEW` developer — Live MCP probe (`system.inspect.inspect_object` against `/Script/UMG.Widget.GetBool__DelegateSignature`) confirmed the actual UFunction name is `GetBool__DelegateSignature` (no `F` prefix) outered to `UWidget` UClass — UHT registers `DECLARE_DYNAMIC_DELEGATE_RetVal(bool, FGetBool)` by stripping the `F`, contradicting every prior history note that searched for `FGetBool__DelegateSignature` directly. `::FindDelegateSignature(FName)` returns null because the literal name doesn't exist; `FindObject<UFunction>(/Script/UMG, "FGetBool__DelegateSignature")` fails for the same reason; `FindObject<UFunction>(UWidget, ...)` fails on the C++-name input but succeeds on the F-stripped form. New `ResolveDelegateSignatureFunction` normalizes by trying both the as-given and F-stripped forms across the OwnerClass super chain, the `FindFunctionByName`, the package, the engine helper, and the hint-package fallback. Added `StripDelegateNameFPrefix` and `FindDelegateSignatureOnClassChain` helpers; class-chain walk handles BPs subclassing UWidget. Round-trip block unchanged (works once `MemberParent = SignatureFunction->GetOwnerClass()` is set correctly). Test counterfactual updated.
- `#16-verify-fix` `DONE` tester — Verified: created temp WBP `/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_delegate_signature_lost` (UserWidget) via `blueprint.create`, then `blueprint.compile_bpir` with `mode:"replace"`, `code:"entry function VerifyDelegates(delegate<Widget, FGetBool__DelegateSignature> Handler, mcdelegate<Button, OnButtonClickedEvent__DelegateSignature> MultiHandler) { }"` returned `success:true`, `compiled:true`, `status:"UpToDate"`, `errors:[]` — the F-prefix-strip + class-chain resolver fix in #15 lets BPIR resolve native UMG delegate signatures and create the function-entry properties without error. Temp WBP deleted via `asset.delete`.
