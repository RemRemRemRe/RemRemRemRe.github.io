---
title: 使用虚幻引擎中在不相关的骨架间共享动画
date: 2026-02-14 17:17:17 +0800
categories: [Unreal Engine, Animation]
tags: [gameplay, animation, tutorial]    # TAG names should always be lowercase
lang: zh
media_subpath: /assets/img/AnimationSharing/
---

## 目标效果

本文主要介绍一种实现下图动画效果的方法：

![Zomboid Dog](ZomboidDog.gif)

与正常的，通过`动画重定向`正确`缩放`并共享动画`相反`，本方法可以用于得到与模型无关的，完全一致的骨骼数据。即不论用什么模型，播放动画`得到的骨骼数据是一样`的。

## 前置知识

首先让我们快速回顾下，虚幻引擎`动画系统`的`基础的动画资产` ：

`骨架`，`骨骼模型`，`动画序列`是虚幻引擎中播放动画所必须的基础资产

`理想情况`下，每种资产各司其职（单一职责），

`骨架`提供骨骼层级数据，
与`动画序列`一起得到每帧的姿势数据
`骨骼模型`提供蒙皮信息，
用于后续将姿势数据转换到顶点数据，最终渲染出来在动的模型

下面我们`逐项了解`一下本文关注的数据项

### 骨架

```c++
/**
 *	USkeleton : that links between mesh and animation
 *		- Bone hierarchy for animations
 *		- Bone/track linkup between mesh and animation
 *		- Retargetting related
 */
class USkeleton 
```

其核心成员变量及用途：

`TArray<struct FBoneNode> BoneTree`
- 描述骨骼间的层级信息（多叉树）

`TMap<TObjectKey<USkinnedAsset>, TUniquePtr<FSkeletonToMeshLinkup>> SkinnedAssetLinkupCache`
- 与`若干`个骨骼模型的参考骨架之间的骨骼下标的映射关系 (骨架的骨骼下标与模型的参考骨架的骨骼下标的映射)

`TMap< FName, FReferencePose > AnimRetargetSources`
- 用于动画重定向

`FReferenceSkeleton ReferenceSkeleton`
- `参考骨架`，保存了来自于原始模型的初始`骨骼及姿势`信息，包括`参考姿势`

- 以及其它很多信息

  `TArray<TSoftObjectPtr<USkeleton>> CompatibleSkeletons`
  - 兼容的其它骨架（兼容性骨架）

  `TArray<FAnimSlotGroup> SlotGroups`
  - 蒙太奇插槽和分组
  
  `TArray<FVirtualBone> VirtualBones`
  - 虚拟骨骼

  `TArray<TObjectPtr<class USkeletalMeshSocket>> Sockets`
  - 插孔

  `TArray<TObjectPtr<UBlendProfile>> BlendProfiles`
  - 混合用的配置文件

### 骨骼模型

```c++
/**
 * SkeletalMesh is geometry bound to a hierarchical skeleton of bones which can be animated for the purpose of deforming the mesh.
 * Skeletal Meshes are built up of two parts; a set of polygons composed to make up the surface of the mesh, and a hierarchical skeleton which can be used to animate the polygons.
 * The 3D models, rigging, and animations are created in an external modeling and animation application (3DSMax, Maya, Softimage, etc).
 *
 * @see https://docs.unrealengine.com/latest/INT/Engine/Content/Types/SkeletalMeshes/
 */
class USkeletalMesh
```

其核心成员变量及用途：

`TObjectPtr<USkeleton> Skeleton`
- 关联的骨架

`FReferenceSkeleton RefSkeleton`
- 参考骨架

- 其它信息

  `TObjectPtr<USkeletalMeshLODSettings> LODSettings`
  - 细节层级配置

  `TArray<TObjectPtr<class USkeletalMeshSocket>> Sockets`
  - 插孔

  `TArray<TObjectPtr<UMorphTarget>> MorphTargets`
  - 形变动画（顶点动画）


### 动画序列 

```c++
class UAnimSequence : public UAnimSequenceBase
```

其核心成员变量及用途：

`TObjectPtr<class USkeleton> Skeleton`
- 关联的骨架

`FRawCurveTracks RawCurveData`
- 动画曲线`原始值`

