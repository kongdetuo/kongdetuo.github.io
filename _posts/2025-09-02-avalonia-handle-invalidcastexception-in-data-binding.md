---
title: 【Avalonia】处理数据绑定中的 InvalidCastException 异常
date: 2025-09-02 22:50:00 +0800
categories: [Avalonia]
tags: [avalonia, validation]     # TAG names should always be lowercase
#description: 
---

在 Avalonia 日常开发中，TextBox 绑定到 int 类型属性是十分常见的情况，但是一旦输入了无效数字，界面上马上就会显示错误消息: `System.InvalidCastException: Could not convert '' (System.String) to System.Int32.`。这种错误消息完全是不合理的，没有本地化不说，也不该展示完整的类型。
 
我们知道，Avalonia 的异常信息是存储在 `DataValidationErrors.Errors` 属性中的，那么解决这个问题的办法大概率也在这里，观察源码可以发现有一个 `ErrorConverterProperty`, 这就是解决这个问题的关键。

考虑如下界面：
```xml
	<StackPanel>
        <TextBox Name="tb1" Text="{Binding IntValue}"/>
	</StackPanel>
```
在xaml.cs 文件中编写如下代码：
```csharp
    this.tb1.SetValue(DataValidationErrors.ErrorConverterProperty, o =>
    {
        if (o is InvalidCastException)
            return "无效输入";
        return o;
    });
```

运行可以发现异常消息已经变成了 `无效输入`。

## 封装一下

为每一个TextBox 单独编写显然是十分低效的做法，我们可以创建一个静态属性，然后通过 Style 为所有控件开启添加这个补丁。
```xml
    <Style Selector=":is(Control)">
        <Setter Property="DataValidationErrors.ErrorConverter" 
                Value="{x:Static ve:DataValidation.ErrorConverter}" />
    </Style>
```

```csharp
    public class DataValidation
    {
        public static Func<object, object> ErrorConverter { get; } = o =>
        {
            if (o is InvalidCastException)
                return "无效输入";
            return o;
        };
    }
```

> 只显示无效输入还是过于粗暴了，可以为多种类型定义不同的转换器，并按需使用。
{: .prompt-tip }

> 如果需要更强大的功能，这种单纯的转换就不够了，可以通过封装Behavior实现更多功能，这已经超出本文范围，这里不做描述。
{: .prompt-tip }

## 走过的弯路

没仔细看定义的时候是这么写的：
```csharp
    DataValidationErrors.ErrorsProperty.Changed
        .Subscribe(p =>
        {
            var arr = p.NewValue.GetValueOrDefault<object[]>();
            if (arr != null && arr.Length > 0 && arr[0] is InvalidCastException)
            {
                arr[0] = "无效输入";
            }
        });
```
你还别说，在测试工程内真的可以运行，而且这种方法可以获取控件，比单纯的 Converter 强多了，唯一的问题是，看起来过于野路子，万一开发组把内部类型改成一个只读列表，这种方法就炸了。

但好歹也花了我几分钟时间，记一下吧，万一用到了呢。
