---
title: 【RxMvvmLight】0x03 观察多个属性
date: 2026-09-05 07:50:00 +0800
categories: [RxMvvmLight]
tags: [mvvm, rx, reactive, r3]     # TAG names should always be lowercase
description: 鱼，我所欲也；熊掌，亦我所欲也。二者不可得兼
---

> 此文编写于实现 Validation 之后 {: .prompt-info }

## 缘由

在使用 RxUI 时，观察多个属性的手感总是让我感觉有些不太舒服，他是这样写的：
```
this.WhenAnyValue(x => x.Name, x => x.Password)
```
看起来是平平无奇，看起来是本该如此，那，手感不好在哪里呢？让我们继续往下写代码：

```
this.WhenAnyValue(x => x.Name, x => x.Password)
    .Subscribe(x =>
    {
        var name = x.Item1; // 这里丢失了属性名称，只能使用 Item1 代替
    });
```

这里手感不好的地方有两个
1. 在第一篇文章中提过的，输入第二个属性的 lambda 时，丢失了智能补全。
2. 在下游，失去了 `Name` 和 `Password` 的强命名。

老实说，第二个问题让我难以忍受，于是在实际使用多属性观察时，我往往是在 Subscribe 中直接访问属性，而不用 Observable 管道中传递下来的变量。

当然，RxUI 也是知道有这个问题的，所以他们提供了带选择器的重载，可以写如下的代码：
```csharp
this.WhenAnyValue(x => x.Name, x => x.Password, (Name, Password) => (Name, Password))
    .Subscribe(x =>
    {
        var name = x.Name; // 这里可以使用有意义的名称。
    });
```
我只能说，这本质上就是我自己重命名了一下元组，很丑，只有那种多个属性转成单个变量的场景才稍微好看一点点，但，给属性起名字这个过程依然是省略不掉的，我非常非常讨厌这一点。

## 那么我的 RxMVVMLight 对此有改善吗？

很遗憾，并没有，在上一篇文章中实现的极简语法在面对这种需求的时候，就变得跟 RxUI 一样了，最多也就是省略了一点点 lambda，解决了第一个手感不好的问题，而第二个难以解决：
```csharp
this.ObserveChanged(Name, Password) // 这里没有lambda
    .Subscribe(x =>
    {
        var name = x.Item1;  // 但这里依旧要使用 Item1
    });
```


在保留极简语法的前提下，必须通过反射才可以实现最流畅的语法，反射的版本用起来如下：

```csharp
this.ObserveChanged((Name, Password)) // 这里传入元组，通过反射构造后续的元组值
    .Subscribe(x =>
    {
        var name = x.Name;  // 这里可以消费带名称的元组
    });
```
可惜，反射的实现略微有些脏，我一直在纠结，要不要把它放进主线。于是我就这么放着了，直到很久之后我回过头来重新审视一下现有的设计，经过一些奇奇怪怪的联想，我发现可以使用稍微带有一点点噪音的语法实现这个需求：
```csharp
this.ObserveChanged(x =>(x.Name, x.Password)) // 这里传入一个返回元组的lambda
    .Subscribe(x =>
    {
        var name = x.Name;  // 这里可以消费带名称的元组
    });
```
是，在使用极简语法的前提下，我必须使用反射才能构造剩后续的元组值，那让一个编译期的lambda来提供这个元组就行了呗。这样可以把我现有的十来个重载压缩成一个：
```csharp
extension<TVM>(TVM vm) where TVM : INotifyPropertyChanged
{
    public Observable<PropertyChangedEventArgs> Changed =>
        Observable.FromEvent<PropertyChangedEventHandler, PropertyChangedEventArgs>(
                h => (sender, e) => h(e),
                h => vm.PropertyChanged += h,
                h => vm.PropertyChanged -= h);

    public Observable<TValue> ObserveChanged<TValue>(
        Func<TVM, TValue> accessor,
        [CallerArgumentExpression(nameof(accessor))] string expression = "")
    {
        var names = PropertyNameHelper.ExtractNames(expression); // 通过解析表达式字符串来提取属性名称

        return vm.Changed
            .Where(p => names.Contains(p.PropertyName))
            .Select(p => accessor(vm))
            .Prepend(accessor(vm));
    }
}
```

这个 `ObserveChanged` 方法可以支持观察单个属性，也可以观察多个属性：

```csharp
this.GetObservable(x => x.Name) // 单属性
    .Subscribe(x => {})

this.GetObservable(x => (x.Name, x.Password)) // 多属性
    .Subscribe(x =>{});
```

这直接解决了不断增长的重载需求。同时，通过扩展 `INotifyPropertyChanged`，可以给所有看起来像是 VM 的东西（比如Avalonia控件）添加 Changed 属性，这在实现VM自带changed这个需求的同时，降低了对特定接口的支持，使得 RxMvvmLight 可以作为辅助框架使用。

## 代价与权衡

常见的观察属性的需求有三种：
- 观察单个属性
- 观察多个属性
- 链式观察属性

这个方案直接统一了前两个，而对我来说，我失去的是：
- 看起来很魔法的语法 （我是真的很喜欢那个语法，但有了更一致的）
- 一个很特立独行的VM实现 （既然不用极简语法，那VM的特色也就不必保留）
- 每次写观察属性要多写三个字符

最终，对语法一致性的需求压过了我对极简语法的向往，极简语法赞，但他适用面有些窄，就让他留在我的记忆里吧。

至于链式观察，必须使用一些魔法比如反射/表达式树/源生成器，其中，反射我不太想用，表达式树本身不能使用元组，这跟观察多属性的实现冲突了，唯一剩下的方案是源生成器，不过感觉太麻烦了，于是决定暂时不做了，如果有需求，可以通过 R3 来实现，R3 其实已经很容易构造一个链式观察了。