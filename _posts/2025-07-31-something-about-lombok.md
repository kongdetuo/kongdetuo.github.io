---
title: Lombok 二三事
date: 2025-07-31 21:00:00 +0800
categories: [Java, plugin]
tags: [java, lombok]     # TAG names should always be lowercase
description: 记录一个野路子对 Lombok 的一些看法
---

## 0x01 谨慎使用 @Data

`@Data` 会自动生成 `getter`、`setter`、`toString()`、`equals()`、`hashCode()`、`canEqual()` 等内容，看起来十分美好，而且写起来也容易才5个字符。

但是需要注意：`@Data` 生成的类，在概念上类似 Java 17 的 `Record`，或者是领域驱动设计中的`值对象`, 这种类型并不适合在所有场景种使用。

其次，`@Data` 生成的类，是可读可写的，这在某些场景下，容易产生意外行为，比如放在 HashMap 种作为 Key，先放进容器再修改值，会导致 Key 丢失。

> 注意场景，不要因为他短就用他
{: .prompt-warning }

> 注意场景，不要因为他短就用他
{: .prompt-warning }

> 注意场景，不要因为他短就用他
{: .prompt-warning }

## 0x02 中立 @AllArgsConstructor

`@AllArgsConstructor` 会生成一个全参构造函数，我对这个构造函数本身倒是没有什么意见，用来简化依赖注入也是极好的，只是觉得有些鸡肋，参数少手写也不费事，数量多不如builder，貌似还不能排除字段。食之无肉，弃之有味。

## 0x03 推荐使用 @Getter @Setter

功能简单，容易理解，好用不解释，谁让 java 那么保守始终不加属性相关语法呢。

## ox04 推荐使用 @RequiredArgsConstructor

这个与上面的 `@AllArgsConstructor` 类似，但是 `@RequiredArgsConstructor` 只会给final字段生成构造函数，这完美适配依赖注入场景。

> 特性完美对应场景，那就是最优解。在 SpringBoot 框架中，我们使用 `@Autowired` 注解来注入依赖，会有警告让我们用构造注入，`@Resource` 用字段名作为bean名称简直离谱，使用构造注入是完美的，但是手写太长了，所以 `@RequiredArgsConstructor` 很适合
{: .prompt-tip }

## 0x05 谨慎使用 @Accessors

使用这个一般是为了链式调用，怎么说呢，可以省掉一个 `Builder` 类，但是链式调用跟接口搭配不佳，如果接口里定义了 void setter，那么要么链式失效，要么接口继承有问题，所以使用时需要确定场景，不要有继承，不要有接口的实体类是最好的。

当然如果不是链式调用，那么 `@Accessors` 也是可以用，只是场景少了，按需使用吧。

## 0x06 其他

用到再说