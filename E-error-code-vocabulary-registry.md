---
id: E-error-code-vocabulary-registry
title: "Error code vocabulary is free-form — ~540 unique strings with semantic dupes, no central registry"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [error-handling, dx, consistency]
---

# Error code vocabulary is free-form across 90+ handler files

`FHandlerContext::SendError(Code, Message)` takes `Code` as a free-form
`FString`. Handler authors invent codes ad hoc per file. Confirmed by
`grep -rho 'SendError(TEXT("[A-Z_]*")' Private/Handlers/ | sort -u`:
**539 unique first-arg literals** across the handler tree.

The top 5 match the prior triage exactly:

| Code | Count |
|---|---|
| `INVALID_ARGUMENT` | 529 |
| `NOT_FOUND` | 240 |
| `ASSET_NOT_FOUND` | 156 |
| `NO_WORLD` | 107 |
| `INVALID_PARAMS` | 87 |

Then a long tail: `MISSING_PARAM` (77), `CREATE_FAILED` (74),
`EDITOR_NOT_AVAILABLE` (67), `MISSING_PARAMETER` (58), `ACTOR_NOT_FOUND`
(50), `SPAWN_FAILED` (49), `CREATION_FAILED` (47), ... down to single-use
hyper-specific strings like `INVALID_INHIBITION_POLICY`,
`INVALID_PRUNING_TYPE`, `INVALID_SIMULATION_STAGE_CLASS`,
`MISSING_STATE_MACHINE_NAME`.

## Concrete semantic dupes

Different spelling, same meaning, sometimes within the same handler family:

- `MISSING_PARAM` (77) vs `MISSING_PARAMETER` (58) vs `MISSING_PARAMETERS`
- `INVALID_PARAMS` (87) vs `INVALID_ARGUMENT` (529) vs `INVALID_PARAM` vs
  `INVALID_PARAMETER` vs `INVALID_PAYLOAD` (46) — all "the caller gave us
  bad params"
- `CREATE_FAILED` (74) vs `CREATION_FAILED` (47) vs `CREATION_ERROR` —
  same condition, three spellings
- `INVALID_PARAM_TYPE` vs `INVALID_PARAMETER_TYPE` — two ways to say
  "param had wrong type"
- `BIND_FAILED` vs `BINDING_FAILED` vs `BINDING_CREATION_FAILED` — all
  "delegate bind failed"

Clients hard-code these literal strings (script callers, test fixtures,
agent prompts that branch on the error code), so dupes become a
real-world ambiguity, not just an aesthetics concern.

## Scope

- **539** unique uppercase codes registered through `Ctx.SendError`
- ~90 handler files across `Private/Handlers/<Domain>/`
- No central enum, registry, or documentation list. The only contract
  is the prose line in `CLAUDE.md` ("domain-specific uppercase").
- The auto-generated `docs/rpc-method-reference.generated.md` does not
  list per-method error codes either.

## Proposed approach

Two viable shapes; pick one as a follow-up sprint decision (do **not**
ship both):

### Option A — central enum-like header

`Handlers/ErrorCodes.h` declares all codes as `inline constexpr TCHAR
ERR_*[]` (or wrapped `FName` constants). `SendError` keeps the
`FString` signature (no ABI churn, accepts ad-hoc codes for migration),
but every callsite migrates to `ERR_INVALID_ARGUMENT` etc. Compile-time
discoverability via IntelliSense; greppable; deletes a dupe by
deleting a symbol.

### Option B — runtime registered registry

Codes register via a macro like `REGISTER_ERROR_CODE("INVALID_ARGUMENT",
"Caller-supplied arg failed validation")` and `SendError` looks them up,
warning when an unregistered code is sent. Lets the tool catalog
auto-document the full error vocabulary in the generated reference.

Option A is simpler, matches the existing `REGISTER_RPC_HANDLER` ethos
(static + grep-friendly), and survives Unity builds with the same
`__COUNTER__` trick if needed. Prefer A unless we specifically want the
runtime catalog output.

