---
title: 【RxMvvmLight】0x02 让 ViewModel 与 Reactive 和谐相处
date: 2026-08-31 22:50:00 +0800
categories: [RxMvvmLight]
tags: [mvvm, rx, reactive, r3]     # TAG names should always be lowercase
description: 包括 ReactiveUI 在内的大多数 MVVM 框架，他们的 VM 定义其实都是一样的，核心就是实现 INotifyPropertyChanged 接口。如果能让 VM 直接提供 IObservable<T>，想来写 Rx 代码的时候能更舒服一些。 
---

## 背景

包括 ReactiveUI 在内的大多数 MVVM 框架，他们的 VM 定义其实都是一样的，核心就是实现 INotifyPropertyChanged 接口。如果能让 VM 直接提供 IObservable&lt;T&gt;，想来写 Rx 代码的时候能更舒服一些。

既然我已经打算做一个新的 RxMvvmLight, 那就大胆一些，给这个二十年没有变化的类型加一点新东西，哪怕失败了，大不了回到传统设计呗。

## 说干就干

### 起名字
首先得给这个类型起个名字，否则后面我都不知道怎么叫它，总不好一直叫它 VM 基类吧，这个没什么好纠结的，就先叫他 `ObservableObject`

### 定义接口
目标已经定下，名字也定好了，现在需要想想怎么定义这个 ObservableObject 才能让他原生支持 Rx，而不是通过 INotifyPropertyChanged 中转。

跟着直觉走其实很快的，我都不知道合不合适，先定义了一个接口如下，反正还不知道能不能走通，就别想合不合适的事情了。
```csharp

public record PropertyValue(string PropertyName, object? Value);

public interface IObservableObject
{
    Observable<PropertyValue> Changing { get; }
    Observable<PropertyValue> Changed { get; }
}
```
Changing 和 Changed 其实就是 INotifyPropertyChanging, INotifyPropertyChanged 的等价物，唯一的区别就是这个 PropertyValue 比事件参数多带了一个当前属性值。

### 实现接口

人家 ReactiveUI 那么多年没做这个，肯定是有原因的，人家只是老，又不是傻。让我想想怎么才能实现这个接口。

#### 尝试通过 INotifyPropertyChanged 转接

这是个很自然的想法，既然总是会触发 INotifyPropertyChanged 接口，那我订阅这个接口，然后把值推出去，那不就实现了吗？不过这里有个小问题，那就是我没办法获取当前属性值。本着大力出奇迹的想法，我觉得，只要我稍微修改一下接口，加一个根据名字获取属性值的方法，比如这样：

```csharp
public interface IObservableObject
{
    Observable<PropertyValue> Changing { get; }
    Observable<PropertyValue> Changed { get; }
    object? GetPropertyValue(string name);
}
```
然后通过源生成器给每一个 VM 生成对应的 GetPropertyValue 实现，想来就可以通过订阅事件来实现推送 `Observable<PropertyValue>`。至于现在，我先用反射实现一个，看看效果。

这种简单的代码 AI 写的是又快又好，比我还好。

经过试用，逻辑是通的，这个逻辑太简单了，没道理不通，代码就不贴了，临时测试用的代码，用完就删了，我也不知道那时候怎么写的hhh

#### 成功了一半

于是，我现在面对以下问题：
- 怎么获取当前值
- 怎么把 object 转换回对应的类型

虽然我没细究过 ReactiveUI 的原理，但是我依稀记得，只要我订阅 this.WhenAnyValue ，马上就可以收到一个属性当前值, 我这个半成品能吗？不太行，我这个必须等待 INotifyPropertyChanged 事件，如果事件不触发，那我收不到当前值。另外，Changed 里全都是 object，这就不清真了，用object那不是又回到了c# 1.0 时代吗？

冥思苦想。

此时，我离走通整个链路就只差三个东西：
- 属性名：用于从 Changed 里过滤属性
- 属性值：用于提供初始值
- 属性类型：用于强制转换

有没有什么东西能同时提供这三个东西呢？

其实有的，比如表达式树，这都是传统成熟方案了，我都不用试就知道一定能成功，别的不说，人家 ReactiveUI 就是用的这个，那问题来了，如果要用表达式树，那我还搞什么 Changed 流啊，还是得想想别的办法。

冥思苦想。

还真让我找到一个方法，我依稀记得，好像在哪个版本的 C# 里，加了一个获取调用方表达式的编译器特性。问了一下AI，说是 `CallerArgumentExpression`, 那么假设哈，我定义这么一个方法
```csharp
    public static Observable<TValue> GetObservable<TVM, TValue>(this TVM vm, TValue value,
        [CallerArgumentExpression(nameof(value))] string name = "")
        where TVM : IObservableObject
    {

    }
```
我能获取什么？
- value 就是当前值，我没道理不能直接用啊，
- name 就是属性名，可以拿来过滤了，
- TValue 就是属性类型

