---
title: Re:从零开始的RemGameplayCamera
date: 2024-10-05 20:55:20 +0800
categories: [Unreal Engine, Plugins]
tags: [gameplay, remgameplaycamera, tutorial, documentation]    # TAG names should always be lowercase
lang: zh
media_subpath: /assets/img/RemGameplayCamera/
---

## 前言

本文是 `RemGameplayCamera` 插件的简单明了的入门教程

它将涵盖使用该插件所需的基础知识。

希望你会喜欢

(我会持续改进本文，欢迎任何反馈或者贡献)


## 介绍

`RemGameplayCamera` 插件是为虚幻引擎打造的一个数据驱动的玩法相机系统。它提供了基于状态（Tag）的，模块化的，具有优先级的相机数据配置，使用数据资产，支持实时编辑

基于引擎已有的相机框架之上搭建，让它完全兼容原生的相机序列，相机抖动，观看对象切换以及其他相机效果

他是基于`AActor`的，所以任意的`AActor`子类对象都可以接入，（当前 `UAbilitySystemComponent` 是必需的）

因为有内置的相机位置，旋转平滑及其他许多机制，只需要调整相机配置，就可以轻易的实现像 `ALS` 或者 `Lyra` 中那样的相机效果，不需要写一行代码

它内置了很多相关的基础功能，比如`自由视角`， `敌人锁定`， `模型渐隐` 和 `后处理管理`等，并且支持自定义拓展

`相机数据处理管线`被分成了多个部分，每个部分都可以通过`蓝图`或者`代码`自由拓展！

你可以很容易的实现像 `基于速度的FOV`， `基于速度的相机偏移量`等功能

注意：当前此插件依赖观看对象上的技能组件来提供基于`状态事件驱动`的相机数据更新机制。同时也通过这种方法，让相机与游戏世界中的其它部分解耦开来

## 0. 从 `第三人称模板` 开始

其实不是从零开始😀， 我将引导你做一个关于 `RemGameplayCamera` 的简单演练：替换第三人称模板中的`默认相机`

## 1. 拷贝 `Config` 目录中的文件到你的项目

假定你已经创建了 `Third Person project` 并且把 `RemGameplayCamera` 放进了插件目录里

请拷贝 `RemGameplayCamera/Config` 文件夹到 `YourProject/Config` 文件夹， 它们是相机系统的`默认` `GameplayTag` 配置。他们会把默认的`标签`注册到系统中，并且自动配置好 `URemCameraSettings` 对象

你可以在编辑器启动之后，去`Project Settings` -> `Game` -> `Rem Camera Settings` 检查配置是否生效。或者自定义它们，如果你想要的话

## 2. 对第三人称角色选择性启用

默认情况下，`rem camera system` 对每个观看对象都是禁用的

想要使用此系统的观看对象，需要在它们的`AActor::Tags`属性中，写入`RemTickCamera`这个名字，以便被正常识别

因此，现在打开 `BP_ThirdPersonCharacter`， 然后从细节面板中添加它:
![AddRemTickCamera](AddRemTickCamera.jpg)

## 3. 创建必需的文件

为了使用 `rem camera system`，我们需要使用 `ARemPlayerCameraManager`，并为它准备好 `相机数据`

### 创建 `BP_RemCameraManager`

所以，首先，让我们创建相机管理器蓝图，是它调度着整个系统，取名为"BP_RemCameraManager"，继承自`ARemPlayerCameraManager`

### 创建 `BP_PlayerController`

为了使用 `BP_RemCameraManager`， 创建一个玩家控制器蓝图，取名为"BP_PlayerController"， 继承自`APlayerController`，并且指定"BP_RemCameraManager" 到它的 `Player Camera Manager Class` 属性:
![AssignPlayerCameraManagerClass](AssignPlayerCameraManagerClass.jpg)

### 创建 `BP_GameMode`

为了使用 `BP_PlayerController`， 创建一个游戏模式蓝图，取名为"BP_GameMode"， 继承自`AYourProjectNameGameMode`，并且指定"BP_PlayerController" 到它的 `Player Controller Class` 属性:
![AssignPlayerControllerClass](AssignPlayerControllerClass.jpg)

### 创建相机数据文件

`BP_RemCameraManager` 需要 `相机数据` 以便正常工作

相机数据通过一个简单的层级组织起来：
- `URemCameraSettingForViewTargets` 拥有游戏中所有可观看对象的相机配置。 被 `ARemPlayerCameraManager`引用着。 这是层级中的顶层或者说根节点
- `URemCameraSettingAssetsForViewTarget` 拥有**一种**类型的观看对象的相机配置。 被 `URemCameraSettingForViewTargets`引用着
- `URemCameraSettingAsset` 是具体的相机配置数据所在的地方。 被`URemCameraSettingAssetsForViewTarget`引用着


