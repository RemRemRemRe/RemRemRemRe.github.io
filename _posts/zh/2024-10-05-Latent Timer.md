---
title: Latent Timer 延时定时器
date: 2024-10-05 22:35:09 +0800
categories: [Unreal Engine, Plugins]
tags: [gameplay, remcommon, tutorial, documentation]    # TAG names should always be lowercase
lang: zh
---

## Latent timer

### 既然已经有了 `TimerManager`，为什么还要造轮子

因为， 在用`TimerManager`做玩法的时候，会有不少问题:

- `SetTimerForNextTick` 实际上是当帧触发，而不是下帧 [see this commit](https://github.com/EpicGames/UnrealEngine/commit/6e2c11d3544ea67564e259703b074c6e24e530fa)

- 回调顺序与定时器的设置顺序不保证一致

- 设置定时器时，不能指定 `tick group`

- `TimerManager::Tick` 是硬编码在当帧稍晚的时刻

- 只支持以秒/时间单位设置延迟，你想不想用帧数来设置延迟?

- 不支持循环次数

- 尽管已经对 `FTimerUnifiedDelegate` 做了[内存大小的优化](https://github.com/EpicGames/UnrealEngine/commit/dc7199ef66bbeed3078a1e658b1043967ff7dbeb)，但`FTimerDta` 还是很臃肿


### 那我们能解决所有提到的问题么?

嗐，全解决有点难，但解决90%的问题就是小菜一碟，因为我们有个大救星 --- `FLatentActionManager`

`FLatentActionManager` 是一个简单又强大的工具，用来每帧更新任意的 `FPendingLatentAction` 实例

每个 `LatentAction` 会绑定到一个 `UObject`， 它会在绑定对象的 `tick group` 更新 ! 但如果绑定对象的更新被禁用了，它就会在“接近帧末”的时刻更新，就在接近且恰好早于`TimerManager::Tick`的时刻

`FPendingLatentAction` 可以被继承，做任意你想要的事情。 例如: `FDelayUntilNextTickAction`， `FDelayAction` 它们就是蓝图中延迟节点的幕后英雄

### 我解决上述问题的方法：[Rem::Latent::FTimerLatentAction_Delay](https://github.com/RemRemRemRe/RemCommon/blob/main/Source/RemCommon/Public/Latent/RemLatentTimer.h)

- 提供 `Rem::Latent::SetTimerForThisTick` 和 `Rem::Latent::SetTimerForNextTick` 以在表达`延迟一帧`的语义时，提供最大限度的明确和灵活性

- `Latent Action` 是按照它们绑定到`UObject`的顺序进行处理的

- `Latent Action`的`Tick group` 可以通过绑定一个在目标分组的对象上来控制。 并且顺理成章的支持指定更新依赖

- 支持按帧数延迟! 这个功能可能不会出现在 `TimerManager`。 两个辅助结构体: `FTimerParameterHelper_Time`， `FTimerParameterHelper_Frame`， 一个统一的API: `Rem::Latent::SetTimer`

- 支持指定循环次数 : `FTimerParameterHelper_Time::LoopCount`， `FTimerParameterHelper_Frame::LoopCount`

- 支持从下帧开始计数: 详见：`FTimerParameterHelper_Time::bSkipCountingThisFrame`

- 我的 `FTimerLatentAction_Delay` 只需要40字节就可以把所有任务做完了， 但 `FTimerDta` 有128字节， 大3.2倍!

- 一种 `一劳永逸` 的暂停定时器的替代方法：暂停一帧: `Rem::Latent::SetTimerPausedOneFrame`

- 还是熟悉的APIs: `Rem::Latent::PauseTimer`， `Rem::Latent::UnpauseTimer`， `Rem::Latent::SetTimerPaused`(我们之前见过吗?)， `Rem::Latent::StopTimer`， `Rem::Latent::FindTimerAction`

- 原生支持可重新触发的定时器: `Rem::Latent::ResetTimerDelay`， 同时支持按时间和帧数来延迟

- 调用次数补偿可以通过 `FTimerParameterHelper_Time::bMaxOncePerFrame` 选择性启用， (跟`TimerManager`一样)

- 目前还有27个比特是浪费的， 它们是未来的希望!

### 限制

- `Rem::Latent::FTimerHandle` 是32位的， 因为 `FLatentActionManager::AddNewAction` 只接受 `int32`， 但它之前在 `TimerManager`是 `uint64`

- 如果你在 `FLatentActionManager::ProcessLatentActions` 期间对同一个对象调用了 `Rem::Latent::SetTimerForThisTick` 可能会发生死循环， 这种情况下，请使用 `Rem::Latent::SetTimerForNextTick`

- 如果绑定对象当帧已经更新过， `Rem::Latent::SetTimerForThisTick` 就不会在对应的 `tick group` 更新， 这种情况下，考虑使用`Rem::Latent::SetTimerForNextTick`

- 为了简单起见，`TimeToDelay`， `LoopCount`， `InitialDelay` 都只有4字节的大小， 将来可能会考虑用上那些空闲的比特位

- 需要绑定的对象启用每帧更新，以支持给我们的定时器指定`tick group`

### 实例代码

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
