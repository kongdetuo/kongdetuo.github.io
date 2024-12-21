---
title: 【学习 Avalonia MVVM 模式】 0x01 MVVM 核心接口 
date: 2024-12-21 13:00:00 +0800 
categories: [Avalonia, MVVM]
tags: [avalonia, mvvm, 教程]
---

本文介绍 `MVVM` 核心的两个接口：`ICommand` 和 `INotifyPropertyChanged`。

## 双向绑定

双向绑定是实现 MVVM 必不可少的部分，他是沟通 `View` 与 `ViewModel` 的桥梁，没有绑定，`ViewModel`定义的输入输出就无从实现。

但绑定并非魔法，再怎么厉害的绑定引擎也无法发现一个变量何时发生变化，除非修改之后告诉绑定引擎。刚好，有一个接口可以做到这个事情：
```csharp
public interface INotifyPropertyChanged
{
    event PropertyChangedEventHandler? PropertyChanged;
}
```

让`ViewModel` 实现`INotifyPropertyChanged`接口，这样在绑定时引擎会注册`PropertyChanged` 事件，当`ViewModel` 属性变化时，触发事件，绑定引擎接收事件并获取最新值，更新到 `View` 上，这样就实现了 `ViewModel` 到 `View` 的单项绑定。至于 `View` 到 `ViewModel` 的绑定，则通过依赖属性实现，依赖属性是一个包装过的属性，当发生变化时会触发相应的 `Changed` 事件。

> 可以模仿依赖属性包装一下 `ViewModel` 属性来实现通知，有些简易的 `MVVM` 框架有这种实现，但是这样用起来就需要解包，不算好使。

可以按如下代码实现属性变化通知：
```csharp
public class FooViewModel : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    private string text = "";

    public string Text
    {
        get { return text; }
        set
        {
            if (text != value)
            {
                text = value;
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Text)));
            }
        }
    }
}
```

当然，这是非常冗长的，相信没有人愿意写十几行代码只为定义一个属性，一个两个还好，如果写几十个属性怕不是要原地爆炸，所以先搞一个基类来简化 set 代码是十分有必要的：
```csharp
public abstract class BindableObject : INotifyPropertyChanging, INotifyPropertyChanged
{
    public event PropertyChangingEventHandler? PropertyChanging;
    public event PropertyChangedEventHandler? PropertyChanged;

    protected bool SetProperty<T>(ref T @field, T value, [CallerMemberName]string propertyName = "")
    {
        if(!EqualityComparer<T>.Default.Equals(@field, value))
        {
            PropertyChanging?.Invoke(this, new PropertyChangingEventArgs(propertyName));
            field = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
            return true;
        }
        return false;
    }
}
```
> 命名为 ``BindableObject`` 而不是 ``ViewModelBase`` 是因为不是所有的 ``BindableObject`` 都是 ``ViewModel``。在真实项目中，应该定义一个 ``ViewModelBase`` 作为所有 ``ViewModel`` 的基类。

通过 `SetProperty` 方法，可以稍微简化一下上面的 `ViewModel`
```csharp
public class FooViewModel : BindableObject
{
    private string text = "";
    public string Text
    {
        get { return text; }
        set { SetProperty(ref text, value); }
    };
}
```
这已经很短了，当然，如果不介意使用预览特性，可以使用C# 13 新加的 `field` 关键字更进一步的简化：
```csharp
public class FooViewModel : BindableObject
{
    public string Text
    {
        get;
        set => SetProperty(ref field, value);
    } = "";
}
```
这种写法有一些优点：

* 足够简短：哪怕全都放在一行也没有多长
* 不依赖框架：任何框架都可以这么写
* 不生成额外的字段：每一个额外的东西都是一个出错点，能少一个是一个
* 不依赖魔法：每种魔法写法都不通用，切换框架麻烦，新人不友好
* 扩展方便：可以直接在set里加逻辑，相比之下魔法一般不好扩展

当然成熟的框架可能会有不同的看法，比如 CommunityToolkit.MVVM 和 ReactiveUI 都提供了源生成器，通过标注的方式生成对应的代码。个人认为，这些概念并不适合初学者接触，可以先学会`MVVM`之后再接触，另外，我其实并不想与某个框架深度绑定，所以更倾向于通用写法。

> 要使用 `field` 关键字，需要设置语言版本为预览，编辑项目文件将 `LangVersion` 节点值改为 `preivew` 即可，如果没有这个节点，可以在 `PropertyGroup` 节点下添加子节点 `<LangVersion>preview<LangVersion>` 