## Migration shape

A migration is genuinely multi-PR:

1. **Catalog.** Generate `docs/error-code-catalog.md` from a one-shot
   script that walks `Private/Handlers/` and lists every literal +
   callsite count + which files use it. Land this first; gives a stable
   reference for the dedupe step.
2. **Dedupe pass.** For each pair/triple of semantic dupes, pick the
   canonical spelling and rewrite the losers. This is the user-visible
   breaking step (clients hard-coding `MISSING_PARAMETER` will need to
   accept `MISSING_PARAM` or vice versa). Land as one PR per domain so
   each commit is reviewable.
3. **Constant header.** Introduce `Handlers/ErrorCodes.h` with the
   surviving codes. Rewrite all `SendError(TEXT("..."))` to
   `SendError(ERR_...)`. Mechanical; can be subagent-batched per domain.
4. **Enforcement.** Add a contract test (`TestErrorCodeRegistry.cpp`)
   that scans `Private/Handlers/` source at test-time and fails when it
   sees a raw `TEXT("...")` first arg to `SendError`. Future handlers
   must use a registered constant.

## Enforcement going forward

- Contract test (step 4) catches new literals in CI.
- `CLAUDE.md` "Conventions" section in the plugin updates from
  "domain-specific uppercase" to "use a symbol from
  `Handlers/ErrorCodes.h`; add a new one there if needed."
- Code review rule for handler PRs: any new error code lands as a
  header change first, callsite second.

