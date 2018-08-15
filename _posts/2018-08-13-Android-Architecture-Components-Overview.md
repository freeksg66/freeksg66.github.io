---
layout:     post
title:      Android Architecture Components
subtitle:   Android开发文档翻译
date:       2018-08-13
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

## 概述

Android架构组件是[Android Jetpack](https://developer.android.google.cn/jetpack/)的一部分。它们是一个库的集合，这个集合可以帮助你设计鲁棒的，可测试的，和持续性强的app。从管理UI组件生命周期和处理数据持久性的类开始。

* 使管理你的app的生命周期变得容易。新的[lifecycle-aware](https://developer.android.google.cn/topic/libraries/architecture/lifecycle)组件帮助你管理你的activity和fragment的生命周期。在设备配置改变时不丢失数据，回避内存泄漏，并且容易的加载数据到你的UI中。
* 使用[LiveData](https://developer.android.google.cn/topic/libraries/architecture/livedata)去构建数据对象，当数据发生改变时，通知相关views进行更新。
* ViewModel存储UI相关的数据，这些数据不会再app发生旋转时销毁。
* Room是一个SQLite对象映射库。使用它去回避样本文件编码（boilerplate code ），并且容易地将SQLite表数据转化为Java对象。Room提供了对SQLite语句的编译时间检查，并且可以返回RxJava、Flowable和LiveData observables。


## 添加这些组件到你的项目中

{% highlight java %}
{% endhighlight %}