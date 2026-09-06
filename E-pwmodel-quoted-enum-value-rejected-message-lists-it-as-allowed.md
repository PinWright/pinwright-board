---
id: E-pwmodel-quoted-enum-value-rejected-message-lists-it-as-allowed
title: "A quoted enum value is rejected by the .pwmodel parser and the error prints the value you passed as one of the allowed ones: `mode=\"repair_or_delete\"` -> \"expects one of: delete_only, repair_or_delete, repair_or_skip\""
status: OPEN
severity: Low
category: ergonomic
tags: [pwmodel, model-validate, model-compile, parser, enum, diagnostics, error-message, describe_ops]
encounters: 1
lastSeen: 2026-09-06T06:20:00Z
---

# The message names the value it just refused

Enum-typed parameters in `.pwmodel` take a BARE token. Quoting one is a hard parse error
(`PWSRC_BAD_VALUE`), and the message lists the allowed values — including, verbatim, the one the
caller wrote. Every other string-shaped argument in the format is quoted (`material="Receiver"`,
`filePath`, `mode` on the RPC side), so quoting an enum is the natural mistake, and the diagnostic
actively argues that the source is already correct.

## What was called

```
model.validate {text: "pwmodel 0 ... part p {
    box size=(1,1,1) material=\"M\"
    remove_degenerates mode=\"delete_only\"
}"}
```

## What happened

```
[MODEL_PARSE_FAILED] [PWSRC_BAD_VALUE] line 3, col 29:
Parameter 'mode' on 'remove_degenerates' expects one of: delete_only, repair_or_delete, repair_or_skip.
```

Format-wide, not one op: `uv ... mode="box"` returns
`Parameter 'mode' on 'uv' expects one of: box, planar, cylindrical, xatlas, patch_builder, layout.`
The same line unquoted compiles clean in both cases (verified: `mode=delete_only` and
`mode=box` both validate, `success: true`).

Cost here: a `.pwmodel` written in a previous session carried `mode="repair_or_delete"` at two
sites; the file did not parse at all, and the message read as though the value was fine, so the
first read was "the parser has lost the enum" rather than "drop the quotes".

## What was expected

Any one of:

1. **Accept the quoted form.** A quoted enum is unambiguous — there is no string-typed reading of
   `mode` to collide with.
2. **Say what is wrong**: `... expects one of: ... (a bare token, not a quoted string)`, or the
   sharper form when the stripped text matches a legal value: `'"repair_or_delete"' is a string;
   this parameter takes the bare token repair_or_delete`.
3. At minimum, **say it in the discovery surface**. `model.describe_ops {op: ...}` reports
   `"type": "enum"` with `allowedValues`, and nothing on that page or in
   `model.authoring.document-structure` says enum values are unquoted while every other
   `key="value"` in the same statement is quoted.

## Workaround

Drop the quotes.

## Root cause — guess, no source read taken

The value parser almost certainly matches the enum against the raw lexeme including its quote
characters, then formats the failure from the static allowed-value table without echoing what was
received. Echoing the received text in the message would have made this self-diagnosing.
**Inference from the message shape; no plugin source was opened.**

## Severity

**Low.** Loud, immediate, and one edit to fix — but it is a first-contact papercut on a format whose
whole point is that an agent can author it from the wiki, and the message is misleading rather than
merely terse.

## Related

- `B-enum-param-syntax-mismatch` (DONE) — the same class of defect on the BPIR side; that one was
  about which spellings compile, this one is about a spelling that does not and a message that says
  it does.
- `B-animation-enum-tokens-fall-through-to-default` — enum tokens silently mis-parsed elsewhere.

## History
- `#1-filed` `OPEN` reporter — Hit on `remove_degenerates mode="repair_or_delete"` in
  `Content/FPS/Weapons/Meshes/SM_WPN_AR.pwmodel` (two sites, whole file failed to parse).
  Reproduced minimally on `remove_degenerates` and on `uv` with `model.validate {text: ...}`,
  and confirmed the bare-token form of each validates clean. UE 5.8, this checkout.