`TArray<struct FAnimNotifyEvent> Notifies`
- 动画通知

- 其它

  `TScriptInterface<IAnimationDataModel> DataModelInterface`
  - 动画`源数据`载体 （Raw Data）

  `TObjectPtr<class UAnimBoneCompressionSettings> BoneCompressionSettings`
  - 骨骼压缩设置
  
  `TObjectPtr<class UAnimCurveCompressionSettings> CurveCompressionSettings`
  - 曲线压缩设置

  `FName RetargetSource`
  - 重定向来源
  
  `TArray<FAnimSyncMarker>		AuthoredSyncMarkers`
  - 同步标记

### USkeletalMeshComponent

```c++
/**
 * SkeletalMeshComponent is used to create an instance of an animated SkeletalMesh asset.
 *
 * @see https://docs.unrealengine.com/latest/INT/Engine/Content/Types/SkeletalMeshes/
 * @see USkeletalMesh
 */
class USkeletalMeshComponent 
```

其核心成员变量及用途：

`TArray<FBoneIndexType> RequiredBones`
- 当前模型的`细节层级`下所需要的骨骼所对应的`模型的``参考骨架的``下标`：如；`0, 5, 10, 13...`

`TSharedPtr<struct FBoneContainer> SharedRequiredBones`
- `骨骼动画计算`所需的`关键数据结构`，在此组件上`所有动画实例`间`共享`


### 骨骼容器

```c++
struct FBoneContainer
```

> 它是`骨骼动画计算`的`核心数据`，无论最终动画表现时，使用的模型和骨架是什么，在动画计算时，所有动画实例及其节点`正常`应当只会使用来自`BoneContainer`的数据
{: .prompt-tip }

其核心成员变量及用途：

`TArray<FBoneIndexType>	BoneIndicesArray`
- 当前`细节层级`下所需要的骨骼所对应的`参考骨架的``下标`

`TWeakObjectPtr<USkeletalMesh> AssetSkeletalMesh`
- 当前使用的骨骼模型

`TWeakObjectPtr<USkeleton> AssetSkeleton`
- 当前使用的骨架

```c++
    // @see FBoneContainer::RemapFromSkelMesh
	SkeletonToPoseBoneIndexArray = LinkupTable.SkeletonToMeshTable;
	PoseToSkeletonBoneIndexArray = LinkupTable.MeshToSkeletonTable;
```

`TArray<int32> SkeletonToPoseBoneIndexArray`
- `骨架的骨骼下标`到`模型参考骨架的骨骼下标`的映射（感觉变量名有歧义，看上面的代码更直观）

`TArray<int32> PoseToSkeletonBoneIndexArray`
- `模型参考骨架的骨骼下标`到`骨架的骨骼下标`的映射（感觉变量名有歧义，看上面的代码更直观）

`TSharedPtr<FSkelMeshRefPoseOverride> RefPoseOverride`
- `参考姿势覆盖`

`bool bDisableRetargeting`
- 禁用重定向的开关

> 可以看到，实际情况比理想中复杂得多，由于系统功能的拓展，例如动画通知和重定向，这些资产类型被加入了许多数据
> 
> 并且除了骨架资产本身外，骨骼模型有自己的参考骨架，动画序列有自己的骨架，这导致资产间的关系复杂了起来，它们也是需要引入动画重定向的原因
{: .prompt-tip }

## 实现方案

在对上述的资产对象及代码流程理解充分之后，实现方案应该就呼之欲出了：

基于默认的`USkeletalMeshComponent::InitAnim`流程，在初始化`SharedRequiredBones`时，`想办法`覆盖`SharedRequiredBones::AssetSkeleton`，然后再执行后续步骤，即建立与`AssetSkeletalMesh`的骨骼下标映射等：

```c++
FBoneContainer::InitializeTo
FBoneContainer::Initialize
// ... 覆盖 AssetSkeleton
FBoneContainer::RemapFromSkelMesh
```

> 目前由于引擎默认未开放相关接口，只能通过改引擎的方式实现，且有少量代码需要适配，官方后续也未必会支持。这是目前方案的风险点
{: .prompt-warning }

