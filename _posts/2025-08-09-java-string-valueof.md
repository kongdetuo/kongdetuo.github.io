---
title: 【Java 迷惑 API】String.valueOf
date: 2025-08-07 21:00:00 +0800
categories: [Java]
tags: [java, 迷惑API]     # TAG names should always be lowercase
description: 一个简单的 toString 也能有坑？
---

众所周知，我是半路出家写 Java 的，所以呢对 Java API 不是很熟，这几天写 `toString` 的时候，觉得总是需要判空很烦，刚好呢，我知道 `String` 类有个 `valueOf` 方法，没有多想就用了，然后就出现了一些莫名其妙的 `"null"`，让人摸不着头脑。调试发现 `String.valueOf(null)` 返回的是 `"null"`，而不是 `null`。

看一眼源码：
```java
/**
 * Returns the string representation of the {@code Object} argument.
 *
 * @param   obj   an {@code Object}.
 * @return  if the argument is {@code null}, then a string equal to
 *          {@code "null"}; otherwise, the value of
 *          {@code obj.toString()} is returned.
 * @see     java.lang.Object#toString()
 */
public static String valueOf(Object obj) {
    return (obj == null) ? "null" : obj.toString();
}
```
注释倒是写的很明白，但谁能告诉我，`obj == null` 的时候为什么要返回 `"null"` 啊？从常理上讲，返回 `null` 或者是空字符串都很合理啊，这返回一个`"null"` 是闹哪样。

> 如果我传入一个 String，这个方法应该按原样返回，但并非如此。
{: .prompt-tip}

算了，换一个API，我貌似记得还有个 `Objects.toString`，去看看他的定义：

```java
public static String toString(Object o) {
    return String.valueOf(o);
}
```

嗯，很棒，行为非常一致。嗯。


那如果我想让空对象返回一个空字符串呢？用这个：

```java
public static String toString(Object o, String nullDefault) {
    return (o != null) ? o.toString() : nullDefault;
}
```

怪不得 Java 项目里标配一个 StringUtils，我还是自己写公共方法吧。

>我怀疑啊，Java 的设计原则里有一个【不提供合理默认值】的设定，前面那个 `BigDecimal.divide` 宁愿报错也不给个默认处理方式，这里给个默认值却是不合理的。嗯。很棒。
{: .prompt-info}
