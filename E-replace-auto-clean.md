---
id: E-replace-auto-clean
title: "`compile_bpir` replace mode should auto-clean orphans"
status: DONE
severity: ""
category: ergonomic
tags: []
---

# `compile_bpir` replace mode should auto-clean orphans

Replace mode leaves orphaned old body nodes. Requires separate `find_orphaned_nodes` call.

**Proposal:** Auto-clean orphans as part of replace compile step.

## History
- `#1-manual-cleanup-required` `OPEN` reporter — Every replace-mode compile left orphans requiring manual cleanup call.
- `#2-added-phase0b-cleanup` `IN-REVIEW` developer — Added Phase 0b orphan cleanup in BpirCompiler.cpp after Phase 0 deletion. Collects affected graphs, then fixed-point loop finds pure nodes (no exec pins, not comments/entry nodes) with all outputs disconnected and deletes them. Added `OrphansRemoved` field to `FCompileResult`, reported as `orphansRemoved` in response JSON when > 0.
- `#3-verified-auto-orphan-clean` `DONE` tester — Verified: compiled TestOrphan event with GetName+PrintString, then replace-mode compiled without GetName. Response included `orphansRemoved: 1`. Orphaned pure node auto-cleaned.
