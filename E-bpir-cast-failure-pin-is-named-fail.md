---
id: E-bpir-cast-failure-pin-is-named-fail
title: "BPIR cast failure label is `fail`, but the error for a wrong guess lists no valid pin names, so `failure` / `cast_failed` cost two round trips to eliminate"
status: OPEN
severity: Low
category: enhancement
tags: [bpir, compile_bpir, cast, error-message, docs]
encounters: 1
lastSeen: 2026-09-03T01:30:00Z
---

# BPIR cast-failure pin name is discoverable only from an example page

## Symptom

The statement form of a cast takes `[success -> @ok, fail -> @nope]`. Guessing the more
natural `failure` or `cast_failed` gives:

```
[COMPILE_FAILED] Line 2: Could not find exec output pin 'failure' on node;
                 Line 6: Could not find exec output pin 'failure' on node
```

The message does not list what the valid names are, which is what every other
"unknown name" error in the plugin does — compare `material.compile_mgir`'s
`Available inputs: Input. Did you mean 'Input'?` and `level.describe`'s
`Did you mean: level.delete; level.stream; ...`.

The correct name is documented, in `bpir.instructions.md` line 159 and on the
`bpir.examples.cast-with-failure` page, but a decompiled cast never shows it: the
decompiler emits only the connected pin, so a cast whose failure pin is unwired
round-trips as `cast<T>(%x) [success -> @ok]` and the second label is invisible.

## Suggested change

Append the node's actual exec output pin names to the `Could not find exec output pin`
error, e.g. `Available exec outputs: success, fail`. That single change makes the name
discoverable from the failure itself, which is where an author is standing when they
need it.
