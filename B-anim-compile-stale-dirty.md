---
id: B-anim-compile-stale-dirty
title: "anim.compile uses stale skeleton state and dirties failures"
status: OPEN
severity: Critical
category: bug
tags: [animation, skeleton, rollback]
encounters: 1
---

# anim.compile uses stale skeleton state and dirties failures

After a skeleton gains a bone in the same editor session, validation sees the new hierarchy but
compiling a loaded animation cannot add a track for that bone. The skeleton generation does not
change, so dependent animation mappings remain stale until restart. Separately, a failure after
animation model mutation leaves its package dirty and the partial model in memory.

A successful hierarchy write must refresh dependent mappings. Animation compilation must restore
both object content and the prior package-dirty state on every post-mutation failure.

## History
- `#1-stale-and-dirty-reproduced` `OPEN` reporter — Same-session validation saw two bones while compile failed to add the new track; an injected post-write failure left one dirty content package.
