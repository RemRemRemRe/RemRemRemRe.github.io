---
title: How to get MontageInstance correctly
date: 2025-08-05 01:00:00 +0800
categories: [Unreal Engine, Animation]
tags: [gameplay, animation, tutorial]    # TAG names should always be lowercase
lang: en
media_subpath: /assets/img/FindMontageInstance/
---

# Terms

## Montage

refers to `UAnimMontage` Object

## MontageInstance

refers to `FAnimMontageInstance` Object

# Get the MontageInstance

## GetInstanceForMontage - Find first matching MontageInstance

Finding from `UAnimInstance::MontageInstances`

## GetActiveInstanceForMontage - Find the active MontageInstance

Finding from `UAnimInstance::ActiveMontagesMap`

## [Simple Logic Flow](https://viewer.diagrams.net/index.html?tags={}&lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1&title=Untitled%20Diagram&dark=auto#R%3Cmxfile%3E%3Cdiagram%20name=%22Page-1%22%20id=%22l2bLqKPbx7WebuRzOWPT%22%3E5VnZkps4FP0aP9olVtOPXppMqrprPOmpmpmnKdmoQQlGjpC3fH0kEIss3E0G07hq8tCxjpaLzjnci2BkLbanTxTuomcSoHhkguA0spYj0zRs0+H/CeScI1PXzYGQ4kAOqoAX/ANJEEh0jwOUKgMZITHDOxXckCRBG6ZgkFJyVIe9kliNuoMh0oCXDYx19C8csChHPQdU+G8Ih1ER2QCyZwuLwRJIIxiQYw2yHkfWghLC8l/b0wLFgryCl3yef6W3vDCKEtZmQvC7CWc03X0ae398ffp3tZ/Tw1iqc4DxXm5YXiw7FwyggBMim4SyiIQkgfFjhc4p2ScBEmEAb1VjngjZcdDg4FfE2FmqC/eMcChi21j25jFFoKt7k1BK9nSD3tiQJT0CaYjYG+PsUgFuXUS2iNEzn0dRDBk+qNcBpYfCclxFM/8hmf4F1o3pXdAewDTK5heNFWQM0YQjHr8Y75eUeZdxw+1IuZy6IpiHNoFMMaYl7y+ZYGwbqEvklpGzLoQrL6ODlg+alrMgaJTzCa55dlQkgDEOBd8bTieiHDggyjBPPzPZscVBkKuNUvwDrrP1hN47saFsi8585Cw58koStiAxoVlECwDfB0Di0gOmXWoqAqGTQrrMoTJIlbnqar/hZ11aufwYTMDUkZ7vKvfFDPL6mqJehLU0XVcxPOvCxjGvRUKhY4QZetnBLD8deTlsSnINtGs301UijanqdatQ6FiVJqPAolpZ8sB1njtlsvtIZDesH3bLbNY1mXVivdzNh7LMuaTnv+X8rPGPaEycork81TuXZ9l6o8wYYhf8z00LTe3pqs9C4zx8bKEx9YQ0Mt2YCVLxgf8MWcZjDomkrzjE/b4nRcc4zTSe8QGmvTvl02R/sdAXtCWcK7kcv+J8RTUKh5XYgxc937f4v16K3Du5WRS5h+Jh4KyE7Gy8pkX7L4C25rcXxlPFPRVA2xu6ABrGXVRAJcfesBy6LXOu0TXndhLBbZMZ/0R0ixPIUNvkNaCvXXtwXwON08lkclckeWBwknTjPfN6BEP0OUkZTDYoLcy2poXRZhtxQ8hx6TPUM2r/VNrgIo8aDVS6DVS6fVFp6n67R6Icd3Ci9Fd2uucG52lqDc6TO0Qdrs5IEwCsi3OSZ79zUspaK0Qxp0A8GA9zfHI6lvJ2x6ep66hL9H188to8JGinnjs43tTf3X3gcefKG47accdz5cmkqzVMxRnjC2P0+IpPLzoXuVTTPMaJUC5llHwrvwjZmYko5xkTITC3ssDrptjV7uo13HxbkwTV7vQG6fkaItii/LAFGqwkDmTWnIvOlrX4xagxjZv8WDcVKDejvDqe+r5/q8qgiNv4yrKhLji9FQar6cSmJYJZgreVCVqlg/+FNXx/7kzBbazh2i2eQhu8Yf2HB3rerD615vmj+mBtPf4E%3C/diagram%3E%3C/mxfile%3E#%7B%22pageId%22%3A%22l2bLqKPbx7WebuRzOWPT%22%7D)

![MontageInstanceLifetime](MontageInstanceLifetime.jpg)

You can see that after `playing` a montage,

the newly generated montage instance can be obtained through any of the interfaces above,

After `stopping/blending out`, the montage instance, is no longer considered `active`,

After the playback is complete/the montage is actually `stopped`, it cannot be obtained by any means.




But in a short period of time, when the `same` montage is played `multiple` times, things start to get a little more complicated,

Suppose that when playing, `bStopAllMontages` is true, which is stopping the old playing montage.

For GetInstanceForMontage

![](GetInstanceForMontage.jpg)

Since the engine implementation is `return the first montage instance whose asset is the incoming montage`,

Then, if there are multiple montage instances blending out with the same montage asset, you can only get the `blending out one` through this interface, not the "active instances" you may need, because the active instance always `get added to the end of the array`. `GetActiveInstanceForMontage` should be used in this case

## GetMontageInstanceForID - Finding with InstanceID

In addition, engine provides this interface to support searching specific instance using `FAnimMontageInstance::InstanceID` which is unique within the process

# Suggestion

Because the `GetActiveInstanceForMontage` interface is queried through a `map` instead of traversing an array, it is the `fastest`.

If you just need to find a montage instance that is `playing and not blending out`/`active`, query it using the montage pointer and the `GetActiveInstanceForMontage` interface.

These is only one `active montage instance` at most, because montage and montage instance is `one-to-one` mapped with map.

![ActiveMontageMap](ActiveMontageMap.jpg)

For inactive montage searches, you can consider caching FAnimMontageInstance::InstanceID for accurate searches to avoid the problem of finding the wrong object when playing the same montage multiple times
