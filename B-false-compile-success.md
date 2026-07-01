---
id: B-false-compile-success
title: "`compile_bpir` reported `compiled: true` for broken BPs"
status: DONE
severity: High
category: bug
tags: []
---

# `compile_bpir` reported `compiled: true` for broken BPs

## History
- `#1-initial-repro` `OPEN` reporter — blueprint_compile returned compiled:true for BPs with fatal enum/delegate/signature errors.
- `#2-error-array-response` `IN-REVIEW` developer — Added error array to compile response, reads MessageLog after FKismetEditorUtilities::CompileBlueprint.
- `#3-verified-error-status` `DONE` tester — Verified: compile now returns status:"Error" with error details for enum, delegate, and signature issues.
