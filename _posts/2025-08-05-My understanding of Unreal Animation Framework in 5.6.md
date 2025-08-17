---
title: My Understanding of the Unreal Animation Framework in 5.6
date: 2025-08-05 01:50:00 +0800
categories: [Unreal Engine, Animation]
tags: [gameplay, animation, tutorial]    # TAG names should always be lowercase
lang: en
media_subpath: /assets/img/UnrealAnimationFramework/
---


About two months ago, the technical demonstration of The Witcher 4 at Unreal Fest showcased the next-generation animation system of the Unreal Engine (Unreal Animation Framework, hereafter referred to as UAF, while the previous animation blueprint system will be referred to as ABP). This sparked my strong curiosity, and I felt it was time to dive into understanding this system.

This article will analyze the system from a architecture perspective, primarily introducing its components, the meanings of various types, and their logical relationships. I hope this will help everyone grasp and get started with this new animation system, but it will not cover details such as animation blending calculations or animation retargeting.

# Simple demo

Let's start with a simple demo:

![UAF vs ABP](UAF_vs_ABP.gif)

The video shows a simple layered blending effect, with the upper body coming from the `static frame of the bow drawing animation` and the lower body using a `looping sprint animation`.

`BlendMask` utilizes a `HierarchyTable`:

![HierarchyTable](HierarchyTable.png)

> It is a general-purpose hierarchical data container, used here as `BlendProfile`
{: .prompt-tip }

From the left view, it's clear that the two models completely overlap, and the animation effects are identical:

![Left View](LeftView.png)

This is because both are running the same `animation graph of UAF`:

![UAF Graph LayerBlend](UAF_Graph_LayerBlend.png)

> The upper left is the `lower body` animation, the lower left is the `upper body` animation, and the right is the `layered blending`
{: .prompt-tip }

The difference is that the animation graph on the left is updated using the UAF framework:

![UAF Module PrePhysics](UAF_Module_PrePhysics.png)

> These nodes here may not be the optimal implementation, but they are sufficient for simple demonstration
{: .prompt-info }

while on the right side, the animation graph is updated using ABP :

![ABP Graph](ABP_Graph.png)

> animation graph of UAF can be integrated into animation blueprints through this special animation node
{: .prompt-tip }

## Unified Workspace Interface

UAF has integrated the `Workspace Editor`, providing a unified view of multiple assets.  
The workspace itself also has a corresponding asset, classified as `UAF Workspace`, which `should` be used to store metadata related to the workspace.

> The `Workspace Editor` module comes from the new experimental plugin `Workspace`, which allows multiple assets to be edited in a unified interface.
> 
> UAF has integrated this feature, which specifies that UAF-related asset types can be edited in the same workspace.
> 
> See：`UAnimNextWorkspaceSchema`, `IWorkspaceEditorModule::RegisterObjectDocumentType`
{: .prompt-tip }

The `workspace` tab in the upper left corner lists the assets opened in the current workspace:

![UAF Module PrePhysics](UAF_Module_PrePhysics.png)

Namely, `AG_SequencePlayer`, `UAFM_Module`, and the workspace's own asset: `UAFW_Workspace`:

![Workspace](Workspace.png)

## System Composition

> These raw cpp types bellow are basically located in the `UE::AnimNext` namespace
{: .prompt-tip }

The logical carriers of UAF currently consist of two main components: `Module` and `AnimationGraph`, both running within `RigVM`, supporting `multithreaded` execution.

![AnimNextDataInterface](AnimNextDataInterface.png) 

![AnimNextRigVMAsset](AnimNextRigVMAsset.png)

Data exchange between threads is accomplished through `UAnimNextComponent::PublicVariablesProxy`.

The comment in `FAnimNextPublicVariablesProxy` mentions that currently, it copies dirty-marked data every frame, with plans to change it to a `double-buffered array` in the future (refer to `USkinnedMeshComponent::ComponentSpaceTransformsArray`).

> See：
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

The `module` here is where various `functions` are used to write `logical business`, similar to the `blueprint section` in ABP / `UAnimInstance::NativeUpdateAnimation`, `UAnimInstance::NativeThreadSafeUpdateAnimation`, but more powerful and flexible.

