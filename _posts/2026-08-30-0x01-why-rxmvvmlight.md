---
title: 【RxMvvmLight】0x01 为什么要做一个新的 MVVM 框架
date: 2026-08-30 22:50:00 +0800
categories: [RxMvvmLight]
tags: [mvvm, rx, reactive, r3, validation]     # TAG names should always be lowercase
description: 现在市面上已经有那么多mvvm框架了，为什么还要做一个？现在都是AI写代码了，你搞基础库有人用吗？AI会用吗？
---

众所周知，我的主力框架一直是 ReactiveUI，一直以来也用者挺好的，为什么突然要另起炉灶？

## 嫌弃

### ReactiveUI 不再基于 System.Reactive
事情说来也简单，最近 ReactiveUI 放弃使用 System.Reactive 了，这是一件好事，我对此双手赞成，也理解他们的做法，毕竟 System.Reactive 这库约等于没人管，名义上的维护者是个 Java 式的极端保守主义者，非常重视10年前甚至更久之前项目的无痛升级，我对此是很不理解，因为上一个放弃 System.Reactive 的是 Avalonia，在我看来 Avalonia、ReactiveUI 这种库是 System.Reactive 生命力的源头，说直白一点，这就是大客户，要优先满足大客户的需求，他倒好，花了三年时间做完了计划里三个月的活，成功耗尽了 ReactiveUI 的耐心，我对此无话可说，因为我的耐心也早没了。

那既然这事简单，并且赞成，那我嫌弃他啥呢？因为ReactiveUI 新做的 [ReactiveUI.Primitives](https://github.com/reactiveui/Primitives) 这个东西他起了一堆奇奇怪怪的名字，兼容性压力嘛，大家都有，但对我就不太友好了，我不在乎与 System.Reactive 的兼容性，所以我得搞一波迁移，烦，我得理解它与 System.Reactive 的对应关系，以及独特之处，烦。

### ReactiveUI 的历史包袱
这个说起来也是简单的，ReactiveUI 历史悠久，至少十几年了，这是好事，尤其对老用户来说，但对我来说就不一样了，不久前有一天，我日常写代码，写到 `this.WhenAnyValue(x=>x.Name, x=>x.` 的时候，智能补全罢工了，我也不知道这是临时性的罢工还是确实不行，反正我就 F12 看了一眼定义，嚯，好家伙，`WhenAnyValue` 有足足 76 个重载。我对此无话可说，老人老需求，新人新需求，他们得新老兼顾，就是对手写代码不太友好了。别的历史包袱我就不一一列举了，反正遇到了就很烦。

### `this.WhenAnyValue` 必须要写lambda
响应式编程是好的，但是这个lambda表达式真的不能省略吗？在过去的几年里，每一次写lambda我都在想，这玩意儿写着有点烦，能不能干掉他。这个东西看起来不能省略，但是我是真的想要省略它。

### 数据验证
一直以来，C# MVVM 平台的数据验证好像就没什么人在做，ReactiveUI.Validation 本身还停留在十年前的样子，而且特别不好用；FluentValidation 根本不在乎桌面平台，而且 FluentValidation 的那个 WithMessage 我真心觉得它是噪音，与上一条一样，我现在对语法噪音的容忍度是越来越低了；另有一个个人库 ReactiveValidation 早就不更新了；还有基于特性验证的流派比如 MvvmToolKit，这个简单用用还好，一旦需要关联验证/异步验证/防抖等立刻就会变得复杂起来。 看了一圈，没一个看起来好用的。

### System.Reactive 本身的缺陷
几年前，我看到了一个新的 Rx 实现 [R3](https://github.com/Cysharp/R3) 它解决了 Rx 用在MVVM上的最大缺陷：Rx 在异常时会取消订阅。这个我一直觉得是硬伤，因为GUI应用程序是强依赖事件的，如果可以，我希望在我手动取消订阅之前，这个订阅一直活着，而这也是 Rx 作为 Linq To Event 最大的缺陷，它并不能保证事件的语义。每次我写订阅的时候都小心翼翼的，生怕哪里报错导致订阅链炸掉，这导致我根本不敢真拿 Rx 当响应式来用，平常也就用用防抖，别的跟用事件基本没区别。这就比较不愉快了，根本就响应不起来。

## 冲动

我在想
- 我能不能做到更简洁的语法？去掉该死的lambda？
- 我能不能基于 R3 来做，让我能放心大胆的开始响应式？
- 我能不能做一个内置的好用的 Validation？

## 开始说服自己

### 现在都是AI写代码了，搞人体工学有意义吗？
简短的代码意味着更少的token消耗，有意义的，而且，有AI了，我撸框架岂不是更容易？

### 现在都是AI写代码了，AI会写你的框架吗？
根据我对AI的了解，给他一篇文档他就会了。而且，AI难道就会写新的 RxUI吗？还不是一样要给他文档。

### 能推广出去吗
难，毕竟现在已经是AI时代了，我能说服自己，很难说服别人。

### 能获得什么
至少一个快乐的周末

## 总结：我觉得可以搞
