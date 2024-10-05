---
title: Re:RemGameplayCamera from zero
date: 2024-02-25 00:21:20 +0800
categories: [Unreal Engine, Plugins]
tags: [gameplay, remgameplaycamera, tutorial, documentation]    # TAG names should always be lowercase
lang: en
media_subpath: /assets/img/RemGameplayCamera/
---

## Preface

This page is a simple and clear tutorial for `RemGameplayCamera` plugin

It would cover the very basics you need to use the plugin

Hope you will like it

(I will continue to improve this page, and any feedback or contribution is welcomed)


## Introduction

`RemGameplayCamera` plugin is a data-driven gameplay camera system for unreal engine projects. It provides a state based(tag based), modular, prioritized camera data configuration with data asset, support real time editing

Built on top of the existing camera framework make it full compatible with camera sequence, camera shake, view target switching and other camera effects.

It's `AActor` based, so any actor class could use it (`UAbilitySystemComponent` is required for now).

With built-in camera location, rotation smoothing (lag) and many other mechanisms, you can easily implement camera system like what's in `ALS` or `Lyra` by just tweaking the camera configurations without writing a line of code.

It also come with some basic functionality for `free look`, `enemy lock`, `mesh fading gradually` and `post processing management`, with extensibility in mind.

The `camera data processing pipeline` is divided into several parts, every part of these could be extended with `blueprint` or `code`!

You could easily implement something like `speed based fov`, `speed based camera offset`.

Note: (currently) It rely on the ability system component from the view target to provide `gameplay tag event-driven` camera data update, in this way, it's also loose-coupled with the rest of the game world.

## 0. Starting from `Third Person template`

Not actually from zero😀, I will guide you to do a simple walk through on the `RemGameplayCamera` system by trying to replace the `default camera` in the "Third Person Template".

## 1. Copy files in `Config` directory into your project

Assuming you've created the `Third Person project` and put RemGameplayCamera plugin in place.

Please copy `RemGameplayCamera/Config` folder to `YourProject/Config` folder, these are the `default` `GameplayTag` configs for the system. It will register these default tags into the system, and get the `URemCameraSettings` object configured.

You could check it by navigating to `Project Settings` -> `Game` -> `Rem Camera Settings` after opening the editor. And customize it if you want.

## 2. Option-in the third person character

By Default, the "rem camera system" is disabled for every view target.

View target actor that wants to use the system should have `RemTickCamera` in their `AActor::Tags` property to be able to get identified.

So, go ahead and open `BP_ThirdPersonCharacter`, and add it from the detail panel:
![AddRemTickCamera](AddRemTickCamera.jpg)

## 3. Create essential files

In order to use the `Rem camera system`, we need to use the `ARemPlayerCameraManager` and prepared the "camera data" for it.

### create `BP_RemCameraManager`

So, first, let's create this camera manager blueprint named "BP_RemCameraManager" that derived from `ARemPlayerCameraManager` class. It's this class that coordinate the camera system.

### create `BP_PlayerController`

In order to utilize the `BP_RemCameraManager`, create a player controller blueprint named "BP_PlayerController" that derived from `APlayerController` and assign "BP_RemCameraManager" to it's `Player Camera Manager Class` property:
![AssignPlayerCameraManagerClass](AssignPlayerCameraManagerClass.jpg)

### create `BP_GameMode`

In order to utilize the `BP_PlayerController`, create a game mode blueprint named "BP_GameMode" that derived from `AYourProjectNameGameMode` and assign "BP_PlayerController" to it's `Player Controller Class` property:
![AssignPlayerControllerClass](AssignPlayerControllerClass.jpg)

### create camera data files

`BP_RemCameraManager` need `camera data` to work as expected.

Camera data is organized by a simple hierarchy:
- `URemCameraSettingForViewTargets` has all the camera settings for all the view targets in the game. Referenced by the `ARemPlayerCameraManager`. It's the top or root node of the hierarchy.
- `URemCameraSettingAssetsForViewTarget` has camera settings for **`a`** kind of view target. Referenced by the `URemCameraSettingForViewTargets`.
- `URemCameraSettingAsset` is where the actual camera setting values resides. Referenced by the `URemCameraSettingAssetsForViewTarget`.


