---
title: Sharing Animation on Unrelated Skeleton in UnrealEngine
date: 2026-02-14 17:17:17 +0800
categories: [Unreal Engine, Animation]
tags: [gameplay, animation, tutorial]    # TAG names should always be lowercase
lang: en
media_subpath: /assets/img/AnimationSharing/
---

## Target Effect

This article primarily introduces a method to achieve the animation effect shown below:

![Zomboid Dog](ZomboidDog.gif)

Unlike the standard approach of properly `scaling` and sharing animations through `animation retargeting`, this method allows you to obtain skeleton data that is entirely independent of the model. In other words, regardless of the model used, the skeleton data generated when playing the animation remains consistent.

## Prerequisite Knowledge

First, let's quickly review the `basic animation assets` of the Unreal Engine `animation system`:

`Skeletons`, `skeletal meshes`, and `animation sequences` are essential assets for playing animations in Unreal Engine.

In an `ideal scenario`, each asset serves its sole purpose (single responsibility):

The `skeleton` provides skeletal hierarchy data, 
which, along with the `animation sequence`, supplies the pose data for each frame. 
The `skeletal mesh` offers skinning information, 
which is used to convert the pose data into vertex data for rendering the animated model.

Now, let's `take a closer look` at the data items this article focuses on.

### Skeleton

```c++
/**
 *	USkeleton : that links between mesh and animation
 *		- Bone hierarchy for animations
 *		- Bone/track linkup between mesh and animation
 *		- Retargetting related
 */
class USkeleton 
```

Its core member variables and their uses:

`TArray<struct FBoneNode> BoneTree`
- Describes the hierarchical information between bones (multi-branch tree)

`TMap<TObjectKey<USkinnedAsset>, TUniquePtr<FSkeletonToMeshLinkup>> SkinnedAssetLinkupCache`
- Maps the bone indices between the reference skeleton of several skeletal models (mapping of bone indices from the skeleton to the reference skeleton of the model)

`TMap< FName, FReferencePose > AnimRetargetSources`
- Used for animation retargeting

`FReferenceSkeleton ReferenceSkeleton`
- The `reference skeleton`, which stores the initial `bone and pose` information from the original model, including the `reference pose`

- And many other pieces of information

  `TArray<TSoftObjectPtr<USkeleton>> CompatibleSkeletons`
  - Skeletons it's compatible with (Compatible Skeletons)

  `TArray<FAnimSlotGroup> SlotGroups`
  - Montage slots and groups
  
  `TArray<FVirtualBone> VirtualBones`
  - Virtual bones

  `TArray<TObjectPtr<class USkeletalMeshSocket>> Sockets`
  - Sockets

  `TArray<TObjectPtr<UBlendProfile>> BlendProfiles`
  - Profiles for blending

### Skeletal Models

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

Its core member variables and their purposes:

`TObjectPtr<USkeleton> Skeleton`
- Associated skeleton

`FReferenceSkeleton RefSkeleton`
- Reference skeleton

- Other information

  `TObjectPtr<USkeletalMeshLODSettings> LODSettings`
  - Level of detail settings

  `TArray<TObjectPtr<class USkeletalMeshSocket>> Sockets`
  - Sockets

  `TArray<TObjectPtr<UMorphTarget>> MorphTargets`
  - Morph animations (vertex animations)

### Animation Sequence

```c++
class UAnimSequence : public UAnimSequenceBase
```

Its core member variables and their purposes:

`TObjectPtr<class USkeleton> Skeleton`
- Associated skeleton

`FRawCurveTracks RawCurveData`
- Animation curve `raw values`

`TArray<struct FAnimNotifyEvent> Notifies`
- Animation notifications

- Others

  `TScriptInterface<IAnimationDataModel> DataModelInterface`
  - Animation `source data` carrier (Raw Data)

  `TObjectPtr<class UAnimBoneCompressionSettings> BoneCompressionSettings`
  - Bone compression settings
  
  `TObjectPtr<class UAnimCurveCompressionSettings> CurveCompressionSettings`
  - Curve compression settings

  `FName RetargetSource`
  - Retargeting source
  
  `TArray<FAnimSyncMarker>		AuthoredSyncMarkers`
  - Sync markers

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

Its core member variables and their uses:

`TArray<FBoneIndexType> RequiredBones`
- The indices of the bones needed at current model's `Level of Detail`: e.g., `0, 5, 10, 13...`

`TSharedPtr<struct FBoneContainer> SharedRequiredBones`
- The `key data structure` necessary for `skeletal animation evaluation`, shared among `all animation instances` on this component.

### Bone Container

```c++
struct FBoneContainer
```

> It's the `core data` for `skeletal animation evaluation`, no matter what modle or skeleton asset you use for the final presentation, at the evaluation time, all animation instances and it's animation nodes `normally` would only use data from `BoneContainer`
{: .prompt-tip }

Core member variables and their purposes:

`TArray<FBoneIndexType>	BoneIndicesArray`
- The indices corresponding to the bones required at the current `level of detail` in the `reference skeleton`

`TWeakObjectPtr<USkeletalMesh> AssetSkeletalMesh`
- The skeletal model currently being utilized

`TWeakObjectPtr<USkeleton> AssetSkeleton`
- The skeleton currently being utilized

```c++
    // @see FBoneContainer::RemapFromSkelMesh
	SkeletonToPoseBoneIndexArray = LinkupTable.SkeletonToMeshTable;
	PoseToSkeletonBoneIndexArray = LinkupTable.MeshToSkeletonTable;
```

`TArray<int32> SkeletonToPoseBoneIndexArray`
- Mapping from `bone index of the skeleton` to `bone index of reference skeleton of the model` (the variable name seems ambiguous; the code above is more intuitive)

