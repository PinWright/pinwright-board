---
id: B-bpir-direct-delegate-outputdelegate-loss
title: "BPIR decompiler emits $OutputDelegate for direct CustomEvent delegate bindings instead of event:@Handler"
status: DONE
severity: High
category: bug
tags: [bpir, decompiler, delegates, events]
---

# BPIR decompiler emits $OutputDelegate for direct CustomEvent delegate bindings instead of event:@Handler

Direct CustomEvent delegate bindings can decompile as generic `$OutputDelegate`, losing the handler identity. The resulting BPIR is ambiguous and does not show which event is bound.

**Workaround:** Inspect live graph pins for the delegate link before using the BPIR as source of truth.

**Fix:** Resolve direct `K2Node_CustomEvent.OutputDelegate` links, including reroute hops, and emit `event:@HandlerName` or another explicit handler reference.

## History
- `#1-fresh-app-game-dump-audit` `OPEN` reporter — Fresh dumps contain 263 `$OutputDelegate` references across 103 BPIR files. Live `/App/App/UI/W_RateSettings` shows AddDelegate/AssignDelegate `Delegate` pins linked directly, through reroutes in one case, to `K2Node_CustomEvent OnValueChanged_Event_0.OutputDelegate`. `blueprint.decompile` emits `bind_dispatcher OnValueChanged(..., Delegate: $OutputDelegate)` instead of `event: @OnValueChanged_Event_0`, losing handler identity and making copied BPIR ambiguous. Existing bind-dispatcher tickets cover CreateDelegate/event keyword compile paths, not direct CustomEvent `OutputDelegate` decompile.
- `#2-direct-customevent-decompile` `IN-REVIEW` developer — Moved `FollowKnotsBackward` out of the anonymous `BpirDecompilerInternal` namespace in `BpirDecompiler.cpp` (lines 69–110) into the named `BpirDecompiler::Helpers` namespace; added the forward declaration to `DecompilerTypes.h`; updated both call sites at BpirDecompiler.cpp:~1580 and :~2000 to use the qualified name. In `BpirTextEmitter.cpp` `EmitDispatcherNode`, restructured the EventArg recovery block (lines ~1226–1259): moved EventArg computation before `FormatArgs`, added knot traversal (`FollowKnotsBackward`) on each linked source before casting, extended the cast chain to match `UK2Node_CustomEvent` (reading `GetFunctionName()` → `CustomFunctionName`) outside the `MCP_HAS_CREATE_DELEGATE` guard so it compiles on all configurations, and added a post-filter that strips `Delegate: ...` from `Args` when EventArg is non-empty. Also added `#include "Decompiler/DecompilerTypes.h"` to `BpirTextEmitter.cpp`. Regression test added: `FBpirDirectOutputDelegateDecompilesAsEventAt` in `TestBpirDirectOutputDelegateDecompilesAsEventAt.cpp` — if the EventArg recovery extension at BpirTextEmitter.cpp:~1226–1259 is reverted to match only UK2Node_CreateDelegate, the CustomEvent's OutputDelegate pin resolves through ResolveInputValue's entry-param branch returning `$OutputDelegate`, the EventArg recovery yields empty, and the assertion Contains(`event: @ProbeHandler`) fails because the output contains only Delegate: $OutputDelegate.
- `#3-verify-fix` `DONE` tester — Verified: ran `blueprint.decompile` on `/App/App/UI/W_RateSettings`. Output contains `bind_dispatcher OnValueChanged(Target: $Acro, event: @OnValueChanged_Event_0)` and `bind_dispatcher OnValueChanged(Target: $Angle, event: @OnValueChanged_Event_0)`; no `$OutputDelegate` or `Delegate:` substrings present. Matches the expected `event:@HandlerName` form per the Fix description.
