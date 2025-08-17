---
title: 我对5.6中 Unreal Animation Framework 的理解
date: 2025-08-05 01:50:00 +0800
categories: [Unreal Engine, Animation]
tags: [gameplay, animation, tutorial]    # TAG names should always be lowercase
lang: zh
media_subpath: /assets/img/UnrealAnimationFramework/
---


约两个月前，Unreal Fest中的巫师4技术演示为我们展示了虚幻引擎下一代动画系统（Unreal Animation Framwork，下文简称UAF。同时把原来的动画蓝图系统简称为ABP），也引起了我强烈的好奇心，感觉是时候好好了解下这个系统了

本文会从程序框架的角度分析此系统
主要介绍系统的构成，各个类型的含义，它们之间的逻辑关系，希望能帮助大家理解和入手这个新动画系统
但不会涉及如动画混合的计算，或是动画重定向等细节

# 简单运行演示

先看一个简单的运行示例：

![UAF vs ABP](UAF_vs_ABP.gif)

视频中是一个简单的分层混合效果，上半身来自`拉弓动画的静帧`，下半身使用`循环的冲刺动画`，

`BlendMask`使用了一个`HierarchyTable`:

![HierarchyTable](HierarchyTable.png)

> 它是一个通用的层级数据容器，这里用作`BlendProfile`
{: .prompt-tip }

从左视图可以看到，两个模型完全重合，动画效果完全一致：

![左视图](LeftView.png)

因为它们两个运行了相同的`UAF的动画图表`：

![UAF Graph LayerBlend](UAF_Graph_LayerBlend.png)

> 左上是`下半身`动画，左下是`上半身`动画，右边是`分层混合`
{: .prompt-tip }

不同的是，左边使用了UAF框架更新上述图表：

![UAF Module PrePhysics](UAF_Module_PrePhysics.png)

> 这里的节点调用可能不是最优的实现，但用于简单的演示足够了
{: .prompt-info }

右边使用了ABP更新上述图表：

![ABP Graph](ABP_Graph.png)

> UAF的动画图表目前可通过这个特殊动画节点集成到动画蓝图
{: .prompt-tip }

## 统一的工作区界面 Workspace

UAF接入了`工作区编辑器`，提供了多个资产的集成视图，
工作区自身也有一个对应资产，类型为`UAF Workspace`，`应该`是用于存放工作区相关的元数据

> `工作区编辑器`模块来自新的实验性插件`Workspace`，它允许多种资产在一个统一的界面中被编辑
> 
> UAF接入了这个功能，指定了UAF相关的资产类型可以在同一个工作区下编辑
> 
> 详见：`UAnimNextWorkspaceSchema`, `IWorkspaceEditorModule::RegisterObjectDocumentType`
{: .prompt-tip }

左上角的`workspace`选项卡，列出了在当前工作区打开的资产：

![UAF Module PrePhysics](UAF_Module_PrePhysics.png)

即`AG_SequencePlayer`, `UAFM_Module`，和工作区自身的资产：`UAFW_Workspace`：

![Workspace](Workspace.png)

## 系统组成

> 以下裸cpp类型基本都位于`UE::AnimNext`名字空间下
{: .prompt-tip }

UAF的逻辑载体目前由两大块组成：`Module`和`AnimationGraph`，都运行于`RigVM`之中，支持`多线程`执行

![AnimNextDataInterface](AnimNextDataInterface.png) 

![AnimNextRigVMAsset](AnimNextRigVMAsset.png)

其中线程间的数据交互，通过`UAnimNextComponent::PublicVariablesProxy`完成

`FAnimNextPublicVariablesProxy`注释中有写到，目前是每帧拷贝脏标记过的数据，将来计划改成`双缓冲数组`(参考`USkinnedMeshComponent::ComponentSpaceTransformsArray`)

> 详见：
> 
> `FAnimNextModuleInstance::CopyProxyVariables`
> 
> `IAnimNextVariableProxyHost::FlipPublicVariablesProxy`
>
> `UAnimNextComponent::SetVariable`
> 
> `UAnimNextComponentWorldSubsystem::Register`
{: .prompt-tip }