`FRigUnit_AnimNextModuleEventBase`：

![AnimNextModuleEventBase](AnimNextModuleEventBase.png)

Through the interface provided by the base class, each `module` can choose whether it needs an independent `TickFunction`, which `Tick Group` to run in, whether to operate on the `game thread`, and other functionalities.

The UAF compiler will also automatically generate some `modules`, such as variable binding-related `FRigUnit_AnimNextExecuteBindings_GT` and `FRigUnit_AnimNextExecuteBindings_WT`.

# AnimationGraph

The `animation graph` is a collection of `animation logic` and its `data`, similar to the `animation tree` in ABP.

The difference is that in UAF, there are no longer various `animation nodes`; instead, there is a `TraitStack` node combined with various `Trait` combinations.

The animation graph itself acts as a `UObject`, also holding references to `UObject` references that referenced by `shared data` within the graph, preventing them from being garbage collected.

> See：
>
> `UAnimNextAnimationGraph::GraphReferencedObjects`
>
> `UAnimNextAnimationGraph::GraphReferencedSoftObjects`
{: .prompt-tip }

Additionally, animated charts can have `multiple entries`, not just `Root`

## TraitStack and TraitStack Node

`TraitStack`: As the name suggests, this is a `stack` structure composed of `Traits`, which includes `1` `base trait` and several `additive traits`.

The corresponding `node` is simply a standard `RigUnit` node (struct):

![Trait Stack Node](TraitStackNode.png)

A TraitStack node can contain `one or more` TraitStacks.

In the editor, it appears as shown in the layered blending animation graph above.

> The node form is just for the convenience of `visualization` in the editor. After `compiling`, the corresponding `TraitStack` will be serialized into the animation graph. This `RigUnit` node will not be executed
{: .prompt-tip }

## Trait

"trait" or feature, it refers to reusable functionalities in animation logic, similar to `animation nodes` in ABP, but again, more powerful and flexible.

### FTrait

`FTrait` is the base class for all `Traits`, defining the necessary basic interface, such as obtaining its `unique ID`.

![FTrait](FTrait.png)

Derived traits are composed of `FBaseTrait` or `FAdditiveTrait` along with the `derived interface class` of `ITraitInterface`.

![Base and Additive Trait](BaseAdditiveTrait.png)

![TraitHierarchy](TraitHierarchy.png)

`ITraitInterface` is the base class for all `trait interfaces`.

![ITraitInterface](ITraitInterface.png)

It contains only one method for `getting UID`, meaning each `trait interface` also has a `unique ID`.

Currently, these two unique IDs are derived by applying the `FNV1a` hash algorithm to the `class name`. This algorithm is characterized by the fact that for the same character combination, whether the characters are `normal characters` or `wide characters`, it does not affect the hash result, producing the same hash value, and is simple and efficient.

> See：
>
> `FTraitUID::MakeUID`
>
> `FTraitInterfaceUID::MakeUID`
{: .prompt-tip }

Trait objects themselves cannot have internal state, meaning they are `stateless`, as their logic runs in worker threads (for example, multiple objects reusing the same animation graph within the same frame execute in different threads).

Their state data should be declared using the type aliases `FSharedData` and `FInstanceData`, which the UAF system will allocate externally for the Trait objects.

![SharedData and InstanceData](SharedDataInstanceData.png)

FSharedData is read-only data that can be shared among multiple instances of the same animation graph; it is a `USTRUCT` that will serialize and `save` to a file, typically consisting of some hardcoded configurations. 

FInstanceData contains the dynamic/instanced data required by the nodes in each animation graph instance and is a raw CPP structure.

> FSharedData is similar to `FoldProperty` in ABP,
>
> while the mechanism of InstanceData is almost the same as `FInstanceDataType`/`UInstanceDataType` in `StateTree`
{: .prompt-tip }

### Code Generation

UAF uses several macros to quickly and easily generate the code required for the framework, reducing repetitive work.

#### Macros for Trait Interface

The trait interface part is relatively simple, with only two macros.

