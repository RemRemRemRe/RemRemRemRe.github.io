---
title: Latent Timer
date: 2024-10-05 13:14:52 +0800
categories: [Unreal Engine, Plugins]
tags: [gameplay, remcommon, tutorial, documentation]    # TAG names should always be lowercase
lang: en
---

## Latent timer

### Why "reinventing the wheel" while we have `TimerManager`

Well, `TimerManager` has many problems when it comes to gameplay:

- `SetTimerForNextTick` actually called this tick [rather than the next tick](https://github.com/EpicGames/UnrealEngine/commit/6e2c11d3544ea67564e259703b074c6e24e530fa)

- Callback order is not guaranteed to be the same as the timer set order

- Can't specify `tick group` when setting timer

- `TimerManager::Tick` itself is hardcoded in some late time of a frame

- Only support delay in time seconds, do you want to delay in frames?

- Don't support loop count

- `FTimerDta` is still quite bloated, with [size optimization](https://github.com/EpicGames/UnrealEngine/commit/dc7199ef66bbeed3078a1e658b1043967ff7dbeb) done on `FTimerUnifiedDelegate`


### Could we solve all the problems?

Well, well, solving 100% of them is hard, but 90% is piece of cake with our great savior --- `FLatentActionManager`

`FLatentActionManager` is a simple but powerful tool to `tick` any instance of `FPendingLatentAction` every frame.

Every `LatentAction` is bound to a `UObject`, it ticks in the `tick group` of the bound object ! Or "by the end of frame" if tick disabled for the bound object where is just near and ahead of `TimerManager::Tick`.

`FPendingLatentAction` could be derived to do anything you want. Eg: `FDelayUntilNextTickAction`, `FDelayAction` they are the heroes behind the beloved delay node in blueprint.

### The way I solve it with [Rem::Latent::FTimerLatentAction_Delay](https://github.com/RemRemRemRe/RemCommon/blob/main/Source/RemCommon/Public/Latent/RemLatentTimer.h)

- Providing `Rem::Latent::SetTimerForThisTick` and `Rem::Latent::SetTimerForNextTick` for maximum explicitly and flexibility when expressing `delay a tick`

- `Latent Action` is processed in the order they get bound to the `UObject`

- The tick group of a `Latent Action` could be controlled by specifying a ticking object within the target tick group. And it support specifying tick dependency with no efforts!

- Support delay in frames! Which doesn't likely to exist in `TimerManager`. Two helper struct: `FTimerParameterHelper_Time`, `FTimerParameterHelper_Frame`, one API: `Rem::Latent::SetTimer`

- Support specific loop count : `FTimerParameterHelper_Time::LoopCount`, `FTimerParameterHelper_Frame::LoopCount`

- Support counting from next frame: see `FTimerParameterHelper_Time::bSkipCountingThisFrame`

- My `FTimerLatentAction_Delay` only has size of 40 bytes to get all the jobs done, while `FTimerDta` has the size of 128, 3.2x bigger!

- A `fire and forget` alternative for pausing timer for one frame pause: `Rem::Latent::SetTimerPausedOneFrame`

- Familiar APIs: `Rem::Latent::PauseTimer`, `Rem::Latent::UnpauseTimer`, `Rem::Latent::SetTimerPaused`(did we met before?), `Rem::Latent::StopTimer`, `Rem::Latent::FindTimerAction`

- Re-triggerable is natively supported: `Rem::Latent::ResetTimerDelay`, support both delay in time and in frame

- Call count compensation and opting out it with `FTimerParameterHelper_Time::bMaxOncePerFrame` (Same as what's in `TimerManager`)

- 27 bits wasted for now, they are the hope for the future!

### Limitations

- `Rem::Latent::FTimerHandle` is 32-bit, because `FLatentActionManager::AddNewAction` only accepts `int32`, while it was `uint64` in `TimerManager`

- Infinite loop map happen if `Rem::Latent::SetTimerForThisTick` is called on the same object within `FLatentActionManager::ProcessLatentActions`, use `Rem::Latent::SetTimerForNextTick` instead in the case

- `Rem::Latent::SetTimerForThisTick` will not get called in the relevant `tick group`, if the bound object is already ticked this frame, consider `Rem::Latent::SetTimerForNextTick` instead in the case

- `TimeToDelay`, `LoopCount`, `InitialDelay` are all 4 bytes only for simplicity, might consider extended to those 27 spared bits in the future

- Requires `tick enabled` on the bound object and it has to be a `blueprint class object` to be able to "set tick group" for our timer latent action

### Sample code

```cpp
void UYourObject::DoJob()
{
    auto TimerHandle = Rem::Latent::SetTimerForThisTick(*this,
        FTimerDelegate::CreateUObject(this, &ThisClass::Callback));
}
```

```cpp
void UYourObject::TryDoJobUntilSucceed()
{
    bool bWantToRetry{true};

    ON_SCOPE_EXIT
    {
        if (bWantToRetry)
        {
            Rem::Latent::SetTimerForNextTick(*this, FTimerDelegate::CreateWeakLambda(this,
            [this]
            {
                TryDoJobUntilSucceed();
            }));
        }
    };

    // ...
}
```

```cpp
void UYourObject::RetriggerableJob()
{
    // ...

    if (!TimerHandle.IsValid())
    {
        TimerHandle = Rem::Latent::SetTimer(*this, FTimerDelegate::CreateWeakLambda(this, [this]
        {
            // ...
            Rem::Latent::StopTimer(*this, TimerHandle);
            TimerHandle = {};
        }), {.TimeToDelay = 1.0f, .LoopCount = 0/*loop infinite*/});
    }
    else
    {
        Rem::Latent::ResetTimerDelay(*this, TimerHandle);
    }
}
```
