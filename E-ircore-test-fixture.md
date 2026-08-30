---
id: E-ircore-test-fixture
title: "IrCore: extract minimal shared test fixture primitives"
status: DONE
severity: Medium
category: ergonomic
tags: [ircore, testing, refactor, bpir, mgir, agir, scir, btir, msir, nir, deduplication, preventative]
---

# IrCore: extract minimal shared test fixture primitives

The IR test suites currently duplicate ~650–700 LoC of fixture boilerplate
across BPIR, MGIR, and AGIR. With four new IRs on the near-term roadmap
(SCIR for state-tree-like graphs, BTIR for behavior-tree-like graphs,
MSIR for MetaSound, NIR for Niagara), letting each one copy the same
asset-cleanup + round-trip + diagnostic-assertion shape will multiply
the duplication by 4×. Cheaper to extract once now than to retrofit later.

## Audit evidence

- **BPIR** — 57 test files (~31k LoC) backed by mature shared headers
  `Tests/BPIR/CompilerTestUtils.h` (~219 LoC) and
  `Tests/BPIR/BpirGraphTestHelpers.h` (~210 LoC). Pattern is healthy
  inside the BPIR subtree but not exposed to other IRs.
- **MGIR** — 1 IR-level test (`TestMGIRCompositeInline.cpp`). No
  MGIR-specific helper header; asset factory + cleanup boilerplate is
  inline. New material-IR tests will reproduce the BPIR pattern from
  scratch unless a shared base exists.
- **AGIR** — 9 tests with a thin `TestAGIRFixtures.h` (~63 LoC, 3
  helpers). Most AnimBP construction is inline. Halfway between MGIR's
  ad-hoc state and BPIR's mature helpers.

### Cross-IR duplication (measured)

| Concern | LoC duplicated across BPIR/MGIR/AGIR |
| --- | --- |
| Asset-path generation + RAII-style cleanup | ~150 |
| Factory-create-fresh-asset (BP / Material / AnimBP) | ~120 |
| Compile-then-check-errors plumbing | ~200 |
| Decompile-and-dump-warnings | ~80 |
| Round-trip recompile (compile → decompile → recompile) | ~125 |
| **Total** | **~675** |

Additionally, **no IR uses golden-file assertions today** — every test
uses token-substring matches. That remains a follow-up concern; this
ticket lands only the primitives already proven by current tests.

## Proposed extraction

New folder `Plugins/PinWright/Source/PinWright/Private/Tests/IrCore/`
with a single header `IrTestFixture.h` (and a thin `.cpp` for the
non-template bodies):

```cpp
namespace IrTest {
  // RAII scratch package + asset name; destructor calls
  // CleanupTestAsset() so abort/throw paths don't leak.
  struct FScratchAsset {
    FString PackagePath;
    FString AssetName;
    explicit FScratchAsset(const TCHAR* Prefix);   // /Game/EditorAutomationTests/<Prefix>_<GUID>
    ~FScratchAsset();
  };

  template<typename TAsset, typename TFactory>
  TAsset* CreateFactoryAssetAtPath(const FString& PackagePath, ...);

  template<typename TAsset, typename TFactory>
  TAsset* CreateFactoryScratchAsset(const FScratchAsset& Scratch, ...);

  FString ReplaceScratchAssetName(
      const FString& InputText,
      const FScratchAsset& Source,
      const FScratchAsset& Target);

  // Substring assertion over typed diagnostic arrays.
  template<typename TError>
  bool ErrorsContain(const TArray<TError>& Errors, const FString& Substring);
}
```

`FRoundTrip`, golden-file assertions, BPIR migration, and bulk AGIR
callsite migration are explicitly deferred to follow-up tickets.

Estimated size: **~100–150 LoC new code** (header + small .cpp). Each new
IR saves repeated scratch/factory/diagnostic boilerplate downstream.

## Scope and migration order

**In scope:**

1. Land `Tests/IrCore/IrTestFixture.h` + `.cpp` with `FScratchAsset`,
   `CreateFactoryAssetAtPath`, `CreateFactoryScratchAsset`,
   `ReplaceScratchAssetName`, and `ErrorsContain` — no behaviour change
   for existing tests.
2. Port MGIR's single IR-level test (`TestMGIRCompositeInline.cpp`)
   onto the new helpers immediately — proves the API, costs almost
   nothing.
