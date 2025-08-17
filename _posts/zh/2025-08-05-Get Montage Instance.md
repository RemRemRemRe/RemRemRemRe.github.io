---
title: 如何正确获蒙太奇的蒙太奇实例
date: 2025-08-05 01:00:00 +0800
categories: [Unreal Engine, Animation]
tags: [gameplay, animation, tutorial]    # TAG names should always be lowercase
lang: zh
media_subpath: /assets/img/FindMontageInstance/
---

# 蒙太奇简介

## 蒙太奇

指的是 `UAnimMontage` 对象

## 蒙太奇实例

指的是 `FAnimMontageInstance` 对象

# 获取蒙太奇实例

## GetInstanceForMontage 查找第一个匹配的蒙太奇实例

从 `UAnimInstance::MontageInstances` 中查找

## GetActiveInstanceForMontage 查找激活的蒙太奇实例

从 `UAnimInstance::ActiveMontagesMap` 中查找

## [简易流程图](https://viewer.diagrams.net/index.html?tags={}&lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1&title=Untitled%20Diagram&dark=auto#R%3Cmxfile%3E%3Cdiagram%20name=%22Page-1%22%20id=%22l2bLqKPbx7WebuRzOWPT%22%3E5VnZkps4FP0aP9olVtOPXppMqrprPOmpmpmnKdmoQQlGjpC3fH0kEIss3E0G07hq8tCxjpaLzjnci2BkLbanTxTuomcSoHhkguA0spYj0zRs0+H/CeScI1PXzYGQ4kAOqoAX/ANJEEh0jwOUKgMZITHDOxXckCRBG6ZgkFJyVIe9kliNuoMh0oCXDYx19C8csChHPQdU+G8Ih1ER2QCyZwuLwRJIIxiQYw2yHkfWghLC8l/b0wLFgryCl3yef6W3vDCKEtZmQvC7CWc03X0ae398ffp3tZ/Tw1iqc4DxXm5YXiw7FwyggBMim4SyiIQkgfFjhc4p2ScBEmEAb1VjngjZcdDg4FfE2FmqC/eMcChi21j25jFFoKt7k1BK9nSD3tiQJT0CaYjYG+PsUgFuXUS2iNEzn0dRDBk+qNcBpYfCclxFM/8hmf4F1o3pXdAewDTK5heNFWQM0YQjHr8Y75eUeZdxw+1IuZy6IpiHNoFMMaYl7y+ZYGwbqEvklpGzLoQrL6ODlg+alrMgaJTzCa55dlQkgDEOBd8bTieiHDggyjBPPzPZscVBkKuNUvwDrrP1hN47saFsi8585Cw58koStiAxoVlECwDfB0Di0gOmXWoqAqGTQrrMoTJIlbnqar/hZ11aufwYTMDUkZ7vKvfFDPL6mqJehLU0XVcxPOvCxjGvRUKhY4QZetnBLD8deTlsSnINtGs301UijanqdatQ6FiVJqPAolpZ8sB1njtlsvtIZDesH3bLbNY1mXVivdzNh7LMuaTnv+X8rPGPaEycork81TuXZ9l6o8wYYhf8z00LTe3pqs9C4zx8bKEx9YQ0Mt2YCVLxgf8MWcZjDomkrzjE/b4nRcc4zTSe8QGmvTvl02R/sdAXtCWcK7kcv+J8RTUKh5XYgxc937f4v16K3Du5WRS5h+Jh4KyE7Gy8pkX7L4C25rcXxlPFPRVA2xu6ABrGXVRAJcfesBy6LXOu0TXndhLBbZMZ/0R0ixPIUNvkNaCvXXtwXwON08lkclckeWBwknTjPfN6BEP0OUkZTDYoLcy2poXRZhtxQ8hx6TPUM2r/VNrgIo8aDVS6DVS6fVFp6n67R6Icd3Ci9Fd2uucG52lqDc6TO0Qdrs5IEwCsi3OSZ79zUspaK0Qxp0A8GA9zfHI6lvJ2x6ep66hL9H188to8JGinnjs43tTf3X3gcefKG47accdz5cmkqzVMxRnjC2P0+IpPLzoXuVTTPMaJUC5llHwrvwjZmYko5xkTITC3ssDrptjV7uo13HxbkwTV7vQG6fkaItii/LAFGqwkDmTWnIvOlrX4xagxjZv8WDcVKDejvDqe+r5/q8qgiNv4yrKhLji9FQar6cSmJYJZgreVCVqlg/+FNXx/7kzBbazh2i2eQhu8Yf2HB3rerD615vmj+mBtPf4E%3C/diagram%3E%3C/mxfile%3E#%7B%22pageId%22%3A%22l2bLqKPbx7WebuRzOWPT%22%7D)

![MontageInstanceLifetime](MontageInstanceLifetime.jpg)

可以看到，在`播放`一次蒙太奇时，

在播放之后，新生成的蒙太奇实例就可以通过上述任意接口获取到,

在触发`停止/开始混出`之后，蒙太奇实例，不再被认为是激活的（`Active`），

在播放完成/蒙太奇实际`停止`之后，无法通过任何方式获取到。




但在短时间，`多次`播放`同一`个蒙太奇时，事情开始变得复杂了一点，

假设，播放时，`bStopAllMontages`为true，即停止之前播的蒙太奇。

对于GetInstanceForMontage

![](GetInstanceForMontage.jpg)

由于引擎的实现是 `查找到第一个资源为传入蒙太奇的实例时，就返回`，

那么如果MontageInstances中有正在混出的同蒙太奇实例，通过此接口拿到的只能是在`混出的实例`，而不是你可能需要的“激活的实例”，因为激活的实例总是`加在数组末尾`。这时候应该使用`GetActiveInstanceForMontage`

## GetMontageInstanceForID 通过InstanceID查找

另外引擎提供了这个接口，来支持通过`FAnimMontageInstance::InstanceID`这个进程间的唯一ID来查找指定实例

# 建议

因为`GetActiveInstanceForMontage`这一个接口是通过`map`而不是遍历数组查询，速度`最快`，

如果只是需要查找`正在播放且没有混出`/`激活` 的蒙太奇实例，使用蒙太奇指针和`GetActiveInstanceForMontage`接口查询。

`激活的蒙太奇实例`最多只有一个，因为蒙太奇和蒙太奇实例是通过map`一一对应`的。

![ActiveMontageMap](ActiveMontageMap.jpg)

对于非激活蒙太奇的查找，可以考虑缓存FAnimMontageInstance::InstanceID，进行精确查找，避免多次播放相同蒙太奇时，可能找错对象的问题