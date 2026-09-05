---
title: 【RxMvvmLight】0x04 实现一个简易的 RxCommand
date: 2026-09-05 09:30:00 +0800
categories: [RxMvvmLight]
tags: [mvvm, rx, reactive, r3]     # TAG names should always be lowercase
description: 简单，但不可或缺
---

## 回忆

很久很久以前，在我刚开始使用 MVVM 模式的时候，定义一个 FooCommand 需要经历以下仪式：
- 声明一个 FooCommand 属性
- 声明一个 Foo 方法
- 声明一个 CanFoo 方法
- 在构造函数中写 `FooCommand = new Command(Foo, CanFoo);`
  
如果只是这样其实还好，有的是方法做到瞬发魔法，但如果你需要在 VM 中修改 FooCommand 的 CanExecute 就略微有一点恶心了。

举个例子：如果你希望用户输入 Name 和 Password 之后登录按钮自动亮起来，你通常需要这么写：

```csharp

private string name;
public string Name{
    get{return name;}
    set
    {
        this.SetProperty(ref name, value);
        FooCommand.NotifyExecuteChanged();
    }
}

private string password;
public string Password{
    get{return password;}
    set
    {
        this.SetProperty(ref password, value);
        FooCommand.NotifyExecuteChanged();
    }
}

private bool CanFoo(){
    return !string.IsNullOrWhitespace(Name) && !string.IsNullOrWhitespace(Password)
}

```

这都是上古代码了，但即便是到了今天，这些仪式依旧存在，即使使用源生成器，你也要在属性上标记`[NotifyCanExecuteChangedFor(nameof(FooCommand))]`，老实说我觉得这个特性只是把那一行通知挪了个位置，用一行更长的代码生成一个更短的代码，我也不知道为什么要这么做。

而且还有一点很气，如果有多个 Command 依赖某个属性，那个属性上就会贴多个`NotifyCanExecuteChangedFor`标签，巨长无比还不能折叠起来。

我甚至觉得 `NotifyCanExecuteChangedFor` 是一种滥用。

## 另一个回忆

与这种实现相比，RxUI 的实现就比较另类，它的 `CanExecute` 不是一个方法，而是一个 `IObservable<bool>`，在有变化的时候自动触发 `CanExecuteChanged`，这就允许我们把 Command 的依赖写到同一处：

```csharp
IObservable<bool> CanFoo(){
    return this.WhenAnyValue(x=>x.Name, x=>x.Password)
        .Select(x=>!string.IsNullOrWhitespace(x.Item1) && !string.IsNullOrWhitespace(x.Item2))
}
```

实质上来说，这种写法同样是把通知挪了一个位置，但他挪的比较巧妙，把所有会引发 CanExecute 变化的属性集中在一起，这样就比较好维护。

## RxCommand

所以对我来说，RxUI 的 `ReactiveCommand` 的最大价值其实就是这个 `IObservable<bool>`，它虽然还允许把 Command 作为 `Observable<T>`使用，但我觉得有些过度设计了（也可能是我不懂 Rx）。

所以在 RxMvvmLight 中，我决定实现一个简易的 RxCommand：基本就是一个普通的 `Command` + `IObservable<bool>` 自动通知，除此之外，也就是默认运行期间自动设置 CanExecute = false 吧，我认为 80% 的场景下这是正确的行为，至于剩下的行为，遇到再说。

这个实现真的很简单，直接贴代码吧：
```csharp

public abstract class RxCommand : ICommand, IDisposable
{
    public abstract event EventHandler? CanExecuteChanged;

    public abstract Observable<bool> IsRunning { get; }

    public abstract bool CanExecute(object? parameter);

    public abstract void Execute(object? parameter);

    public abstract Task ExecuteAsync(object? parameter);

    public abstract void Dispose();

    public static RxCommand Create(Action action, Observable<bool>? canExecuteObservable = null) =>
        new RxCommand<object>(_ => { action(); return Task.CompletedTask; }, canExecuteObservable);

    public static RxCommand Create(Func<Task> action, Observable<bool>? canExecuteObservable = null) =>
        new RxCommand<object>(_ => action(), canExecuteObservable);

    public static RxCommand<T> Create<T>(Action<T> action, Observable<bool>? canExecuteObservable = null) =>
        new RxCommand<T>(obj => { action(obj); return Task.CompletedTask; }, canExecuteObservable);

    public static RxCommand<T> Create<T>(Func<T, Task> action, Observable<bool>? canExecuteObservable = null) =>
        new RxCommand<T>(action, canExecuteObservable);
}

public class RxCommand<T> : RxCommand
{
    private readonly Func<T, Task> execute;
    private readonly IDisposable dis;
    private bool canExecute;
    private readonly BehaviorSubject<bool> runningSubject = new(false);

    internal RxCommand(Func<T, Task> execute, Observable<bool>? canExecuteObservable)
    {
        canExecuteObservable ??= Observable.Return(true);
        canExecuteObservable = canExecuteObservable.Prepend(true); // 保证有一个初始值
        this.execute = execute;
        this.dis = IsRunning.CombineLatest(canExecuteObservable, (isRunning, canExecute) => !isRunning && canExecute)
            .Subscribe(can =>
            {
                if (can != this.canExecute)
                {
                    this.canExecute = can;
                    CanExecuteChanged?.Invoke(this, EventArgs.Empty);
                }
            });
    }

    public override Observable<bool> IsRunning => runningSubject;

    public override event EventHandler? CanExecuteChanged;

    public override bool CanExecute(object? parameter) => canExecute;

    public override async void Execute(object? parameter) => await ExecuteAsync(parameter);

    public override async Task ExecuteAsync(object? parameter)
    {
        if (!canExecute)
            return;

        T param;
        if (parameter is T typed)
        {
            param = typed;
        }
        else if (parameter is null && default(T) is null)
        {
            param = default(T)!;
        }
        else
        {
            return;
        }

        try
        {
            runningSubject.OnNext(true);
            await execute(param);
        }
        finally
        {
            runningSubject.OnNext(false);
        }
    }

    public override void Dispose()
    {
        dis.Dispose();
        runningSubject.OnCompleted();
    }
}
```