`DECLARE_ANIM_TRAIT_INTERFACE` declares and implements `GetInterfaceUID`, returning a `compile-time constant`:

![DECLARE_ANIM_TRAIT_INTERFACE](DECLARE_ANIM_TRAIT_INTERFACE.png)

![Trait Interface: IEvaluate](IEvaluate.png)

`AUTO_REGISTER_ANIM_TRAIT_INTERFACE` `statically registers` the shared pointer of the trait interface class to the `global trait interface registry`:

![AUTO_REGISTER_ANIM_TRAIT_INTERFACE](AUTO_REGISTER_ANIM_TRAIT_INTERFACE.png)

#### Macros for Trait

The trait section is considerably more complex:

First, you need to use the `DECLARE_ANIM_TRAIT` macro within the trait class to `declare` some virtual function overrides:

![DECLARE_ANIM_TRAIT](DECLARE_ANIM_TRAIT.png)

This includes several nested macros:

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_BASIC` declares and implements `GetTraitUID`, returning a `compile-time constant`; `GetTraitName` returns the trait name; declares an alias for `TraitSuper`.

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_INSTANCING_SUPPORT` add declarations related to trait data.

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_INTERFACE_SUPPORT` add declarations for accessing the trait interface.

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_EVENT_SUPPORT` add declarations related to trait events.

`ANIM_NEXT_IMPL_DECLARE_ANIM_TRAIT_LATENT_PROPERTY_SUPPORT` add declarations related to Latent Property (see below for the meaning of `Latent Property`).

Then, use the `GENERATE_ANIM_TRAIT_IMPLEMENTATION` macro to define the above interfaces.

![GENERATE_ANIM_TRAIT_IMPLEMENTATION](GENERATE_ANIM_TRAIT_IMPLEMENTATION.png)

Notably, the parameters `InterfaceEnumeratorMacro, RequiredInterfaceEnumeratorMacro, EventEnumeratorMacro` are all `EnumeratorMacros`, which are macros used for enumeration, with their prefixes indicating what they enumerate: `trait interface`, `required trait interface`, and `trait events`.

The `enumeration macro` has one parameter, which is also a macro that takes the `enumerated item` as an argument and performs the corresponding operations.

Taking `FBlendTwoWayTrait` as an example:

![FBlendTwoWayTrait](FBlendTwoWayTrait.png)

![Macro definition for BlendTwoWayTrait](MacroDefinitionForBlendTwoWayTrait.png)

The locally defined `TRAIT_INTERFACE_ENUMERATOR` macro enumerates all the trait interfaces implemented by `FBlendTwoWayTrait` and passes these interfaces to the `GeneratorMacro` parameter.