3. Convert AGIR's `TestAGIRFixtures.h` into a thin shim that delegates
   to the shared API (keep the AGIR-facing names so callsites don't
   churn).

**Out of scope (do later, separately):**

- Retrofitting BPIR's 57 test files. That migration is mechanical and
  risk-free but large; keep it off this ticket so the refactor lands
  small. File a follow-up ticket once the shared API has settled
  through at least one new IR.
- Introducing golden files for existing tests. New IR tests should
  get golden-file assertions through a separate ticket once the API is
  proven by current helper consumers.
- Adding `FRoundTrip`.
- Bulk migration of AGIR callsites beyond `CreateFreshAnimBlueprint`.

## Soft prerequisite for upcoming IR work

This ticket is a **soft prerequisite** for the four new-IR tickets on
the roadmap (none filed yet at time of writing):

- **SCIR** (state-chart-style graphs; cross-ref the existing
  `F-state-tree-conditions-evaluators` work area).
- **BTIR** (behavior-tree IR; cross-ref
  `F-bt-create-blueprint-node-classes` and
  `F-bt-attach-decorator-service-to-parent`).
- **MSIR** (MetaSound IR; cross-ref the `F-metasound-*` family,
  especially `F-metasound-no-destructive-graph-ops`,
  `F-metasound-no-input-output-mutation`).
- **NIR** (Niagara IR; cross-ref the `F-niagara-graph-create-node`,
  `F-niagara-set-module-script` cluster).

If any of those start before this lands, raise the severity to High —
the cost of retrofitting them is exactly what this ticket exists to
avoid.

## Why severity Medium

- Not blocking any current work; BPIR/MGIR/AGIR all compile and test
  green today.
- Cheap to land now (~250 LoC new code, one MGIR test ported, AGIR
  shim).
- Expensive to defer once new-IR tests start: each missed extraction
  multiplies the eventual retrofit by 4.

**Workaround:** none required; current per-IR boilerplate works.

**Fix:** Land `Tests/IrCore/IrTestFixture.h`/`.cpp` with the minimal shared primitives proven by current tests: scratch asset RAII cleanup, factory-created assets, asset-name replacement for round-trip text, and diagnostic substring helpers. Port `TestMGIRCompositeInline.cpp` to prove the helpers. Reduce `TestAGIRFixtures.h::CreateFreshAnimBlueprint` to delegate its factory path through the shared helper while keeping AGIR-specific Lyra loaders local. Defer `FRoundTrip`, golden-file assertions, BPIR migration, and bulk AGIR callsite migration to follow-up tickets.

## History
- `#1-initial-spec` `OPEN` reporter — Filed pre-emptively before SCIR/BTIR/MSIR/NIR tickets land. Audit measured ~675 LoC of fixture duplication across BPIR (57 files, mature helpers), MGIR (1 file, no helper header), AGIR (9 files, thin helper). No golden-file pattern in any IR today. Extraction proposal: `Tests/IrCore/IrTestFixture.h` with `FScratchAsset` (RAII), `CreateScratchAsset<T>`, `FRoundTrip<C,D>`, `AssertMatchesGolden`, `ErrorsContain<T>`. Scope strictly limited to scaffolding extraction + MGIR port + AGIR shim; BPIR retrofit deferred. Soft prereq for the four upcoming new-IR tickets — escalate to High if any of them start before this lands.
- `#2-narrow-ir-fixture-api` `IN-REVIEW` developer - Narrowed the original FRoundTrip/golden/specialization-heavy fixture proposal to proven IrTest scratch/factory/diagnostic primitives, ported MGIR composite inline coverage, and made the AGIR fixture delegate asset creation without migrating unrelated AGIR callsites.
- `#3-verify-fixture-tests` `DONE` tester — Verified: `system.run_tests` exact tests `EditorAutomationRpcGateway.material.mgir.CompositeInlineFlatten` and `EditorAutomationRpcGateway.AGIR.Interface.RoundTrip` both resolved with `missingTests:[]` and completed with `has_errors:false`; source check confirmed `IrTestFixture.{h,cpp}` exists, MGIR includes it, and `TestAGIRFixtures.h::CreateFreshAnimBlueprint` delegates through `IrTest::CreateFactoryAssetAtPath`.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
