---
title: Java BigDecimal 踩坑记
date: 2025-07-28 21:00:00 +0800
categories: [语言, Java, 踩坑]
tags: [java]     # TAG names should always be lowercase
description："本文记录 Java BigDecimal 的一些坑"
---

本文记录 Java BigDecimal 的一些坑

1. BigDecimal 的构造函数

    不要使用```BigDecimal(double value)``` 构造函数，使用 ```BigDecimal(String value)``` 构造函数，因为`BigDecimal(double value)`的行为是不可预料的，无法确定构造的结果具体是多少。
    
    以下内容来自文档：
    有人可能会认为在 Java 中编写 `new BigDecimal(0.1)` 会创建一个恰好等于 `0.1` 的 `BigDecima`l（未缩放值为 `1`，缩放位数为 `1`），但实际上它等于 `0.1000000000000000055511151231257827021181583404541015625`。这是因为 0.1 无法精确地表示为 double（或者，实际上，无法表示为任何有限长度的二进制分数）。

    反正我是不理解为啥要保留这么一个构造函数，完全不能用嘛

2. BigDecimal 的相等运算

   - 不能使用 `==` 运算符，老生常谈了, 因为 `==` 运算符比较的是两个对象的引用，而不是对象的内容
   - 不能使用 `equals()` 方法，因为 `new BigDicamal("1.0").equals(new BigDecimal("1.00"))` 返回 `false`
   - 应当使用 `compareTo()` 方法比较，这种方法可以正确处理不同精度的数字
   - 当然最应该使用的是各种 Util 类提供的方法

    相等运算和比较运算返回值不一样可还行

3. BigDecimal 的除法

    `new BigDecimal(1).divide(new BigDecimal(3))` 会报错，因为这个结果无法用有限小数位数表示，正确写法应该是 `new BigDecimal(1).divide(new BigDecimal(3), 10, RoundingMode.HALF_UP)`

    既然如此，第一个 `divide` 方法的用处是什么？

4. 缓存
   
   BigDecimal 也自带了几个缓存值，这样，通过 `BigDecimal.valueOf(1)` 可以获取一个缓存值，可以避免重复创建对象。虽说相比 `Integer`、`Long` 这种缓存上百个的值，BigDecimal 的缓存少很多，但蚊子再小也是肉，推荐用 `valueOf` 构造。

   同样的，这也会造成部分值可以通过 `==` 比较，应该没人还好在意这个问题了吧