有戏，写完
```csharp
    public static Observable<TValue> GetObservable<TVM, TValue>(this TVM vm, TValue value,
        [CallerArgumentExpression(nameof(value))] string name = "")
        where TVM : IObservableObject
    {
        var index = name.LastIndexOf('.');
        if (index > 0)
        {
            name = name[(index + 1)..].Trim();
        }

        return vm.Changed.Where(p => p.PropertyName == name).Select(p => (TValue)p.Value!).Prepend(value);
    }
```

试用一下，完美。

### 现在我有了新想法

既然我获取可以通过 `CallerArgumentExpression` 获取属性名，那如果我改造一下传统的 `OnPropertyChanged`, 也用这东西获取属性名，那是不是就可以直接在这里 `OnPropertyChanged` 里面推送新值了？如果可以，我就可以丢掉那个一看就很暴力的 `GetPropertyValue` 方法了？

说干就干，改造后 `ObservableObject` 定义如下
```csharp
public class ObservableObject : IObservableObject, INotifyPropertyChanging, INotifyPropertyChanged
{
    private readonly Subject<PropertyValue> changingSubject = new();
    private readonly Subject<PropertyValue> changedSubject = new();

    PropertyChangingEventHandler? propertyChangingEventHandler;

    PropertyChangedEventHandler? propertyChangedEventHandler;

    public Observable<PropertyValue> Changing => changingSubject;

    public Observable<PropertyValue> Changed => changedSubject;

    // 用显式接口实现，避免子类绕过 OnPropertyChanged 方法
    event PropertyChangedEventHandler? INotifyPropertyChanged.PropertyChanged
    {
        add
        {
            propertyChangedEventHandler += value;
        }

        remove
        {
            propertyChangedEventHandler -= value;
        }
    }

    event PropertyChangingEventHandler? INotifyPropertyChanging.PropertyChanging
    {
        add
        {
            propertyChangingEventHandler += value;
        }

        remove
        {
            propertyChangingEventHandler -= value;
        }
    }


    protected void OnPropertyChanging<T>(T value, [CallerArgumentExpression(nameof(value))] string propertyName = "")
    {
        propertyName = propertyName[(propertyName.IndexOf('.') + 1)..].Trim();

        changingSubject?.OnNext(new(propertyName, value));
        propertyChangingEventHandler?.Invoke(this, new(propertyName));
    }

    protected void OnPropertyChanged<T>(T value, [CallerArgumentExpression(nameof(value))] string propertyName = "")
    {
        propertyName = propertyName[(propertyName.IndexOf('.') + 1)..].Trim();

        changedSubject?.OnNext(new(propertyName, value));
        propertyChangedEventHandler?.Invoke(this, new(propertyName));
    }

    protected bool SetProperty<T>(ref T field, T value, [CallerMemberName] string propertyName = "")
    {
        if (!EqualityComparer<T>.Default.Equals(field, value))
        {
            OnPropertyChanging(field, propertyName);
            field = value;
            OnPropertyChanged(value, propertyName);
            return true;
        }
        return false;
    }
}

```

不知道我是不是第一个对 OnPropertyChanged 下手的人，反正吧我感觉我现在大逆不道。不过，管他呢，反正别人也不用。

这样一来，子类在写 `OnPropertyChanged` 的时候需要带上对应的属性，比如写 `this.OnPropertyChanged(Property)` 写 `this.OnPropertyChanged(value)` 或者 `this.OnPropertyChanged(nameof(Property))` 都是不行的。 AI 在单纯读代码的情况下也不一定能正确理解这个API怎么用，反正我记得它写半天没写对，然后在思维链里面嘀咕“这个API怎么如此难用”

我承认，这个是容易写错的，但我不打算改，因为写他的概率不大，大概率是修改属性顺便通知另一个属性变化的时候才会用，而在这种场景下, 原本的写法是 `OnPropertyChanged(nameof(Property))`, 新写法是 `OnPropertyChanged(Property)` 其实是更短了，只是写法不常规，但作为一个娱乐大于实用的项目来说，这个新API足够好用，留着挺好的。

## 就这么定下了

这个实现挺好的，我已经懒得想别的实现会不会更好，给这个 GetObservable 补上几个重载，理论上应该比 ReactiveUI 的 `this.WhenAnyValue` 好用一些，至少不会有76个重载，AOT 也没问题。

而有了这个 `GetObservable` 不管是简单订阅变化也好，还是做数据验证，总之 ViewModel 可以提供原生的响应式流，后续在做响应式数据验证的时候大概不用从监听属性变化开始了。

我感觉，这个设计应该挺有独创性的，它不一定是最好的，但理论上讲，它是目前为止把 Rx 和 VM 结合的最紧密的，我突然有点狂了，我宣布，从现在开始，这个 RxMvvmLight 定位是下一代 RxUI。

（小声逼逼：定位是定位，实现是实现，随时烂尾hhh）