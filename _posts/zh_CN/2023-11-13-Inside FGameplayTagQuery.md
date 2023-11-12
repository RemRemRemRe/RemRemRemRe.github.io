---
title: 简析FGameplayTagQuery
date: 2023-11-13 01:13:14 +0800
categories: [Unreal Engine]
tags: [gameplay]    # TAG names should always be lowercase
lang: zh_CN
---

## 什么是`FGameplayTagQuery`

引用自源码注释:

> `FGameplayTagQuery`是可以查询`FGameplayTagContainer`中的一组Tag是否满足特定条件的一组`逻辑表达式`. 一个匹配成功的query则表示对应的tag container是满足条件的. 其中`逻辑表达式`支持"与,或,非",以及**嵌套的子表达式**. 在内部,它将这些逻辑表达式以字节流的形式表达,实现内存上的高效率,并且运行时可以快速检验

## 为什么要使用FGameplayTagQuery

因为使用`FGameplayTagQuery`进行逻辑匹配时,tag数量和匹配逻辑都可以是任意的,且**支持逻辑嵌套**,而不像:
- `FGameplayTag`限定了只能用1个tag(虽然除了本身,也可以用于匹配父级tag)
- `FGameplayTagContainer`只有有限的匹配逻辑(与,或,非中的哪种匹配逻辑,取决于代码如何使用)
- `FGameplayTagRequirements`在未加入`FGameplayTagRequirements::TagQuery`成员之前,它只有两个`FGameplayTagContainer`成员,对应了"与","非"的匹配逻辑,依然有限(在TagQuery成员加入的改动之后,可以用`FGameplayTagRequirements::ConvertTagFieldsToTagQuery`获得两个tag container逻辑合并之后的`Query`对象)

## FGameplayTagQuery原理

### 数据结构

- `TokenStreamVersion` 版本号,保留数据以便于处理后续可能的实现方式变更,对应枚举类型`EGameplayTagQueryStreamVersion`

- `TagDictionary` 去重后的tag数组,来自于逻辑表达式中需要用到的tag

- `QueryTokenStream` 一组元数据,存储了`版本号`(冗余存储),是否有逻辑表达式表达式,逻辑表达式类型,用到的tag数量,以及每个用到的tag在`TagDictionary`中的下标.是实现内存紧凑和性能高效的关键

- `UserDescription` 字符串,自定义的描述信息

- `AutoDescription` 字符串,自动生成的描述信息

### 生成方式

#### 使用C++构造query对象

使用`建造者模式`的API,来进行逻辑表达式的构建:
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

    // 随便写的逻辑,不建议尝试理解它
    // 我用缩进层级表示嵌套层级,每层中的一行定义了一个逻辑表达式
```

`FGameplayTagQuery::Build`简略流程:
1. 写入版本号和用户描述信息,重置关键数据
2. 对`QueryTokenStream`的第0和1号元素写入"版本号"和"是否含有逻辑表达式"的信息
3. 对`逻辑表达式`进行解析,在`QueryTokenStream`中写入表达式类型的枚举,对应`EGameplayTagQueryExprType`,对于"非嵌套表达式"类型,写入它用到的tag数量,将每个tag去重添加到`TagDictionary`,并写入下标;对于"嵌套表达式"类型,递归解析

简单来说,`Build`流程使用`深度优先遍历`将树形结构的逻辑表达式,平铺成了数组.

#### 使用编辑器构造query对象

底层逻辑与使用C++构造一致,只不过由于`FGameplayTagQueryExpression`不是`USTRUCT`,以及当时`InstancedStruct`还没诞生,所以对它用UObject"镜像"了一遍(个人猜测),所以有了`UEditableGameplayTagQueryExpression`以及相关类型,以支持编辑时的表达式嵌套,提供了更好的调试信息,参照`FGameplayTagQuery::BuildFromEditableQuery`

### `FGameplayTagQuery::Matches`, 检验逻辑表达式

使用了辅助类型`FQueryEvaluator`,持有`TagQuery`的不可变引用,记录当前元数据下标和检测是否有读取错误.根据读取到的表达式类型,执行对应的逻辑判定.每次读取元数据都会检测是否存在读取错误.参照`FQueryEvaluator::EvalExpr`

## 结语
感觉`FGameplayTagQuery`的实现比较巧妙,提供了强大的匹配逻辑,适合任意需要Tag匹配功能的需求,不过其中的编辑器逻辑存在代码重复,不够优雅,以及对读取错误的检测并不严谨,后续频繁的检测也就不太必要了.但总体上瑕不掩瑜