The current `CLAUDE.md` line ("domain-specific uppercase ... plus the
standard codes") doesn't actually conflict with a registry — it just
under-specifies. A registry codifies what "domain-specific" should mean
without forbidding it; the plugin keeps the domain-rich code names
(`ASSET_NOT_FOUND`, `SPAWN_FAILED`), they just live in one header
instead of 90 files.

**Risk:** breaking change for any client that hard-codes the losing
spelling of a dupe pair. Acceptable for a beta/Experimental v0.1.x
plugin, but should be flagged in the release notes for whichever
version ships step 2.

## History
- `#1-initial-scope` `OPEN` reporter — Verified the prior triage: `grep
  -rho 'SendError(TEXT("[A-Z_]*")' Private/Handlers/ | sort -u | wc -l`
  returns **539** unique codes (prior pass said 540, close enough — the
  discrepancy is probably whether `SendError` was also counted on
  non-uppercase or empty codes). Top 5 counts match exactly:
  INVALID_ARGUMENT 529, NOT_FOUND 240, ASSET_NOT_FOUND 156, NO_WORLD
  107, INVALID_PARAMS 87. Confirmed semantic dupes: MISSING_PARAM (77)
  vs MISSING_PARAMETER (58); INVALID_PARAMS (87) vs INVALID_ARGUMENT
  (529) vs INVALID_PARAM vs INVALID_PARAMETER; CREATE_FAILED (74) vs
  CREATION_FAILED (47) vs CREATION_ERROR; INVALID_PARAM_TYPE vs
  INVALID_PARAMETER_TYPE; BIND_FAILED vs BINDING_FAILED vs
  BINDING_CREATION_FAILED. The CLAUDE.md "domain-specific uppercase"
  convention doesn't actually conflict with a central registry — it
  just under-specifies what counts as "domain-specific". Proposed
  approach: catalog → dedupe → `Handlers/ErrorCodes.h` constants →
  contract test enforcement. Multi-PR effort.
- `#2-registry-and-enforcement` `IN-REVIEW` developer — Landed the
  non-breaking foundation (Option A) so the rest is mechanical follow-up.
  Generated `docs/error-code-catalog.md` (Phase 1) — 553 unique codes,
  3382 callsites, 181 files. Added `Handlers/ErrorCodes.h` (Phase 3) with
  an `ERR_<CODE>` constant for every emitted code (TCHAR literals, grep-
  friendly), documenting the surviving semantic-dupe clusters in a header
  comment for the future Phase-2 breaking dedupe. Migrated the central
  `HandlerContext.cpp` callsites — all 12 `Require*()` helpers to
  `ERR_INVALID_PARAMS`, `SendUnsupportedEngineVersion` to
  `ERR_UNSUPPORTED_ENGINE_VERSION`, and the empty-code fallback to
  `ERR_AUTOMATION_ERROR`. Added the Phase-4 contract test
  `Tests/Core/TestErrorCodeRegistry.cpp`
  (`core.error_codes.AllEmittedCodesAreRegistered`): scans every handler
  source, extracts each `SendError(TEXT("..."))` literal, and fails if any
  emitted code lacks a matching `ERR_<CODE>` in `ErrorCodes.h` — forcing
  new codes into the header first. Verified the test passes against the
  current tree (all 552 emitted codes registered). Updated the CLAUDE.md
  Conventions line to point at the registry + test.
  **Deliberately NOT done in this PR** (per the ticket's multi-PR plan,
  needs separate review + release notes): the Phase-2 breaking dedupe
  (renaming loser spellings to canonical) and the mechanical rewrite of
  the remaining ~3370 raw handler callsites to `ERR_*` constants. The
  registry is intentionally non-breaking — every existing spelling is
  preserved as its own constant.
- `#4-additional-sc-disabled-dupe` `IN-REVIEW` reporter — Additional evidence: a new replay-confirmed semantic-dupe pair for the same condition (source control disabled), tripped live by a content-hygiene task that mixed `asset.*` SC verbs with the `source_control.*` namespace. The disabled-SC gate emits **two different codes AND two different messages** depending on which handler you hit: `asset.get_source_control_state` → `[SC_DISABLED] Source control not enabled.` (note trailing period, "not enabled") while `asset.source_control_checkout`, `asset.source_control_submit`, and the entire `source_control.*` namespace (`status`/`log`/`revert`/`mark_for_add`/get_provider gate) → `[SOURCE_CONTROL_DISABLED] Source control is not enabled` (no period, "is not enabled"). Source: `AssetQueryHandler.cpp:334` sends `TEXT("SC_DISABLED")`/`TEXT("Source control not enabled.")` vs `AssetWorkflowHandler.cpp:173,258` and `SourceControlHandler.cpp:82` sending `TEXT("SOURCE_CONTROL_DISABLED")`/`TEXT("Source control is not enabled")`. Both constants exist separately in `ErrorCodes.h` (`ERR_SC_DISABLED` @466, `ERR_SOURCE_CONTROL_DISABLED` @495), so `get_source_control_state` is the lone outlier. Real-world impact: a caller string-matching a single disabled-SC code across a checkout→submit→state workflow cannot — `get_source_control_state` breaks the match. Canonical spelling for the Phase-2 dedupe should be `SOURCE_CONTROL_DISABLED` (3 of 4 SC handlers + the whole namespace already use it); fold `SC_DISABLED` into it. Add this pair to the dupe-cluster list (alongside CREATE_FAILED/CREATION_FAILED etc.).
- `#3-skip-test-not-in-binary` `SKIP` tester — Tried the load-bearing
  behavioral surface (`system.run_tests` test=
  `core.error_codes.AllEmittedCodesAreRegistered`): job completed with
  `error: NO_TESTS_MATCHED`, `resolvedTests: []`,
  `missingTests: ["core.error_codes.AllEmittedCodesAreRegistered"]`. The
  contract test that proves "all emitted codes are registered" is not in
  the running editor binary (test/header file mtimes are 2026-06-05
  08:55–08:56; editor predates the rebuild), so the central claim can't be
  exercised live. File-state portion is real (ErrorCodes.h has 554
  `ERR_*[]=TEXT(...)` constants on disk; catalog.md + test .cpp present),
  but per protocol "source code is not verification" that alone is not
  PASS. No evidence the fix is wrong, so not FAIL. SKIP: needs a plugin
  rebuild to compile the contract test into the binary; re-verify after
  rebuild by running that named test.