#### create `DA_Camera_Setting`

First, we create a data asset named `DA_Camera_Setting` of type `URemCameraSettingAsset`, 

copy these value and paste into `State Query` property:

```
(TokenStreamVersion=0,TagDictionary=,QueryTokenStream=(0,1,6,1,1,0),UserDescription="",AutoDescription=" NONE(  ANY( ) )")
```

this will make the `tag query` always matching, letting the camera setting asset we create take effect.

> Normally, this should match specific view target `state` which is represented as `gameplay tag`
{: .prompt-tip }

then, copy these value and paste into `Setting Values` property:

```
((SettingTag=(TagName="CameraSettingValue.CameraTransform.Location.Offset"),Value=/Script/RemGameplayCamera.RemCameraDataLocationOffset_Fixed(Offset=(X=-280.000000,Y=0.000000,Z=0.000000))),(SettingTag=(TagName="CameraSettingValue.Fov.Value"),Value=/Script/RemGameplayCamera.RemCameraDataFov_Fixed(Fov=90.000000)),(SettingTag=(TagName="CameraSettingValue.PivotTransform.Value"),Value=/Script/RemGameplayCamera.RemCameraDataTransform_MeshTransform(SocketName="spine_05",Offset=(X=0.000000,Y=0.000000,Z=0.000000))),(SettingTag=(TagName="CameraSettingValue.Trace"),Value=/Script/RemGameplayCamera.RemCameraDataTrace_Collision(TraceRadius=12.000000,TraceDistanceRatioInterpolationSpeed=10.000000,TraceStartLocationAlpha=(Blend=None),TraceStartTransform=None)),(SettingTag=(TagName="CameraSettingValue.CameraTransform.Location.Blend"),Value=/Script/RemGameplayCamera.RemCameraDataBlendAlpha_Blend(Blend=/Script/RemGameplayCamera.RemCameraAlphaBlend(Blend=/Script/RemGameplayCamera.RemAlphaBlendOption(BlendTime=1.000000)))))
```

these values tries to mimic the spring arm settings on `BP_ThirdPersonCharacter`.

#### create `DA_Camera_ViewTarget`

Now that the setting asset is ready, we gonna create another data asset named `DA_Camera_ViewTarget` of type `URemCameraSettingAssetsForViewTarget`. It specifies the setting assets to use for our character.

For the `View Target Tag Query` property, we would use the same value up there👆 as `DA_Camera_Setting::State Query` as we simply want it to be matched and used.

> Normally, this should match specific view target `identifier` which is also represented as `gameplay tag`
{: .prompt-tip }

then add the `DA_Camera_Setting` to it's `SettingAssets` property.


#### create `DA_Camera_ViewTargets`

Finally, we create the last data asset named `DA_Camera_ViewTargets` of type `URemCameraSettingForViewTargets`. It has all the camera data about every view targets in the game.

Add the `DA_Camera_ViewTarget` to it's `Settings for View Targets` property, that's it.


Last but not least, assign `DA_Camera_ViewTargets` to the `BP_RemCameraManager::CameraSettingForViewTargets` property.

![AssetsWeNeed](AssetsWeNeed.jpg)

## 4. PIE, start!

After changing the game mode to `BP_GameMode`, and hit Play

You may find the camera following character's spine movement, more visible when it landing. And the camera has a smoothing effect when getting away from a colliding object (ALS like). 

After typing the console command `Rem.Camera.DrawDebug.Shape 1`, a blue sphere would show up around the spine of the character indicating the `Pivot Location` which is the same with ALS.

![TheBlueSphere](TheBlueSphere.jpg)

## 5. Congratulations

thanks for your time

♥

### How the camera location get calculated

![HowToGetCameraLocation](HowToGetCameraLocation.png)

This👆 image contains `terms` that is crucial to understand the system, anytime feeling confused, you can refer to it or asking for help in our group.

Every piece of the data that is needed by the camera pipeline could be extended.

There are also many built-in functionalities for you, feel free to explorer it!

> For more information, please look at the tooltips of properties on `URemCameraSettings` and `FRemCameraSettingTagValue`
{: .prompt-tip }