Combining with the nested macros in `GENERATE_ANIM_TRAIT_IMPLEMENTATION`:

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT` defines the memory size and alignment for shared and instance data, as well as the constructor and destructor.

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_LATENT_PROPERTY_MEMORY_LAYOUT` defines the function to retrieve the memory layout information for Latent Property.

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_IS_PROPERTY_LATENT` defines the function to determine whether the property with the corresponding name is a Latent Property.

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_INTERFACE` defines the function to retrieve a pointer to the specified trait interface.

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_INTERFACES` defines the function to retrieve the IDs of all implemented trait interfaces.

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_REQUIRED_INTERFACES` defines the function to retrieve the IDs of all required trait interfaces.

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_ON_TRAIT_EVENT` defines the function to respond to the required trait event callbacks.

`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT_GET_TRAIT_EVENTS` defines the function to retrieve the IDs of all responsive trait events.

At this point, the necessary trait definitions for the framework has been automatically generated.

`AUTO_REGISTER_ANIM_TRAIT`, similar to `AUTO_REGISTER_ANIM_TRAIT_INTERFACE`, registers the trait this time.

> Trait registration does not use shared pointers, but is constructed from the callback of `UE::AnimNext::TraitConstructorFunc` with the `DestPtr` address passed in.
{: .prompt-tip }

##### Latent Property

`Latent Property` is the part of shared data that needs to be instantiated, placed after the `FInstanceData` section.

For shared data that inherits from `FAnimNextTraitSharedData`, you need to use the `GENERATE_TRAIT_LATENT_PROPERTIES` macro to `manually`、`selectively` register the properties that should be marked as `Latent Property`:

![Latent Property](LATENT_PROPERTIES.png)

This macro also uses an enumeration macro as a parameter, within which it nests the macro:

`ANIM_NEXT_IMPL_DEFINE_LATENT_CONSTRUCTOR` uses `placement new` to `individually` construct `Latent Property` (the memory address is discussed in the following section on `FNodeInstance`).

> See: `FExecutionContext::AllocateNodeInstance` for how it allocates memory, constructs `instance data`, and `Latent Property`
>
> `GENERATE_ANIM_TRAIT_IMPLEMENTATION`-`ANIM_NEXT_IMPL_DEFINE_ANIM_TRAIT`-`ConstructTraitInstance`
>
> `GENERATE_TRAIT_LATENT_PROPERTIES`-`ANIM_NEXT_IMPL_DEFINE_LATENT_CONSTRUCTOR`-`ConstructLatentProperties`
{: .prompt-tip }

`ANIM_NEXT_IMPL_DEFINE_LATENT_DESTRUCTOR` `Destruct` `Latent Property` one by one

`ANIM_NEXT_IMPL_DEFINE_GET_LATENT_PROPERTY_INDEX` Query the index of the `Latent Property` for the corresponding offset

> `FAnimNextTraitSharedData::GetLatentPropertyIndex` comments mentioned: If the `Latent Property` corresponding to the offset can be found, it returns the index starting from 1; if not found, it returns the number of `Latent Property`, which is less than or equal to 0 (needs to be negative)
{: .prompt-tip }

`ANIM_NEXT_IMPL_DEFINE_LATENT_GETTER` generates a `getter` function for retrieving property values from `FTraitBinding` for each `Latent Property`

> Magic of Macro ! It uses the `constexpr` function `GetLatentPropertyIndex` to get the `LatentPropertyIndex`, and then get the `Latent Property` reference from the `Binding`.
{: .prompt-tip }

#### TraitEvent Trait Events

`FAnimNextTraitEvent` is the base class for `trait events`.

![FAnimNextTraitEvent](FAnimNextTraitEvent.png)

`DECLARE_ANIM_TRAIT_EVENT`, similar to traits and trait interfaces, declares and defines `EventUID`, and additionally supports the `IsA` functionality, serving as a simple `alternative mechanism for RTTI`.

> Since FAnimNextTraitEvent is `USTRUCT`, the `IsA` should not be necessary. This may be for performance or other considerations.
{: .prompt-info }

Trait events are similar to UI click events and can be marked as `Handled`, with the option to set a valid duration or an infinite duration, among other settings.

### Global Registry

#### FTraitRegistry

The global registry for `trait` objects.

When registering traits using macros, `FTraitRegistry` prioritizes the default allocated `8KB` size `StaticTraitBuffer` to store `traits`. If this limit is exceeded, it uses `DynamicTraits` for storing new trait, which is an optimization for `memory locality`.

> There is an interesting little detail here. The DynamicTraits array stores `uintptr_t` instead of `void*` or `FTrait*`, which means `using integers to store pointers`.
> 
> Because integers are used, `index of array` can be stored at the same time to implement the subsequent `FreeList` mechanism:
> 
> When `DynamicTraitFreeIndexHead` is valid, `DynamicTraits[DynamicTraitFreeIndexHead]` stores the next reusable array element
{: .prompt-tip }

Additionally, several `Map`s are stored to speed up queries.

`FTraitRegistry::Register` is used to register traits to DynamicTraits.

#### FTraitInterfaceRegistry

The global registry for `trait interface` objects.

In comparison, `FTraitInterfaceRegistry` is quite straightforward, simply a map from interface IDs to smart pointers.

## Node

> [rough memory layout，drawing with drawio](https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1&title=rough%20memory%20layout%20of%20UAF%20types&dark=auto#R%3Cmxfile%3E%3Cdiagram%20id%3D%22kCmcn5g-ny7A861wYZVO%22%20name%3D%22MemoryLayoutOfFNodeInstance%22%3E3ZjZctsgFIafxpfNIMlafOklW6fpJJOZdnpJJCyRIuFByNvT92CjFXmcpkns9MaCHzhI338sjj1wpun6WuBFcscjwgY2itYDZzaw7cD24FMJm70wdLUQCxrtJasWHumWaBFptaARyVsTJedM0kVbDHmWkVC2NCwEX7WnzTlr77rAMTGExxAzU%2F1JI5nox3JRrd8QGiflzhbSIykuJ2shT3DEVw3JuRw4U8G53LfS9ZQwxa7ksl93dWC0ujFBMvmSBQ%2FXkx%2FPIc%2B33vwGTb7%2BGpMV%2FaKjLDEr9ANffQcLb7Nc4iwk%2Bs7lpsQheJFFREVEA2eySqgkjwscqtEV%2BA9aIlMGPQuaOjYRkqwP3rRVoYAUIjwlUmxgil7glKB1%2Brie7q9qM6yScNIwopyHtf9xFbpGBA1N6S%2BI2QaxEtYMSwwjfK5yVGAq4WqdnOAwODeCzhGCHk4VhuwpV5cWTvvkOD3rOM7gI2kOX08zOzlN%2FwU0qzfqh%2BB0DZyAkMGuk4guoRmr5sXFRanCJo0BAyigkW1qmNE4g3YIiIgAQQGkcOCM9UBKo0gt78XfNugt8tnvOOC7hgNejwHOexng9RjQgUqyaKyOduhxmNfm2yYE3wQhO5N32hVl5QoI1%2BjNoTnljIvdXg5CLiLz3SrBf5PWiDWbTKuRsjqwK19IZJQWR11pUHd7qJeaIAxLumyH77NC73DPaSYPn6pe0HEz54UIiV7VrCk6gYb%2BkUCAOibSCASO4E1j2kJNyI3cqcC8Pp38c0unACPUn06uj9AnSJ6jnr80eSwUnHv2BEb2jENZYLUuJSnfUceM8RCAcvM4%2FZe3vyA53eKnXSiVFvohIa47GbgzFauQPN%2F%2FYLKMjMp4Rjrpp6U3ODScUds43%2B45tt%2Fr1LiWd7cF3TyMnu%2FYcroc%2B1s66vkd8w1Lssu2e8EX8LDq9ueCp8oh2LQugzqeHSl63vwADuwT1ue9LM36%2FLOwHHnnxtKszv%2FvcnLUzeb3KyehW%2F95sn%2Fh1%2F9AOZd%2FAA%3D%3D%3C%2Fdiagram%3E%3Cdiagram%20name%3D%22MemoryLayoutOfFNodeDescription%22%20id%3D%22hcCWD1qce99PYXvevLv8%22%3E5Zhdb9sgFIZ%2FTaTuopWJ68S9zEc%2Fpm1VpW5qtTtiSMyKTYRJk%2FTX75BgxzakTdc0sbSrwAsc4DkHc0jLHySLa4mn8Q9BKG%2B1PbJo%2BcNWu41QiOBHK8u10gmMMJGMmE4b4Z69UCN6Rp0xQrNKRyUEV2xaFSORpjRSFQ1LKebVbmPBq7NO8YRawn2Eua0%2BMKLitRoG3ka%2FoWwS5zMjz7QkOO9shCzGRMxLkn%2FZ8gdSCLUuJYsB5RpezmU97mpLa7EwSVO1ywCspB8%2B4Oyp8%2B3P6a%2FH8HF4wU%2BNlWfMZ2bDV7fgwyHNIsmmionULF4tcyJSzFJCtVGv5ffnMVP0fooj3TqHGAAtVgmHGoKivch8RioVXZQks%2BhrKhKq5BK6mNZ2xwA0EeSHpj7f%2BAPlkOOSL%2FJx2ITApDC9oQQFA%2Bod0NoWtPsYdkiGWGHQxVgHqcRMwS%2FszTsZ4Yx%2BOTrIc69pIP13gNQdTzAhTLHnBsBEXvA2zfCQMM%2FfARNOdYfDgvojCaWJLjUkSBHagWvxkT0I2MACm9Mj7DnHd3Z2lqswSanBIgogVBUb5mySQjkCbFSCoHExuIN6piFhhOjhTv5VD%2B3FBajmgovAckHH4QH%2FszzQcXigRpWmpKeve6gJ6FcFXEWUKSxVrfNKu2I8HwHmSrUxFAeCC7may%2Ffg7NPxapQUT7TSgob9QdGSZwzt1xxDSSUDsd1Soh44qOeapBzrb2M1BXK4wsxwJ1iqtl%2BzgVfzZiZmMqJmVDnPqBkqDuw2Q4B6QpVlCDyCl6VuU90hs2KnAPPv4dRtWjiF2PPc4RR0vVdPdUOC502f7xo8yA%2BaHj2hFT29SM2wHpfQRKyoY85FhJ3p80c%2B%2F5Jm7AWPVqZ0WJhNgt2g3wqG2tZMiWz9iEJWRKUipbXwM9Iebg2%2Fll0G5457%2B7NujZ8AoncbdXt9X4y%2BPt0p%2Fhu53jbfsYId3kkx1YhpdkMxAeR7SHosUA6cW9kVrF7LeVzHdx85j5OdnZlX2S1vcKpjEqY%2BOr1uPV05Or2dMsZtQEc75pGHxhyGTcPsuscbnph%2FyAOo%2BFYeITF3esC%2BCw%2F5gP8gzO7xHvBOmBf%2FWzh7h3tnQnXzT%2Bs6E9z8X%2B1f%2FgU%3D%3C%2Fdiagram%3E%3Cdiagram%20id%3D%22oWyUg2EomUwlymL4ElC6%22%20name%3D%22MemoryLayoutOfFNodeTemplate%22%3E1ZfdctowEIWfxpfNyBIGchlI0jSTtJ1JJr1WrcXWVLYYWQTI03eN5V%2BRIW3TQm8Y%2BWh3Zb6zsHbA5tnmo%2BHL9F4LUAElYhOwy4DS8%2BkIP0thWwnRiFZCYqSopLAVHuQLOJE4dSUFFL1Aq7WyctkXY53nENuexo3R637YQqv%2BqUuegCc8xFz56jcpbFqp04i0%2Bg3IJK1PDonbyXgd7IQi5UKvOxK7CtjcaG2rVbaZgyrZ1VyqvOtXdpsbM5DbtyTcZk%2FC6Pnjl7vtHbt9mt7cq08fXJVnrlbuC19%2FRgsfIVsqbsHdud3WOIxe5QLKiiRgs3UqLTwseVzurtF%2F1FKbKbwKcelqg7GwefWmwwYFthDoDKzZYohLoDVO1z6jui%2FWrRlhTTjtGDF2Gnf%2BJ03pFhEuHKVfIEZ9Yo%2BGS9siI%2BHRsTF2GFuD9p9gY4ex0aNji0anhm10GFt%2BdGyT8alhizxsAR0rPHUm5DMuk3J5dnZWq3hIZ8MDimhsnxpXMslxHSMiMCiUACWOjQu3kUkhyvS9%2BPsGvYcDdOAAjTwHxnsMYH%2FLgPEeAwZQIRcX5YDGK41xfb59QoXlxg6Cd9q1VHUGlutcLXA510qb3VmMkIjAYpdl9A%2Fo7YSXs3mzU8942vgCwntAOOhKh3q0h3qtGcCfsHzul99nhTvhq5Z48KuzMRq6WeiVicFldZ8MBoUYPVAIUSdgvULoCN92wpZlQOH1TgPm99tpcmrtNOWE7G%2BnaELIf9A8Bz1%2Fa%2FOcj0%2B9eaZe81zEdsXLvAwyvYPOldIx8tT%2BNP2TP38DhXzh33elyq5wXxLrRrMguixrrawuqree0GuoXOcw6D4nvcPMoIOHHWxcb2aE7zQ08LJ90al8bd8W2dVP%3C%2Fdiagram%3E%3C%2Fmxfile%3E)

This section introduces some key types related to nodes.

### FNodeTemplate

A `FNodeTemplate` is a combination of a set of `traits`, which can include multiple sets of `base + additive` traits. All animation graphs can share the same template object.

![FNodeTemplate](FNodeTemplate.png)

The `traits` that make up the node template are stored as `FTraitTemplate` objects at the end of the `contiguous memory` of the `FNodeTemplate` object.

![FNodeTemplate::GetTraits](FNodeTemplate_GetTraits.png)

> `FNodeTemplateBuilder::BuildNodeTemplate` constructs an FNodeTemplate object and the FTraitTemplate contained in it in a contiguous buffer of `TArray<uint8>`
{: .prompt-tip }

From it, you can obtain the `UID`, `base address of the trait array`, the number of traits, and other information.

- `FNodeTemplate::NodeSharedDataSize` is the size of the `shared data` for all traits (including the `FLatentPropertiesHeader` of the base trait) after alignment.
- `FNodeTemplate::NodeInstanceDataSize` is the size of the `instance data` for all traits after alignment.

![rough memory layout of FNodeTemplate](MemoryLayoutOfFNodeTemplate.png)

#### FTraitTemplate

`FTraitTemplate` is the `trait` within the node template. In addition to providing basic information about the trait such as `UID`, trait type, shared/instance data, number of subtraits, and number of `Latent Properties`, it also allows you to obtain the `offset of the shared data`, the `offset of the pointer to the shared latent property array`, and the `offset of the instance data`.

![FTraitTemplate](FTraitTemplate.png)

> See：`FNodeTemplate::Finalize`
{: .prompt-tip }

> I think the naming of the two member functions `FTraitTemplate::GetTraitDescription` is a bit confusing. The more understandable name should be `GetTraitSharedData`, which is used to get the shared data pointer of the trait. It may be a typo or a name change.
{: .prompt-tip }

### FNodeDescription

The only read-only data within an `animation graph`. Although the object itself is 8 bytes in size, when allocating memory, it includes the size of the shared data from the traits on the node, making it `an object whose size varies in usage`.  

![rough memory layout of FNodeDescription](MemoryLayoutOfFNodeDescription.png)

> See：
>
> `FTraitReader::ReadGraphSharedData`
>
> and some other details:
> `FNodeDescription::Serialize`
> `FTrait::SerializeTraitSharedData`
> `FTraitWriter::WriteNode`
> `FTrait::SaveTraitSharedData`
> 
> After reading, it is stored in `UAnimNextAnimationGraph::SharedDataBuffer`. For usage, refer to `FExecutionContext::GetNodeDescription`
{: .prompt-tip }

- `FNodeDescription::TemplateHandle` is used to obtain an instance of the `FNodeTemplate` object from `FNodeTemplateRegistry`.
- `FNodeDescription::NodeInstanceDataSize` contains the size of the `instantiation data`, plus the `total size` of all `Latent Properties`.

> Note the difference between `FNodeDescription::NodeInstanceDataSize` and `FNodeTemplate::NodeInstanceDataSize`
>
> In addition, `FNodeDescription` and `FNodeTemplate` are `many-to-one`, which can be understood from their usage.
{: .prompt-tip }


### FNodeInstance

The instantiated data of the node, dynamically created at runtime, has a size of 16 bytes itself. When allocating memory, it includes the size of the instantiated data of the traits and their `Latent Property` size, `which is also an object that varies in size depending on usage`; built-in reference counting.

![rough memory layout of FNodeInstance](MemoryLayoutOfFNodeInstance.png)

> For usage, refer to ：`FAnimationAnimNextRuntimeTest_TraitSerialization::RunTest`，`FExecutionContext::AllocateNodeInstance`
{: .prompt-tip }

### FNodeTemplateRegistry

The global registry of the `FNodeTemplate` object, ensuring that all `FNodeTemplate` instances are contiguous in memory.

### FTraitStackBinding

Describes the data required for a `set of traits` (one base trait and its children/additive traits) used to query traits or the `trait interface`.

> See：`FTraitStackBinding::FTraitStackBinding`，especially the last few lines
{: .prompt-tip }

For example, `FTraitStackBinding::GetInterfaceImpl`: attempts to find the trait that implements the specified `InterfaceUID` from the `trait stack` and returns the `binding` of that trait.

### FTraitBinding

Describes the data of a `specific trait` within a set of traits, allowing you to query whether the current trait implements the specified `trait interface`.

### TTraitBinding

Strongly typed/type-safe `FTraitBinding`.

## FExecutionContext

Since traits are stateless, the `dynamic data`/`instance data` during execution requires an object to hold it, which is the `FExecutionContext`, the execution context object.

It is used to `bind` to a graph instance and provides a unified `trait query` interface for nodes.

The `Update` and `Evaluate` processes of the graph are encapsulated in the execution function of a RigVM node: `FRigUnit_AnimNextRunAnimationGraph_v2_Execute()`.

### FUpdateTraversalContext

The derived context object used when `updating` the graph.

Internally, it uses a stack (LIFO) allocated on the `MemStack` to implement a `depth-first traversal` of the `trait tree`, rather than the `recursive` method used in ABP.

> See: `UE::AnimNext::UpdateGraph`. Each trait is executed twice in the while loop, corresponding to `IUpdate::PreUpdate` and `IUpdate::PostUpdate`.
> 
> `IUpdate::OnBecomeRelevant` is also called here.
{: .prompt-tip }

### FEvaluateTraversalContext

The derived context object used when `evaluating` the graph.

The internal `FEvaluationProgram` is used to store the `FAnimNextEvaluationTask` that each node needs to execute while traversing the graph.  
It then calls `FEvaluationProgram::Execute` to perform each evaluation task on the `FEvaluationVMStack`.

> See: `UE::AnimNext::EvaluateGraph`, the execution process is similar to `UpdateGraph`
{: .prompt-tip }

#### FAnimNextEvaluationTask

`FAnimNextEvaluationTask` is a logical object that can be reused between traits, representing micro-instructions running on the `evaluation virtual machine`, which can handle input and output at the same time through the internal state of the virtual machine (also a `stack`).

# UAF Modules

UAF consists of multiple modules:

![UAF On 5.6](UAFOn5.6.png)

On the `ue5-main` branch, the prefix of the series of module names has changed from `AnimNext` to `UAF`.

![UAF On ue5-main](UAFOnMain.png)

Among them:
`UAF`/`AnimNext`: Provides core animation utility functions, defines base class interfaces, etc. For example: `UE::AnimNext::FDecompressionTools::GetAnimationPose`

`UAFAnimGraph`/`AnimNextAnimGraph`: Implements RigVM-based functionality related to animation graphs.

These two modules are the main focus of this article.

Other modules introduce functionalities from `other modules/systems` to `UAF`, such as incorporating `StateTree`, `PoseSearch`, etc.

## SoA (struct of array)

Each utility function in `TransformArrayOperations.h` has both `AoS` and `SoA` versions, with the code prioritizing the use of `AoS`. This indicates that in terms of data structure design, UAF is more `data-oriented`.

# Conclusion

UAF is a redesigned, data-oriented, composition-oriented, high-performance, flexible, concise, and easily extensible animation framework.

It completely abandons the ABP framework and embraces RigVM.

Due to its feature set still being in development, it is currently in an experimental phase.

# Miscellaneous

## Writing Animation Blueprints in Code

Refer to code of `test cases` from `UAFTestSuite`/`AnimNextTestSuite` and `UAFAnimGraphTestSuite`/`AnimNextAnimGraphTestSuite` modules.

## Playing "Montages" in UAF

Due to the length of this article, I decide to not to elaborate further.

The relevant nodes are `UInjectionCallbackProxy`, `UPlayAnimCallbackProxy`.

The related code can be found in `UE::AnimNext::FInjectionUtils::Inject`.

## Official FAQ Link

[Unreal Animation Framework (UAF, AnimNext) FAQ](https://dev.epicgames.com/community/learning/knowledge-base/nWWx/unreal-engine-unreal-animation-framework-uaf-faq)