# Module

`模块`在这里是使用各个`函数`编写`逻辑业务`的地方，类似ABP里的`蓝图部分`/`UAnimInstance::NativeUpdateAnimation`,`UAnimInstance::NativeThreadSafeUpdateAnimation`，但更加强大，灵活

`FRigUnit_AnimNextModuleEventBase`：

![AnimNextModuleEventBase](AnimNextModuleEventBase.png)

通过基类提供的接口，每个`模块`可以选择是否需要独立的`TickFunction`，要运行在哪个`Tick Group`，是否运行在`游戏线程`等功能

UAF的编译器也会自动生成部分`模块`，比如变量绑定相关的`FRigUnit_AnimNextExecuteBindings_GT` `FRigUnit_AnimNextExecuteBindings_WT`

# AnimationGraph

`动画图表`是`动画逻辑`及其`数据`的集合，类似ABP里的`动画树`

不一样的是，UAF里不再有各式各样的`动画节点`，取而代之的是一个`TraitStack`节点加上各种各样的`Trait`的`组合`

动画图表自身作为`UObject`，也承担着持有图表中`共享数据`的`UObject`对象引用，不让它们被GC的功能

> 详见：
>
> `UAnimNextAnimationGraph::GraphReferencedObjects`
>
> `UAnimNextAnimationGraph::GraphReferencedSoftObjects`
{: .prompt-tip }

另外动画图表可以有`多入口`，而不只是`Root`

## TraitStack和TraitStack节点

`TraitStack`：顾名思义，这是一个由`Trait`组成的`栈`结构，这个栈包含`1`个`base trait`和若干个`additive trait`

而与它对应的`节点`，只是一个正常的`RigUnit`节点（结构体）：

![Trait Stack Node](TraitStackNode.png)

一个TraitStack节点可以包含`一个或多个`TraitStack

在编辑器中，就是上文分层混合图表中的样子

> 节点形式只是为了方便编辑器下的`可视化`，`编译`后会把对应的`TraitStack`序列化到动画图表中，这个`RigUnit`节点并不会被执行
{: .prompt-tip }

## Trait

直译为特性，理解为动画逻辑中可复用的功能，类似ABP中的`动画节点`，但同样的，更为强大，灵活

### FTrait

`FTrait`是所有`Trait`的基类，定义了所需的基础接口，比如获取其`唯一ID`

![FTrait](FTrait.png)

子Trait由`FBaseTrait`或`FAdditiveTrait`加上`ITraitInterface`的`子接口类``组合`而来，

![Base and Additive Trait](BaseAdditiveTrait.png)

![TraitHierarchy](TraitHierarchy.png)


ITraitInterface是所有`trait interface`的基类，

![ITraitInterface](ITraitInterface.png)

它里面只有一个`获取UID`的方法，即每个trait interface`也`有`唯一ID`

目前这两个唯一ID都是对`类名`应用`FNV1a`哈希算法得来，
这个算法的特点是，对于同样的字符组合，无论字符是`普通字符`或是`宽字符`，不会影响哈希结果，产生的哈希值相同，并且简单高效

> 详见：
>
> `FTraitUID::MakeUID`
>
> `FTraitInterfaceUID::MakeUID`
{: .prompt-tip }

Trait对象本身不能有内部状态，即是`无状态的`，因为它们的逻辑会跑在工作线程中（比如同一帧，多个复用同一动画图表的对象，在不同的线程执行）

它的状态数据应该通过`FSharedData`和`FInstanceData`这两个类型别名来声明，UAF系统会在Trait对象`外部`分配好

![SharedData and InstanceData](SharedDataInstanceData.png)

FSharedData是同一动画图表的多个实例可以共享的只读数据，是`USTRUCT`，会序列化`保存`到文件，通常是一些硬编码的配置

FInstanceData是每个动画图表实例中的节点所需的动态数据，是裸CPP结构体

> FSharedData类似于ABP中的`FoldProperty`，
>
> 而InstanceData的机制与`StateTree`中的`FInstanceDataType`/`UInstanceDataType`几乎一致
{: .prompt-tip }

