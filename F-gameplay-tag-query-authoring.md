---
id: F-gameplay-tag-query-authoring
title: "No recursive gameplay tag query authoring surface"
status: DONE
severity: High
category: feature
tags: [gas, gameplay-tags, query, authoring]
---

# No recursive gameplay tag query authoring surface

The registry CRUD slice now covers project-level gameplay tag creation,
removal, source creation, and listing, but there is still no RPC for composing
`FGameplayTagQuery` values. Ability activation requirements, target tag
requirements, and gameplay-effect gates often need recursive query expressions
such as `AnyTagsMatch`, `AllTagsMatch`, `NoTagsMatch`, nested
`AnyExpressionsMatch`, or nested `AllExpressionsMatch`.

Without a typed RPC, callers either hand-edit serialized query internals or
fall back to flat `FGameplayTagContainer` fields and lose expressive power.

**Fix:** Add a follow-up query-authoring method under `gameplay_tags` that
accepts a recursive JSON expression tree, builds an `FGameplayTagQuery`, returns
a human-readable description and serialized size metadata, and can optionally
write the query into a target asset property.

Proposed shape:

```
gameplay_tags.build_query(
    expression: {
        op: "any_tags_match"|"all_tags_match"|"no_tags_match"
          |"any_expressions_match"|"all_expressions_match"|"no_expressions_match",
        tags?: string[],
        expressions?: [ ... ]
    },
    target?: { assetPath, propertyPath }
) -> {
    description: string,
    tokenStreamBytes: number,
    wrote: bool
}
```

Validation should reject leaf expressions without tags, composite expressions
without children, mixed leaf/composite payloads when the operation does not
support them, unknown operation names, and excessive recursion depth.

## History
- `#1-split-from-registry-namespace` `OPEN` reporter — Split from `F-gameplay-tags-namespace` because recursive `FGameplayTagQuery` expression composition and optional property writes do not share implementation with INI-backed gameplay tag registry CRUD/listing.
- `#2-build-query-handler` `IN-REVIEW` developer — Added gameplay_tags.build_query in Handlers/GameplayTags/GameplayTagBuildQueryHandler.cpp with recursive expression parser, depth guard (default 12), six op kinds, optional target asset write via PropertyUtils::ResolveNestedPropertyPath under FScopedTransaction, and regression test Tests/GameplayTags/TestGameplayTagBuildQuery.cpp covering nested composite + EMPTY_TAGS validation. tokenStreamBytes derived via FMemoryWriter+Serialize.
- `#3-verify-fix` `DONE` tester — Verified: schema via `gameplay_tags.build_query?` exposes expression/target/maxDepth/description params; recursive call with op=all_expressions_match wrapping any_tags_match + no_tags_match over real tags returned {description:"verify-test", tokenStreamBytes:121, wrote:false}; validation rejected empty leaf tags with EMPTY_TAGS and unknown op with UNKNOWN_OP.
