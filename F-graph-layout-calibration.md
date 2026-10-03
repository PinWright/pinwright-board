---
id: F-graph-layout-calibration
title: "Calibrate FGraphLayoutMetrics thresholds against human-authored vs MCP graph corpora"
status: WONTFIX
severity: Low
category: feature
tags: [layout, metrics, calibration]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# Calibrate FGraphLayoutMetrics thresholds against human-authored vs MCP corpora

`F-graph-layout-metrics-core` lands the pure `FGraphLayoutMetrics` util and a
node-size adapter seam, but it deliberately ships **no calibrated
thresholds** — the unit test only asserts clean-high / overlap-low, it does
not derive the floor below which a real graph is "poor". This ticket is the
open-ended measurement pass that was split out of the metrics-core ticket so
that ticket stayed a bounded, unit-testable unit of work.

## What's missing

Run a **calibration pass** scoring two corpora, then record the derived
thresholds where the metrics core / settings exposes them (so the
test-workflow can read/restate the value rather than hardcode a fresh one —
see `docs/layout-quality-integration.md` "The calibrated threshold"):

- **"Good" reference** — human-authored graphs (project Blueprints /
  Content-Examples).
- **"To-improve"** — current MCP / auto-layout output.

From that data:
- **Overlap is an absolute fail** (no overlap tolerated on a clean layout).
- **Relative metrics' floor = ~P10 of the human-authored distribution.**

THEN, using the calibration data, file the deferred follow-up tickets:
a **BPIR-refinement** ticket and an **edge-crossing-reduction** ticket, each
seeded with the measured gap between MCP output and the human floor.

## Why this is split from metrics-core

The corpus is undefined (which Blueprints? which Content-Examples assets?),
the derived thresholds are not consumed by the metrics-core unit test, and
none of the seven `F-graph-layout-metrics-core` dependents block on this
calibration — they consume the util and the estimator seam. So this is a
separate deliverable with its own (currently open-ended) acceptance bar, not
part of the keystone util's definition of done.

## Acceptance criteria

- Thresholds recorded in a single source of truth (a settings entry or a
  constant in the metrics core), with the corpus described.
- `F-...-bpir-refinement` and `F-...-edge-crossing-reduction` follow-up
  tickets filed, each carrying the measured MCP-vs-human gap.

## Severity justification

**Medium.** Soft blocker: the metrics util is fully usable without calibrated
thresholds (the L2 engine regression tests guard layout quality via the unit
suite), and `docs/layout-quality-integration.md` defers the runtime
threshold consumption to L3. This pass only adds the calibrated floor and
seeds the two follow-ups; no crash, no data corruption.

## History
- `#1-split-from-metrics-core` `OPEN` reporter — Split out of F-graph-layout-metrics-core, which was over-scoped: the calibration pass (score human-authored vs MCP corpora → derive the absolute overlap fail + ~P10 relative-metric floor → record the threshold → file BPIR-refinement and edge-crossing follow-ups seeded with the measured gap) is open-ended research with an undefined corpus that none of the metrics-core dependents block on, so it is its own deferred ticket gated on F-graph-layout-metrics-core landing.
- `#2-re-rated` `OPEN` triage — Severity Medium -> Low. An internal calibration/research pass with no caller-facing blocker (its own justification says the metrics util is fully usable without it); layout quality is cosmetic, so it rates Low.
- `#3-moot-no-consumer` `OPEN` developer — Moot at 7230b41d. The only consumer of the calibrated floor is the L3 LayoutCheck in docs/layout-quality-integration.md:38-49, which scores graphs through `*.layout_report` (:25-27, :72-76); that RPC is F-graph-layout-report-rpc, WONTFIX (#2: no consumer, and the shipped core is overlap-free by construction). GraphLayout::ComputeGraphLayoutMetrics (Source/PinWright/Private/Layout/GraphLayoutMetrics.h:107) has no non-test caller, and the layout tests assert zero overlaps directly (Tests/Layout/PwGraphLayoutTestUtils.h:227) rather than a score floor. The two follow-ups it was to seed are already covered: BPIR now lays out with PwGraphLayout::ArrangeEdGraph (Compiler/BpirCompiler.cpp:2178, B-layout-engine-ba-derived DONE), and the core is overlap-free by construction with barycenter crossing sweeps (Layout/PwGraphLayout.h:14-24). Leftover doc debt only: layout-quality-integration.md still lists this pass and layout_report as prerequisites.
- `#4-wontfix-moot` `WONTFIX` developer — Closed as moot per #3: its only consumer, F-graph-layout-report-rpc, is WONTFIX and the follow-ups are covered by F-graph-layout-core.