### 代码生成

UAF使用了一些宏，来简单快速的生成框架所需的代码，减少重复劳动

#### Trait Interface的宏

trait interface这边，比较简单，只有两个宏

`DECLARE_ANIM_TRAIT_INTERFACE` 声明并实现`GetInterfaceUID`，返回`编译期常量`：

![DECLARE_ANIM_TRAIT_INTERFACE](DECLARE_ANIM_TRAIT_INTERFACE.png)

![Trait Interface: IEvaluate](IEvaluate.png)

`AUTO_REGISTER_ANIM_TRAIT_INTERFACE` `静态注册`trait interface类的`共享指针`到`全局trait interface注册表`：

![AUTO_REGISTER_ANIM_TRAIT_INTERFACE](AUTO_REGISTER_ANIM_TRAIT_INTERFACE.png)

#### Trait的宏

Trait这边相对就复杂很多：

首先需要在trait类中使用`DECLARE_ANIM_TRAIT`宏，`声明`一些接口的覆写：

![DECLARE_ANIM_TRAIT](DECLARE_ANIM_TRAIT.png)

其包含的几个嵌套宏：

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_BASIC` 声明并实现`GetTraitUID`，返回`编译期常量`；`GetTraitName`返回trait名字；声明`TraitSuper`别名

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_INSTANCING_SUPPORT` trait数据相关的声明

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_INTERFACE_SUPPORT` trait interface获取相关的声明

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_EVENT_SUPPORT` trait事件相关的声明

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_LATENT_PROPERTY_SUPPORT` Latent Property相关的声明（含义见下文`Latent Property`）

然后使用`GENERATE_ANIM_TRAIT_IMPLEMENTATION`宏，定义上述接口，

![GENERATE_ANIM_TRAIT_IMPLEMENTATION](GENERATE_ANIM_TRAIT_IMPLEMENTATION.png)

比较特别的是`InterfaceEnumeratorMacro, RequiredInterfaceEnumeratorMacro, EventEnumeratorMacro`这三个参数，
从名字可以看出它们是`EnumeratorMacro`，是用于枚举的宏，枚举的东西是它们的前缀：`trait interface`，`必须的trait interface`，`trait事件`

`枚举宏`有一个参数，也是宏，这个宏接收`枚举的东西`作为参数，执行相应的操作

以`FBlendTwoWayTrait`为例：

![FBlendTwoWayTrait](FBlendTwoWayTrait.png)

![Macro definition for BlendTwoWayTrait](MacroDefinitionForBlendTwoWayTrait.png)

局部定义的`TRAIT_INTERFACE_ENUMERATOR`宏，枚举了`FBlendTwoWayTrait`实现的所有的trait interface，
并把这些接口传给了`GeneratorMacro`这个参数

结合`GENERATE_ANIM_TRAIT_IMPLEMENTATION`中嵌套的宏：

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT` 定义共享数据和实例数据的内存大小和对其，构造和析构函数

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_LATENT_PROPERTY_MEMORY_LAYOUT` 定义获取Latent Property的内存布局信息的函数

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_IS_PROPERTY_LATENT` 定义判定对应名字的属性是否为Latent Property的函数

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_INTERFACE` 定义获取指定trait interface指针的函数

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_INTERFACES` 定义获取所有实现的trait interface的ID的函数

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_REQUIRED_INTERFACES` 定义获取所有必须的trait interface的ID的函数

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_ON_TRAIT_EVENT` 定义响应所需trait事件回调的函数

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_TRAIT_EVENTS` 定义获取所有响应的trait事件的ID的函数

至此，便可自动生成框架所需的trait定义


`AUTO_REGISTER_ANIM_TRAIT`，与`AUTO_REGISTER_ANIM_TRAIT_INTERFACE`类似，只不过这次注册的是trait

> trait注册没有使用共享指针，而是在`UE::AnimNext::TraitConstructorFunc`回调传入的`DestPtr`地址构造
{: .prompt-tip }

##### Latent Property

`Latent Property`是共享数据中需要实例化的部分，放在`实例化数据`区的后面

对于继承自`FAnimNextTraitSharedData`的共享数据，需要用`GENERATE_TRAIT_LATENT_PROPERTIES`宏，`手动``选择性`注册需要标记为`Latent Property`的属性：

![Latent Property](LATENT_PROPERTIES.png)

这个宏同样使用了枚举宏作为参数，其中它嵌套的宏：

`ANIM_NEXT_IMPL_DEFINE_LATENT_CONSTRUCTOR` 使用`placement new`，`逐个`构造`Latent Property` (内存地址的见后文`FNodeInstance`这节)

> 详见：`FExecutionContext::AllocateNodeInstance` 分配内存，构造`实例化数据`，以及`Latent Property`
>
> `GENERATE_ANIM_TRAIT_IMPLEMENTATION`-`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT`-`ConstructTraitInstance`
>
> `GENERATE_TRAIT_LATENT_PROPERTIES`-`ANIM_NEXT_IMPL_DEFINE_LATENT_CONSTRUCTOR`-`ConstructLatentProperties`
{: .prompt-tip }

`ANIM_NEXT_IMPL_DEFINE_LATENT_DESTRUCTOR` `逐个`析构`Latent Property`

`ANIM_NEXT_IMPL_DEFINE_GET_LATENT_PROPERTY_INDEX` 查询对应偏移量的`Latent Property`下标

> `FAnimNextTraitSharedData::GetLatentPropertyIndex` 注释中提到：如果对应偏移量的`Latent Property`能找到，则返回从1开始数的下标；找不到则返回`Latent Property`的数量，数值小于等于0（需要取负号）
{: .prompt-tip }

`ANIM_NEXT_IMPL_DEFINE_LATENT_GETTER` 为每个`Latent Property`生成从`FTraitBinding`中获取属性值的`getter`函数

> 宏魔法！使用了`constexpr`函数`GetLatentPropertyIndex`获取到`LatentPropertyIndex`，然后从`Binding`中获得`Latent Property`引用
{: .prompt-tip }

#### TraitEvent trait事件

`FAnimNextTraitEvent`是`trait事件`的基类

![FAnimNextTraitEvent](FAnimNextTraitEvent.png)

`DECLARE_ANIM_TRAIT_EVENT` 与trait和trait interface类似，声明和定义了`EventUID`，并且还`额外`支持`IsA`的功能，是一个简单的`RTTI的替代机制`

> 由于FAnimNextTraitEvent是`USTRUCT`，自定义的`IsA`应该不是必要的，这里可能是出于性能或其它考虑
{: .prompt-info }

trait事件类似UI的点击事件，可以被标记为`Handled`，并且可以设定有效时间或者无限时间等

### 全局注册表

#### FTraitRegistry

`trait`对象的全局注册表

使用宏注册trait时，`FTraitRegistry` 优先使用默认分配的`8KB`大小的`StaticTraitBuffer`来存放`trait`，超出后使用`DynamicTraits`存放，是一个`内存局部性`的优化

> 这里有个有趣的小细节，DynamicTraits数组存放的是`uintptr_t`而不是`void*`或者`FTrait*`，即选择了`用整数存放指针`
> 
> 因为用整数，就可以同时存放`数组下标`，以实现后续的`FreeList`机制：
> 
> `DynamicTraitFreeIndexHead`有效时，`DynamicTraits[DynamicTraitFreeIndexHead]`存放的是下一个可复用的数组元素
{: .prompt-tip }

另外还存了几个`Map`用于加速查询

`FTraitRegistry::Register` 用于注册trait到DynamicTraits

#### FTraitInterfaceRegistry

`trait interface`对象的全局注册表

对比下来，`FTraitInterfaceRegistry`就显得朴实无华，只是一个interface ID到智能指针的`map`

## Node 节点

> [粗略的内存布局视图，用drawio绘制](https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1&title=rough%20memory%20layout%20of%20UAF%20types&dark=auto#R%3Cmxfile%3E%3Cdiagram%20id%3D%22kCmcn5g-ny7A861wYZVO%22%20name%3D%22MemoryLayoutOfFNodeInstance%22%3E3ZjZctsgFIafxpfNIMlafOklW6fpJJOZdnpJJCyRIuFByNvT92CjFXmcpkns9MaCHzhI338sjj1wpun6WuBFcscjwgY2itYDZzaw7cD24FMJm70wdLUQCxrtJasWHumWaBFptaARyVsTJedM0kVbDHmWkVC2NCwEX7WnzTlr77rAMTGExxAzU%2F1JI5nox3JRrd8QGiflzhbSIykuJ2shT3DEVw3JuRw4U8G53LfS9ZQwxa7ksl93dWC0ujFBMvmSBQ%2FXkx%2FPIc%2B33vwGTb7%2BGpMV%2FaKjLDEr9ANffQcLb7Nc4iwk%2Bs7lpsQheJFFREVEA2eySqgkjwscqtEV%2BA9aIlMGPQuaOjYRkqwP3rRVoYAUIjwlUmxgil7glKB1%2Brie7q9qM6yScNIwopyHtf9xFbpGBA1N6S%2BI2QaxEtYMSwwjfK5yVGAq4WqdnOAwODeCzhGCHk4VhuwpV5cWTvvkOD3rOM7gI2kOX08zOzlN%2FwU0qzfqh%2BB0DZyAkMGuk4guoRmr5sXFRanCJo0BAyigkW1qmNE4g3YIiIgAQQGkcOCM9UBKo0gt78XfNugt8tnvOOC7hgNejwHOexng9RjQgUqyaKyOduhxmNfm2yYE3wQhO5N32hVl5QoI1%2BjNoTnljIvdXg5CLiLz3SrBf5PWiDWbTKuRsjqwK19IZJQWR11pUHd7qJeaIAxLumyH77NC73DPaSYPn6pe0HEz54UIiV7VrCk6gYb%2BkUCAOibSCASO4E1j2kJNyI3cqcC8Pp38c0unACPUn06uj9AnSJ6jnr80eSwUnHv2BEb2jENZYLUuJSnfUceM8RCAcvM4%2FZe3vyA53eKnXSiVFvohIa47GbgzFauQPN%2F%2FYLKMjMp4Rjrpp6U3ODScUds43%2B45tt%2Fr1LiWd7cF3TyMnu%2FYcroc%2B1s66vkd8w1Lssu2e8EX8LDq9ueCp8oh2LQugzqeHSl63vwADuwT1ue9LM36%2FLOwHHnnxtKszv%2FvcnLUzeb3KyehW%2F95sn%2Fh1%2F9AOZd%2FAA%3D%3D%3C%2Fdiagram%3E%3Cdiagram%20name%3D%22MemoryLayoutOfFNodeDescription%22%20id%3D%22hcCWD1qce99PYXvevLv8%22%3E5Zhdb9sgFIZ%2FTaTuopWJ68S9zEc%2Fpm1VpW5qtTtiSMyKTYRJk%2FTX75BgxzakTdc0sbSrwAsc4DkHc0jLHySLa4mn8Q9BKG%2B1PbJo%2BcNWu41QiOBHK8u10gmMMJGMmE4b4Z69UCN6Rp0xQrNKRyUEV2xaFSORpjRSFQ1LKebVbmPBq7NO8YRawn2Eua0%2BMKLitRoG3ka%2FoWwS5zMjz7QkOO9shCzGRMxLkn%2FZ8gdSCLUuJYsB5RpezmU97mpLa7EwSVO1ywCspB8%2B4Oyp8%2B3P6a%2FH8HF4wU%2BNlWfMZ2bDV7fgwyHNIsmmionULF4tcyJSzFJCtVGv5ffnMVP0fooj3TqHGAAtVgmHGoKivch8RioVXZQks%2BhrKhKq5BK6mNZ2xwA0EeSHpj7f%2BAPlkOOSL%2FJx2ITApDC9oQQFA%2Bod0NoWtPsYdkiGWGHQxVgHqcRMwS%2FszTsZ4Yx%2BOTrIc69pIP13gNQdTzAhTLHnBsBEXvA2zfCQMM%2FfARNOdYfDgvojCaWJLjUkSBHagWvxkT0I2MACm9Mj7DnHd3Z2lqswSanBIgogVBUb5mySQjkCbFSCoHExuIN6piFhhOjhTv5VD%2B3FBajmgovAckHH4QH%2FszzQcXigRpWmpKeve6gJ6FcFXEWUKSxVrfNKu2I8HwHmSrUxFAeCC7may%2Ffg7NPxapQUT7TSgob9QdGSZwzt1xxDSSUDsd1Soh44qOeapBzrb2M1BXK4wsxwJ1iqtl%2BzgVfzZiZmMqJmVDnPqBkqDuw2Q4B6QpVlCDyCl6VuU90hs2KnAPPv4dRtWjiF2PPc4RR0vVdPdUOC502f7xo8yA%2BaHj2hFT29SM2wHpfQRKyoY85FhJ3p80c%2B%2F5Jm7AWPVqZ0WJhNgt2g3wqG2tZMiWz9iEJWRKUipbXwM9Iebg2%2Fll0G5457%2B7NujZ8AoncbdXt9X4y%2BPt0p%2Fhu53jbfsYId3kkx1YhpdkMxAeR7SHosUA6cW9kVrF7LeVzHdx85j5OdnZlX2S1vcKpjEqY%2BOr1uPV05Or2dMsZtQEc75pGHxhyGTcPsuscbnph%2FyAOo%2BFYeITF3esC%2BCw%2F5gP8gzO7xHvBOmBf%2FWzh7h3tnQnXzT%2Bs6E9z8X%2B1f%2FgU%3D%3C%2Fdiagram%3E%3Cdiagram%20id%3D%22oWyUg2EomUwlymL4ElC6%22%20name%3D%22MemoryLayoutOfFNodeTemplate%22%3E1ZfdctowEIWfxpfNyBIGchlI0jSTtJ1JJr1WrcXWVLYYWQTI03eN5V%2BRIW3TQm8Y%2BWh3Zb6zsHbA5tnmo%2BHL9F4LUAElYhOwy4DS8%2BkIP0thWwnRiFZCYqSopLAVHuQLOJE4dSUFFL1Aq7WyctkXY53nENuexo3R637YQqv%2BqUuegCc8xFz56jcpbFqp04i0%2Bg3IJK1PDonbyXgd7IQi5UKvOxK7CtjcaG2rVbaZgyrZ1VyqvOtXdpsbM5DbtyTcZk%2FC6Pnjl7vtHbt9mt7cq08fXJVnrlbuC19%2FRgsfIVsqbsHdud3WOIxe5QLKiiRgs3UqLTwseVzurtF%2F1FKbKbwKcelqg7GwefWmwwYFthDoDKzZYohLoDVO1z6jui%2FWrRlhTTjtGDF2Gnf%2BJ03pFhEuHKVfIEZ9Yo%2BGS9siI%2BHRsTF2GFuD9p9gY4ex0aNji0anhm10GFt%2BdGyT8alhizxsAR0rPHUm5DMuk3J5dnZWq3hIZ8MDimhsnxpXMslxHSMiMCiUACWOjQu3kUkhyvS9%2BPsGvYcDdOAAjTwHxnsMYH%2FLgPEeAwZQIRcX5YDGK41xfb59QoXlxg6Cd9q1VHUGlutcLXA510qb3VmMkIjAYpdl9A%2Fo7YSXs3mzU8942vgCwntAOOhKh3q0h3qtGcCfsHzul99nhTvhq5Z48KuzMRq6WeiVicFldZ8MBoUYPVAIUSdgvULoCN92wpZlQOH1TgPm99tpcmrtNOWE7G%2BnaELIf9A8Bz1%2Fa%2FOcj0%2B9eaZe81zEdsXLvAwyvYPOldIx8tT%2BNP2TP38DhXzh33elyq5wXxLrRrMguixrrawuqree0GuoXOcw6D4nvcPMoIOHHWxcb2aE7zQ08LJ90al8bd8W2dVP%3C%2Fdiagram%3E%3C%2Fmxfile%3E)

这节介绍一些跟节点相关的关键类型

### FNodeTemplate

一个`FNodeTemplate`就是一组`trait`的排列组合，可以有多组`base + additive`trait，所有动画图表共享此模板对象

![FNodeTemplate](FNodeTemplate.png)

组成节点模板的`trait`们，以`FTraitTemplate`的对象形式存放在`FNodeTemplate`对象末尾的`连续内存`处

![FNodeTemplate::GetTraits](FNodeTemplate_GetTraits.png)

> `FNodeTemplateBuilder::BuildNodeTemplate` 在一个`TArray<uint8>`的连续缓冲区中构造FNodeTemplate对象及其中包含的FTraitTemplate
{: .prompt-tip }

从它可以获取`UID`，`trait数组首地址`，trait数量等信息

- `FNodeTemplate::NodeSharedDataSize` 所有trait的`共享数据`（加上其中base trait的`FLatentPropertiesHeader`），对齐之后的大小
- `FNodeTemplate::NodeInstanceDataSize` 所有trait的`实例数据`，对齐之后的大小

![rough memory layout of FNodeTemplate](MemoryLayoutOfFNodeTemplate.png)

#### FTraitTemplate

`FTraitTemplate`就是节点模板里的`trait`，用它除了可以获取到关于trait的`UID`，trait类型，共享/实例数据，子trait数量，`Latent Property`数量等基本信息外，还可以获取到`共享数据的偏移量`，`共享的latent property的数组指针偏移量`以及`实例数据的偏移量`

![FTraitTemplate](FTraitTemplate.png)

> 详见：`FNodeTemplate::Finalize`
{: .prompt-tip }

> 我觉得`FTraitTemplate::GetTraitDescription`这两个成员函数命名有点容易混淆，更好理解的命名应该是`GetTraitSharedData`，用于获取trait的共享数据指针，可能是笔误或者是改名了
{: .prompt-tip }

### FNodeDescription

在一个`动画图表`内唯一的只读数据，对象本身虽然是8字节大小，但分配内存时，加上节点上trait们的共享数据的大小，`是一个用法上大小变化的对象`

![rough memory layout of FNodeDescription](MemoryLayoutOfFNodeDescription.png)

> 详见：
>
> `FTraitReader::ReadGraphSharedData`
>
> 和一些其它细节：
> `FNodeDescription::Serialize`
> `FTrait::SerializeTraitSharedData`
> `FTraitWriter::WriteNode`
> `FTrait::SaveTraitSharedData`
> 
> 读取之后存放在`UAnimNextAnimationGraph::SharedDataBuffer`，使用上参考`FExecutionContext::GetNodeDescription`
{: .prompt-tip }

- `FNodeDescription::TemplateHandle` 用于从`FNodeTemplateRegistry`获取`FNodeTemplate`对象实例
- `FNodeDescription::NodeInstanceDataSize` 包含`实例化数据`的大小，加上所有`Latent Property`的`总大小`

> 注意`FNodeDescription::NodeInstanceDataSize`与`FNodeTemplate::NodeInstanceDataSize`的区别
> 
> 另外，`FNodeDescription`跟`FNodeTemplate`是`多对一`的，从它们的含义上可以理解
{: .prompt-tip }


### FNodeInstance

节点的实例化数据，运行时动态创建，自身16字节大小，分配内存时，加上了trait们的实例化数据大小以及它们的`Latent Property`大小，也`是一个用法上大小变化的对象`；内置引用计数

![rough memory layout of FNodeInstance](MemoryLayoutOfFNodeInstance.png)

> 用法上可以参考：`FAnimationAnimNextRuntimeTest_TraitSerialization::RunTest`，`FExecutionContext::AllocateNodeInstance`
{: .prompt-tip }

### FNodeTemplateRegistry

`FNodeTemplate`对象的全局注册表，确保了所有FNodeTemplate`内存连续`

### FTraitStackBinding

描述`一组trait`（1个base trait及其children/additive trait们）所需要的数据，用于查询trait或者`trait interface`

> 详见：`FTraitStackBinding::FTraitStackBinding`，特别是最后几行
{: .prompt-tip }

例如，`FTraitStackBinding::GetInterfaceImpl`：从`trait栈`上尝试找到实现了指定`InterfaceUID`的trait，并返回该trait的`binding`

### FTraitBinding

描述一组trait中的`特定trait`的数据，可以查询当前trait是否实现了指定`trait interface`

### TTraitBinding

强类型/类型安全的`FTraitBinding`

## FExecutionContext

由于trait是无状态的，执行时的`动态数据`/`实例数据`需要一个对象来承载，它就是`FExecutionContext`，执行上下文对象

它用于`绑定`到一个图表实例，并为节点提供统一的`trait查询`接口


图表的`Update`和`Evaluate`流程封装在一个RigVM节点的执行函数中：`FRigUnit_AnimNextRunAnimationGraph_v2_Execute()`

### FUpdateTraversalContext

`更新`图表时，用到的上下文子类对象

内部使用了在`MemStack`上分配的栈（`LIFO`）实现`trait树`的`深度优先遍历`，不再是ABP中的`递归`方式

> 详见：`UE::AnimNext::UpdateGraph`，每个trait在while循环中会被执行两次，分别对应`IUpdate::PreUpdate`和`IUpdate::PostUpdate`
> 
> `IUpdate::OnBecomeRelevant`也是在这里调用
{: .prompt-tip }

### FEvaluateTraversalContext

`评估`图表时，用到的上下文子类对象

内部的`FEvaluationProgram`用于存放遍历图表时，各个节点需要执行的`FAnimNextEvaluationTask`
随后调用`FEvaluationProgram::Execute`在`FEvaluationVMStack`上执行各评估任务

> 详见：`UE::AnimNext::EvaluateGraph`，执行流程与`UpdateGraph`类似
{: .prompt-tip }

#### FAnimNextEvaluationTask

`FAnimNextEvaluationTask`是可在trait之间复用的逻辑对象，代表运行于`评估虚拟机`上的微指令，通过虚拟机的内部状态（也是`栈`）可以同时处理输入和输出

# 系统模块

UAF由多个模块组成：

![UAF On 5.6](UAFOn5.6.png)

在`ue5-main`分支上，系列模块的名字前缀从`AnimNext`改成了`UAF`

![UAF On ue5-main](UAFOnMain.png)

其中
`UAF`/`AnimNext` ：提供核心的动画工具函数，定义基类接口等。如：`UE::AnimNext::FDecompressionTools::GetAnimationPose`

`UAFAnimGraph`/`AnimNextAnimGraph`：实现基于RigVM的动画图表相关功能

这两个模块也是本文主要讨论的范围

其他模块则是为`UAF`引入`其它模块/系统`的功能，比如引入`StateTree`，`PoseSearch`等

## SoA （struct of array）

在`TransformArrayOperations.h`中的每个工具函数都有`AoS`和`SoA`两个版本，并且代码中优先使用了`AoS`，由此可见在数据结构设计上，UAF会`更面向数据`一些

# 结论

UAF是一个重新设计的，面向数据的，倾向于组合范式的，高性能，灵活，简洁，易拓展的动画框架

完全抛弃了ABP的框架，拥抱RigVM

因其特性集尚未成熟，而处于实验性阶段

# 杂项

## 代码里写动画蓝图

参考`UAFTestSuite`/`AnimNextTestSuite`以及`UAFAnimGraphTestSuite`/`AnimNextAnimGraphTestSuite`模块中的`测试用例`代码

## 在UAF中播放“蒙太奇”

由于本文篇幅已经较长，我觉得还是不要再详细阐述了，

相关节点是`UInjectionCallbackProxy`，`UPlayAnimCallbackProxy`

相关代码在`UE::AnimNext::FInjectionUtils::Inject`

## 官方FAQ链接

[Unreal Animation Framework (UAF，AnimNext) FAQ](https://dev.epicgames.com/community/learning/knowledge-base/nWWx/unreal-engine-unreal-animation-framework-uaf-faq)