套用到`僵尸毁灭工程`中`德牧翻越`的例子，就是先使用`德牧模型`执行InitAnim，在初始化SharedRequiredBones时，将`角色骨架`覆盖到AssetSkeleton，然后执行后续`FBoneContainer::RemapFromSkelMesh` （需要德牧骨架与角色骨架的层级相似，且共同层级的骨骼名字一致）

## 相关思考

### 骨骼下标的映射是如何建立的？

`默认`通过`骨骼名`映射，即骨骼`名字相同`的骨骼，互相映射

参见：
```c++
USkeleton::BuildLinkupData
```
#### CopyPoseFromMesh

参见：
```c++
FAnimNode_CopyPoseFromMesh::BoneMapToSource
FAnimNode_CopyPoseFromMesh::Evaluate_AnyThread
FAnimNode_CopyPoseFromMesh::ReinitializeMeshComponent
```

#### UPoseableMeshComponent

参见：
```c++
UPoseableMeshComponent::CopyPoseFromSkeletalComponent
```
#### LeaderPoseComponent

参见：
```c++
USkinnedMeshComponent::LeaderPoseComponent

// 更新下标映射
USkinnedMeshComponent::UpdateLeaderBoneMap

// 渲染相关
UpdateRefToLocalMatricesInner (SkeletalRender.cpp)

// 获取骨骼变换
USkinnedMeshComponent::GetBoneTransform
```

> 可以看到，引擎通过`下标映射`实现了名字各不相同但相似的功能
> 
> 也就是说，只要建立好下标映射，一定程度上，就能在任意模型上`强制共享动画`
{: .prompt-tip }

### 更新速率优化（URO）

#### 判断是否可跳过更新

参见：
```c++
USkinnedMeshComponent::TickUpdateRate
FAnimUpdateRateManager::TickUpdateRateParameters
USkeletalMeshComponent::ShouldTickAnimation

FAnimUpdateRateParameters::SetTrailMode
```

#### 插值骨骼数据

参见：
```c++
USkeletalMeshComponent::RefreshBoneTransforms
USkeletalMeshComponent::ParallelAnimationEvaluation
USkeletalMeshComponent::ParallelDuplicateAndInterpolate
FAnimationRuntime::LerpBoneTransforms
```

### 当动画蓝图的图表为空时，动画计算的输出结果是什么？

参考姿势

参见：
```c++
UAnimInstance::ParallelEvaluateAnimation
FPoseContext::ResetToRefPose
FBaseCompactPose::ResetToRefPose
FBoneContainer::FillWithCompactRefPose
```

### CacheBones_AnyThread缓存的是什么数据？

参考姿势的骨骼数据

参见：
```c++
UAnimInstance::ParallelEvaluateAnimation
FAnimInstanceProxy::EvaluateAnimation_WithRoot
FAnimInstanceProxy::CacheBones / FAnimInstanceProxy::CacheBones_WithRoot
```

### 参考姿势如何与动画序列的数据混合/交互/应用

骨骼数据解压流程：
```c++
UAnimSequence::GetAnimationPose
UAnimSequence::GetBonePose
UE::Anim::Decompression::DecompressPose

// ACLImpl.h
```

> 跟压缩算法有关：如果动画数据在压缩时没有剔除参考姿势的数据，即`全量数据`压缩，则动画序列的骨骼数据会覆盖掉参考姿势。旧版压缩算法就是如此。
> 
> 在虚幻引擎`5.3`版本后，动画的默认压缩算法切换到了[ACL](https://github.com/nfrechette/acl)，默认配置下，它在压缩动画时，会剔除参考姿势数据，相对它进行`增量数据`压缩，增量为零，即数据与参考姿势一致的轨道会被跳过；
> 对应的，解压时也会跳过数据与参考姿势数据一致的轨道。所以它可以做到压缩率更高，解压速度更快。也因为如此，解压时需要获取参考姿势的信息。
{: .prompt-tip }

## 探索

### 给定一个骨骼下标，如何最快判断它是否来自一个虚拟骨骼？

### 禁用重定向且使用ACL压缩时，参考姿势不同的，比例接近的骨骼模型，播放同一动画序列时，动画结果是否会一致？

### 如果名字不同的骨骼也能互相映射下标，会有什么用？