> 这样写其实还是有很多样板代码，写起来不是很方便，但是用其他方法写也会有其他的样板代码，一样简单不到哪去。我的解决方案是使用[代码片段](https://learn.microsoft.com/zh-cn/visualstudio/ide/code-snippets?view=vs-2022)，我专门定义了 `rxcmd` 和 `rxprop` 两个代码片段，适用于 ReactiveUI，链接在[这里](https://github.com/kongdetuo/CodeSnippets)，如果需要其他框架的可以自行修改一下。

## 集合

上面的定义可以创建可绑定属性，但是数组/List这些类型是放在一起的多个对象，集合定义中并没有对应每一个元素的属性，那么对集合的增删改查的通知就需要接口 `INotifyCollectionChanged` 来实现。`INotifyCollectionChanged`接口可以通知新增/删除/重置等多种变化，由于 .Net 已经内置了一个 `ObservableCollection<T>`，这里就不再多说，只需要记住在VM 中不要直接使用数组/List，要使用`ObservableCollection<T>`

## 命令

在 MVVM 模式中，代表用户指令输入的是命令，对应的接口是 `ICommand`。

> 所谓指令输入就是用户发起了一个操作，比如确认、取消、登录、前进、后退等等。

看 `ICommand` 定义：
```csharp
    public interface ICommand
    {
        event EventHandler? CanExecuteChanged;
        bool CanExecute(object? parameter);
        void Execute(object? parameter);
    }
```

> 关于命令和事件：命令的关注点是做什么和可不可以做，怎么触发不重要；而事件则完全相反，只关注如何触发，不关心触发之后做什么。这种语义上的差别是使用命令的重要原因，解耦之类目标是顺带的。

命令的接口是非泛型的，而现在已经是2024年，非泛型的接口多多少少有些过时了，所以先扩展一下，搞一个泛型版本接口出来：
```csharp

public interface IDelegateCommand : ICommand
{
    void NotifyCanExecuteChanged();
}

public interface IDelegateCommand<T> : IDelegateCommand
{
    bool CanExecute(T? parameter);

    void Execute(T? parameter);

    bool ICommand.CanExecute(object? parameter)
    {
        if (parameter is T para)
        {
            return this.CanExecute(para);
        }
        return false;
    }

    void ICommand.Execute(object? parameter)
    {
        if (parameter is T para)
        {
            this.Execute(para);
        }
    }
}
```

> `NotifyCanExecuteChanged` 是个比较常用的方法，单独定义一个 `IDelegateCommand` 作为所有命令基类

这里使用了接口默认方法来实现 `ICommand` 接口的方法，重新定义了泛型版本的 `Execute` 和 `CanExecute` 方法，顺便添加一个 `NotifyCanExecuteChanged` 用来触发 `CanExecuteChanged` 事件。

这已经是一个完善的接口了，只要继承这个接口就可以很容易的实现确认Commmand、取消Command、保存Command、查询Command、登录Command了！是不是听起来就很麻烦？事实上，WPF刚刚出现的时候，开发者确实是这么写，时间一长就没有人继续这么搞了，于是 `MVVM` 模式的第二个核心类型就出现了：

```csharp
public class DelegateCommand<T> : IDelegateCommand<T>
{
    public event EventHandler? CanExecuteChanged;

    private readonly Func<T?, bool>? _canExecute;
    private readonly Action<T?> action;

    public DelegateCommand(Action<T?> execute)
    {
        this.action = execute;
    }

    public DelegateCommand(Action<T?> execute, Func<T?, bool> _canExecute)
    {
        this.action = execute;
        this._canExecute = _canExecute;
    }

    public bool CanExecute(T? parameter)
    {
        return _canExecute == null || _canExecute(parameter);
    }

    public void Execute(T? parameter)
    {
        action?.Invoke(parameter);
    }

    public void NotifyCanExecuteChanged()
    {
        CanExecuteChanged?.Invoke(this, EventArgs.Empty);
    }
}
```

逻辑十分简单，`Execute` 和 `CanExecute` 的内容都是通过 lambda 传入的，有了这个类型，创建特定`Command` 类型不再必要，简化到了极致。

> 不同的框架核心接口名称不同，比如 CommunityToolkit.MVVM 是 `IRelayCommand`，ReactiveUI 的是 `IReactiveCommand`

现在写一个小小的示例 ：
```csharp
public class FooViewModel : BindableObject
{
    public string Text
    {
        get;
        set => SetProperty(ref field, value);
    } = "";

    [field:AllowNull]
    public IDelegateCommand FooCommand => field ??= new DelegateCommand<object>(_ =>
    {
        this.Text = "Hello Command";
    });
}
```
> `Avalonia` 允许直接绑定方法，应该尽量使用 lambda 表达式来编写 `Command` 具体逻辑，避免创建额外方法造成混淆。还是那句话，每多一个东西，就多一个出错点。

> 由于C#的语法限制，命令要么在构造函数里初始化，要么在 get 里懒加载，都是比较繁琐的，好在 `field` 关键字、`??=`以及`属性的表达式主体`这三个特性配合使用可以写出比较简短的懒加载只读属性代码。当然现阶段的 `field` 关键字的空推断还不够完善，需要 `[field:AllowNull]` 标注来消除警告

> ### `Avalonia` 直接绑定方法的缺点：
> * 通用性差，如果想切换其他 UI 框架比如 WPF/UNO/Maui/CPF就需要修改 `ViewModel`
> * 不可以在 VM 中触发 ``CanExecuteChanged`` 事件
> * 异步命令支持不佳，异步命令通常具有额外逻辑，而内置的绑定不可能做太过灵活的支持（后面会简单介绍异步命令）
> * 容易与 `Command` 造成混淆，有些框架使用源生成器根据方法生成 `Command`，这样就会同时存在方法和 `Command` 有概率会错误绑定到方法上导致 `CanExecuteChanged` 失效