#### 创建 `DA_Camera_Setting`

首先，我们创建一个数据资产， 取名为 `DA_Camera_Setting`， 拥有`URemCameraSettingAsset`类型

复制下面的数据，粘贴到`State Query` property:

```
(TokenStreamVersion=0,TagDictionary=,QueryTokenStream=(0,1,6,1,1,0),UserDescription="",AutoDescription=" NONE(  ANY( ) )")
```
它会让 `tag query`（状态查询/标签查询） 总是匹配的， 以便让我们的相机配置资产总是生效

> 通常，这里应该匹配特定观看对象的`状态`，这些状态是用`标签`表示的
{: .prompt-tip }

然后，复制下面的数据，粘贴到`Setting Values` 属性:

```
((SettingTag=(TagName="CameraSettingValue.CameraTransform.Location.Offset"),Value=/Script/RemGameplayCamera.RemCameraDataLocationOffset_Fixed(Offset=(X=-280.000000,Y=0.000000,Z=0.000000))),(SettingTag=(TagName="CameraSettingValue.Fov.Value"),Value=/Script/RemGameplayCamera.RemCameraDataFov_Fixed(Fov=90.000000)),(SettingTag=(TagName="CameraSettingValue.PivotTransform.Value"),Value=/Script/RemGameplayCamera.RemCameraDataTransform_MeshTransform(SocketName="spine_05",Offset=(X=0.000000,Y=0.000000,Z=0.000000))),(SettingTag=(TagName="CameraSettingValue.Trace"),Value=/Script/RemGameplayCamera.RemCameraDataTrace_Collision(TraceRadius=12.000000,TraceDistanceRatioInterpolationSpeed=10.000000,TraceStartLocationAlpha=(Blend=None),TraceStartTransform=None)),(SettingTag=(TagName="CameraSettingValue.CameraTransform.Location.Blend"),Value=/Script/RemGameplayCamera.RemCameraDataBlendAlpha_Blend(Blend=/Script/RemGameplayCamera.RemCameraAlphaBlend(Blend=/Script/RemGameplayCamera.RemAlphaBlendOption(BlendTime=1.000000)))))
```

这些数据尝试模拟`BP_ThirdPersonCharacter`的相机臂的配置

#### 创建 `DA_Camera_ViewTarget`

既然具体的配置数据准备好了，我们将要创建另一种数据资产，取名为 `DA_Camera_ViewTarget`， 拥有`URemCameraSettingAssetsForViewTarget`类型。 它指定了要给我们的角色使用的配置数据资产

对于 `View Target Tag Query` 属性， 我们将会使用跟上面👆   `DA_Camera_Setting::State Query`一样的值，因为我们希望它可以匹配成功并被使用

> 通常，这里应该匹配特定观看对象的`识别码`，它也是用`标签`表示的
{: .prompt-tip }

然后添加 `DA_Camera_Setting` 到它的 `SettingAssets` 属性


#### 创建 `DA_Camera_ViewTargets`

最后，我们创建最后一个数据资产，取名为 `DA_Camera_ViewTargets`， 拥有 `URemCameraSettingForViewTargets`类型。 它有着游戏中所有观看对象的相机配置的引用

添加 `DA_Camera_ViewTarget` 到它的 `Settings for View Targets` 属性， 完成


最后但同样重要的， 指定 `DA_Camera_ViewTargets` 到 `BP_RemCameraManager::CameraSettingForViewTargets`属性

![AssetsWeNeed](AssetsWeNeed.jpg)

## 4. PIE，启动！

在把游戏模式换成 `BP_GameMode` 后， 点击开始按钮

你可能会发现相机正跟着角色的脊柱运动，比较明显的是观察它落地的时候。另外，相机在远离发生碰撞的物体时，会有一个平滑过渡的过程，跟ALS中的表现类似

在控制台输入命令：`Rem.Camera.DrawDebug.Shape 1`， 一个蓝色的球体会显示在角色的脊柱附近，它代表着`Pivot Location`（枢纽点，支点），这点也跟ALS一样

![TheBlueSphere](TheBlueSphere.jpg)

## 5. 祝贺您完成教程

感谢您花时间阅读本文

♥

### 相机位置是如何计算的

![HowToGetCameraLocation](HowToGetCameraLocation.png)

👆 这张图片包含了一些便于理解这个系统的关键`用语`，当你感到困惑的时候，可以随时参考它，或者在我们的群里提问求助。

相机管线中需要的每份数据都可以被自由拓展

每份数据都为你内置了不少相关的功能，请自由探索它们

> 想知道更多信息，请查看 `URemCameraSettings` 和 `FRemCameraSettingTagValue` 的浮动提示
{: .prompt-tip }
