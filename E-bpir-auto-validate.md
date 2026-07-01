---
id: E-bpir-auto-validate
title: "`compile_bpir` should auto-validate via `blueprint_compile`"
status: DONE
severity: ""
category: ergonomic
tags: []
---

# `compile_bpir` should auto-validate via `blueprint_compile`

Currently requires 4-step dance: compile_bpir → blueprint_compile → fix pins → blueprint_compile. Should optionally run full UE BP compile after emitting nodes and include errors in response.

## History
- `#1-separate-compile-required` `OPEN` reporter — Every BPIR compile required a separate blueprint_compile to catch enum/delegate errors.
- `#2-verified-inline-compile-fields` `DONE` tester — Verified: compile_bpir response now includes `compiled`, `status`, `errors`, `warnings` fields from UE BP compile. TestEnum on W_PhotoPopup returned compiled:true with status:UpToDate inline.