`TArray<int32> PoseToSkeletonBoneIndexArray`
- Mapping from `bone index of reference skeleton of the model` to `bone index of the skeleton` (the variable name seems ambiguous; the code above is more intuitive)

`TSharedPtr<FSkelMeshRefPoseOverride> RefPoseOverride`
- `reference pose override`

`bool bDisableRetargeting`
- Switch to disable retargeting

> As can be seen, the actual situation is much more complex than ideal, as the expansion of system functionalities, such as animation notifications and retargeting, has added a lot of data to these asset types.
> 
> Additionally, aside from the skeletal assets themselves, skeletal models have their own reference skeletons, and animation sequences have their own skeletons, which complicates the relationships between assets and is also the reason they need to incorporate animation retargeting.
{: .prompt-tip }

## Implementation Detail

After thoroughly understanding the asset objects mentioned above and code flow, the implementation should become ituitive:

Based on the default `USkeletalMeshComponent::InitAnim` process, when initializing `SharedRequiredBones`, `find a way` to override `SharedRequiredBones::AssetSkeleton`, and then proceed with the subsequent steps, such as establishing the bone index mapping with `AssetSkeletalMesh`, etc.:

```c++
FBoneContainer::InitializeTo
FBoneContainer::Initialize
// ... override AssetSkeleton
FBoneContainer::RemapFromSkelMesh
```

> Currently, due to the engine did not expose the relevant interfaces by default, it can only be achieved by modifying the engine source, and some code adjustments are necessary. The official support for this may not be guaranteed in the future. This is a risk point of the current solution.
{: .prompt-warning }

An example applied to `Project Zomboid` with `German Shepherd Vault` is to first use the `German Shepherd Model` to execute InitAnim. During the initialization of SharedRequiredBones, override AssetSkeleton with the `Character Skeleton`, and then the subsequent `FBoneContainer::RemapFromSkelMesh` is executed (the German Shepherd skeleton and the character skeleton need to have a similar hierarchy, and the names of the bones at the common hierarchy must match).

## Related Thoughts

### How is the mapping of bone indices established?

By `default`, mapping is done through `bone names`, meaning bones with the `same name` are mapped to each other.

See:
```c++
USkeleton::BuildLinkupData
```
#### CopyPoseFromMesh

See:
```c++
FAnimNode_CopyPoseFromMesh::BoneMapToSource
FAnimNode_CopyPoseFromMesh::Evaluate_AnyThread
FAnimNode_CopyPoseFromMesh::ReinitializeMeshComponent
```

#### UPoseableMeshComponent

See:
```c++
UPoseableMeshComponent::CopyPoseFromSkeletalComponent
```
#### LeaderPoseComponent

See:
```c++
USkinnedMeshComponent::LeaderPoseComponent

// update bone index mapping
USkinnedMeshComponent::UpdateLeaderBoneMap

// rendering related
UpdateRefToLocalMatricesInner (SkeletalRender.cpp)

USkinnedMeshComponent::GetBoneTransform
```

> You can see that the engine achieves similar functionality with different names through `index mapping`.
> 
> In other words, as long as the index mapping is established, to a certain extent, `animation sharing can be enforced` on any model.
{: .prompt-tip }

### Update Rate Optimization (URO)

#### Determine if Updates Can Be Skipped

See:
```c++
USkinnedMeshComponent::TickUpdateRate
FAnimUpdateRateManager::TickUpdateRateParameters
USkeletalMeshComponent::ShouldTickAnimation

FAnimUpdateRateParameters::SetTrailMode
```

#### Interpolated Skeleton Data

See:
```c++
USkeletalMeshComponent::RefreshBoneTransforms
USkeletalMeshComponent::ParallelAnimationEvaluation
USkeletalMeshComponent::ParallelDuplicateAndInterpolate
FAnimationRuntime::LerpBoneTransforms
```

### What is the output of animation calculations when the animation blueprint graph is empty?

Reference Pose

See:
```c++
UAnimInstance::ParallelEvaluateAnimation
FPoseContext::ResetToRefPose
FBaseCompactPose::ResetToRefPose
FBoneContainer::FillWithCompactRefPose
```

### What Data is Cached on CacheBones_AnyThread?

Bone data of the reference pose

See:
```c++
UAnimInstance::ParallelEvaluateAnimation
FAnimInstanceProxy::EvaluateAnimation_WithRoot
FAnimInstanceProxy::CacheBones / FAnimInstanceProxy::CacheBones_WithRoot
```

### How to Blend/Interact/Apply Reference Poses with Animation Sequence Data

Skeleton data decompression process:
```c++
UAnimSequence::GetAnimationPose
UAnimSequence::GetBonePose
UE::Anim::Decompression::DecompressPose

// ACLImpl.h
```

> It's up to the compression algorithms: If the animation data does not exclude reference pose data during compression, that is, if `full data` compression is used, the skeletal data of the animation sequence will overwrite the reference pose. This was the case with the old compression algorithm.
> 
> After version `5.3` of Unreal Engine, the default compression algorithm for animations switched to [ACL](https://github.com/nfrechette/acl). Under the default settings, it removes reference pose data when compressing animations, performing `incremental data` compression where the increment is zero, meaning that tracks with data identical to the reference pose will be skipped; correspondingly, during decompression, tracks with data matching the reference pose will also be skipped. This allows for higher compression rates and faster decompression speeds. Because of this, reference pose information is needed during decompression.
{: .prompt-tip }

## Exploration

### Given a bone index, what's the most efficient way to determine if it's from a virtual bone?

### When disabling retargeting and using ACL compression, if the reference poses of the skeletal models are different but their proportions are similar, will the animation results be consistent when playing the same animation sequence?

### What would be the benefit if bones with different names could also be mapped to each other on indices?