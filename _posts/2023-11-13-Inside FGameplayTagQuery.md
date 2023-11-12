---
title: Inside FGameplayTagQuery
date: 2023-11-13 01:13:14 +0800
categories: [Unreal Engine]
tags: [gameplay]    # TAG names should always be lowercase
lang: en
---

## What is `FGameplayTagQuery`

Quoted from source code comments:

> An `FGameplayTagQuery` is a logical query that can be run against an `FGameplayTagContainer`.  A query that succeeds is said to "match".
Queries are `logical expressions` that can test the intersection properties of another tag container (all, any, or none), or the matching state of a set of **sub-expressions**
(all, any, or none). This allows queries to be arbitrarily recursive and very expressive.  For instance, if you wanted to test if a given tag container contained tags 
((A && B) || (C)) && (!D), you would construct your query in the form ALL( ANY( ALL(A,B), ALL(C) ), NONE(D) )

## Why use FGameplayTagQuery

Because when using `FGameplayTagQuery` for logical matching, the number of tags and matching logic can be arbitrary, and it supports **nesting on logic**, unlike:
- `FGameplayTag` limits the use of only 1 tag (although in addition to match itself, it can also be used to match parent tags)
- `FGameplayTagContainer` has only limited matching logic (which is one of AND, OR, NOT, depends on how the code is used)
- Before `FGameplayTagRequirements` was added to the `FGameplayTagRequirements::TagQuery` member, it had only two `FGameplayTagContainer` members, corresponding to the "AND" and "NOT" matching logic, which is still limited (after the time of TagQuery member added, you can use `FGameplayTagRequirements ::ConvertTagFieldsToTagQuery` to obtain a `Query` object which is the logical combination of two tag containers)

## Implementation of FGameplayTagQuery

### data structure

- `TokenStreamVersion` version number, retains data to facilitate subsequent possible implementation changes, corresponding to the enumeration type `EGameplayTagQueryStreamVersion`

- `TagDictionary` The tag array after deduplication, which comes from the tags that need to be used in logical expressions

- `QueryTokenStream` is a set of metadata that stores the `version number` (redundant storage), whether there is a logical expression expression, the logical expression type, the number of tags used, and the index in `TagDictionary`. It is the key to achieving memory-efficient and fast evaluation.

- `UserDescription` string, customized description information

- `AutoDescription` string, automatically generated description information

### Generation method

#### Use C++ to construct query objects

Use the `Builder Pattern` API to construct logical expressions:
```C++
	FGameplayTagQuery TagQuery;
	const FGameplayTagContainer TagContainerA{};
	const FGameplayTagContainer TagContainerB{};
	const FGameplayTag TagC{};

	TagQuery.Build(FGameplayTagQueryExpression().AllExprMatch()
		.AddExpr(FGameplayTagQueryExpression().AnyTagsMatch().AddTags(TagContainerA))
		.AddExpr(FGameplayTagQueryExpression().NoExprMatch()
				.AddExpr(FGameplayTagQueryExpression().NoTagsMatch().AddTags(TagContainerB))
				.AddExpr(FGameplayTagQueryExpression().AnyTagsMatch().AddTag(TagC))), FString{TEXTVIEW("Test Logic")});

	// Randomly written logic, it is not recommended to try to understand it
	// I use indentation levels to represent nesting levels, and one line in each level defines a logical expression
```

Brief process of `FGameplayTagQuery::Build`:
1. Write the version number and user description information, and reset those key data
2. Write the "version number" and "whether it contains a logical expression" information to elements 0 and 1 of `QueryTokenStream`
3. Parse the `logical expression` and write the enumeration of the expression type in `QueryTokenStream`, which is of type `EGameplayTagQueryExprType`. For "non-nested expression", write the array number of tags used by it, then add each tag to `TagDictionary` which is deduplicated, and write the index; for "nested expression", parsing it recursively

To put it simply, the `Build` process uses `depth-first traversal` to flatten the logical expressions of the tree structure into an array.

#### Use the editor to construct the query object

The underlying logic is consistent with the C++ structure, except that because `FGameplayTagQueryExpression` is not `USTRUCT`, and `InstancedStruct` has not yet been born, it is "mirrored" with UObject (personal guess), so there are `UEditableGameplayTagQueryExpression` and related types to support expression nesting during editing and providing better debugging information, you can refer to `FGameplayTagQuery::BuildFromEditableQuery`

### `FGameplayTagQuery::Matches`, test the logical expressions

The auxiliary type `FQueryEvaluator` is used to hold the immutable reference of `TagQuery`, record the current metadata index and detect whether there are read errors. According to the read expression type, the corresponding logical definition is executed. Every time the token array is read, it will detect whether there is a read error. You can refer to `FQueryEvaluator::EvalExpr`

## Conclusion
I feel that the implementation of `FGameplayTagQuery` is quite clever and provides powerful matching logic, which is suitable for any needs that require Tag matching functionality. However, the editor logic contains code duplication, is not elegant enough, and the detection of reading errors is not rigorous, so the follow-up, frequent testing is not that necessary. But overall, flaws do not cover up strengths.