---
title: 为什么会有卡顿
date: 2025-07-30 21:00:00 +0800
categories: [UI, quality]
tags: [ui, quality, 答群友问]     # TAG names should always be lowercase
---

在我混迹在各种技术群的时候，经常会有人问，为什么这个页面会卡顿？大多时候，我都会回一句：肯定是你在UI线程上有什么骚操作。大多时候的确如此，但总是有些其他情况，这里就稍稍总结一下我的看法。

## 0x01 什么是卡顿

这个其实是个挺重要的问题，如果连卡顿的定义都不统一，那么谈论卡顿的就是鸡同鸭讲。这里列举几种情况来帮助大家理解：

1. 点击按钮后，页面发白，无响应
2. 点击按钮后，页面发白，无响应，但一秒后可以响应
3. 点击按钮后，页面无响应，但点击其他地方可以响应
4. 点击按钮后，数据一秒一个往外蹦
5. 点击按钮后，等十秒后，数据一次性出来
6. 点击按钮后，出现进度条，等十秒后，进度条消失，数据一次性出来
7. 点击按钮后，三秒后出现进度条，等十秒后，进度条消失，数据一次性出来

其中 1 是卡死毫无疑问，2 是卡顿也毫无疑问，6 不是卡顿，其他的大概全都可以认为是卡顿。

为什么？

我们在操作时，总是需要一些反馈，比如点击按钮后出现数据，鼠标移动到按钮上后改变背景色等等。及时反馈就会给人流畅的感觉，但如果反馈过慢，那么就会产生卡顿感。一般来说300ms内的延时是可以认为是流畅的，超过三秒还没有反馈则会让人怀疑是不是已经卡死了，而如果本该一次性出现的数据一个个出来，也会让人感觉很卡顿。

这其实就是一个预期的问题，反馈应该是**及时且正确**的，反之就是卡顿，甚至是卡死。


## 0x02 原因：UI线程阻塞
   
这是最常见的原因，在 `async` `await` 关键字出现之前，正确实现异步编程是一件很麻烦的事情，所以早年的桌面软件比较容易阻塞 UI 线程，在我这不长的编程生涯中，也见过认为卡死才是正确反馈的前辈，这没法说

解决方法十分简单：使用异步函数或者其他方法将耗时操作移动到其他线程执行

## 0x03 原因：缺少进度提示

有些数据需要很长时间准备，如果没有进度提示，用户会认为软件卡了。我这里根据自己体验建议，如果需要 500 毫秒，就考虑加一个简单的菊花进度条展示；如果需要 2 秒以上，就必须加进度条；如果超过 30 秒，考虑加一个带步骤提示的进度条；超过三分钟的，考虑再加一个取消按钮。

同样的，进度条本身应该在500毫秒内完成，否则还是卡顿。

## 0x04 原因：逐个准备数据

有些使用 WinForm 的前辈，在转向 WPF 或者 Avalonia 的时候，可能会害怕数据量过大导致卡顿，于是手动处理数据，每15毫秒加一些数据这样。这种处理方法不能说毫无价值吧，只能说在 WPF 以及 Avalonia 中的虚拟化可以很大程度上解决这个问题，一次性处理几万个数据是毫无压力的。

## 0x05 我在想，要不要写一些实例