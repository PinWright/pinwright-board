---
id: B-skeleton-compile-postcondition
title: "skeleton.compile can report success for an incomplete asset"
status: OPEN
severity: Critical
category: bug
tags: [skeleton, compile, postcondition]
encounters: 1
---

# skeleton.compile can report success for an incomplete asset

A skeleton recompile can finish with the reference hierarchy containing every source bone while
the asset's retargeting table contains fewer entries. The verb still reports success because it
does not verify the finished UObject. Reproduce by compiling 21 bones, recompiling 24, forcing the
retargeting table back to 21 at the final boundary, and observing a successful result.

The compiler must read the finished asset and refuse success with a stable diagnostic whenever
the hierarchy, transforms, retargeting table, or source-owned metadata differs from source.

## History
- `#1-incomplete-write-reproduced` `OPEN` reporter — A controlled 24-reference-bone and 21-retarget-entry asset still returned compile success and no post-condition diagnostic.
