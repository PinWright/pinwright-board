---
id: B-error-code-adoption-test-scans-comments
title: "RegistryAdoptingFilesUseConstantsOnly scans source text without stripping comments, so naming a constant in a comment - even to explain avoiding it - flips a file to adopting and fails the suite"
status: OPEN
severity: Medium
category: bug
tags: [test-gap, error-codes, registry, RegistryAdoptingFilesUseConstantsOnly, false-positive, comment-scanning]
encounters: 1
lastSeen: 2026-08-28
---

# A comment explaining the trap is enough to spring it

`PinWright.core.error_codes.RegistryAdoptingFilesUseConstantsOnly` decides whether a handler file
"uses the ErrorCodes registry" by scanning its source text for `ErrorCodes::ERR_`. It does not
neutralize comments first. So a **comment** mentioning the constant flips the file to "adopting", and
the test then fails it for every code it spells by hand.

Reproduced live, not inferred. `Handlers/Environment/EnvironmentHandler.cpp` hand-spells 40 codes and
is not on the grandfathered baseline. A fix deliberately used a raw literal for that reason and wrote
a comment saying so:

```cpp
// Raw literal, not ErrorCodes::ERR_INVALID_PARAMS: this file spells all 40 of
// its codes by hand and is not on the registry's grandfathered baseline ...
```

The next suite run failed with all 16 of that file's codes listed. The code was correct; the sentence
explaining why it was correct was the defect. Worked around by rewording the comment so it does not
contain the token.

This is a false positive in a guard test, so the cost is not just a red: the obvious reading of the
failure is "convert this file to constants", which is the opposite of what the file needs and would
have been a 40-site change made for no reason.

**Fix:** neutralize comments and string literals before the scan. **The technique already exists in
this codebase** — `Tests/Infra/TestDeclaredParamCoverage.cpp` ships `NeutralizeSourceText` for exactly
this purpose, written for the declared-parameter walk. Extracting it to a shared test helper and using
it here fixes this and prevents the next scanner from repeating it; `core.error_codes.AllEmittedCodesAreRegistered`
and `infra.wiki_src.SourcePagesFollowRenderingRules` scan source the same way and are worth checking
for the same blindness at the same time.

## History
- `#1-comment-tripped-the-guard` `OPEN` reporter — Found when suite run 4 of the 2026-08-26/27 fix
  batch went red on `EnvironmentHandler.cpp` immediately after a fix that added **no** constant
  reference to it. `grep -c 'ErrorCodes::ERR_'` on the file returned 1, and the single hit was inside
  a `//` comment. Swept every other handler `.cpp` for the same shape (all `ErrorCodes::ERR_` hits in
  comments only): no other file currently has it, so this is the first and only instance.
