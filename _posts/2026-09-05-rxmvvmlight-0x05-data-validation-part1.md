---
title: 【RxMvvmLight】0x05 数据验证-核心引擎
date: 2026-09-05 12:30:00 +0800
categories: [RxMvvmLight]
tags: [mvvm, rx, reactive, r3, data-validation]     # TAG names should always be lowercase
description: 内含虚构情节，请注意甄别
---

> 完整代码比较长，这里只展示关键决策，详细实现请看[RxMvvmLight](https://github.com/kongdetuo/RxMvvmLight) 
{: .prompt-tip }

该吐的槽都吐过了，这里直接开始

### 分析需求

数据验证其实很简单，说白了就是判断一下，false 就返回一条消息，但作为一个框架，就不能如此儿戏，让我好好回忆一下我见过的验证需求：
1. 普通校验：必填/长度/大小之类的
1. 异步校验：校验用户名是否可用之类的
1. 防抖：因为有异步校验，你不能每输入一个字符就去查数据库
1. 条件：如果用户没填，那就算了，如果他填了，就必须输入真正的圆周率前100位
1. 联动：如果用户填了A，那B就得填圆周率
1. 提示状态1：如果属性正在验证，要给用户一个提示动画，如果属性验证通过，要给用户显示一个绿色的叉叉
1. 提示状态2：如果用户该填的都填了，按钮亮起来（比如登录按钮）
1. 本地化：只要是展示文字的地方，都得做本地化
1. 语法舒服：API 设计要符合人体工学，要容易扩展自定义规则

由于内容众多，初版不考虑本地化（超绝挖坑中）

### 设计：语法
语法是一个库好不好用最直接的指标，像 MvvmToolkit 使用特性来声明验证规则就属于是易用性拉满（如果不考虑自定义/本地化的话）。所以这里是重中只重，值得专门处理。

我现在的想法是，FluentValidation 的语法挺好的，值得抄过来，不过他的 `WithMessage` 方法实在是有些噪音，不知道他们为什么要设计这个（那当然是有原因的，不用这个后面等着哭吧）。所以设计语法如下：

#### 基本语法
```
this.RuleFor(x => x.Name)
    .Required("此项必填哦~")
    .Subscribe();
```
除了最后的 `Subscribe` 不太常规之外，这是一个典型的 Builder 模式，但在 Rx 环境下到处都是 `Subscribe`，所以这个噪音应该还可以接受。

#### 防抖语法
```
this.RuleFor(x => x.Name)
    .Required("此项必填哦~")
    .Debounce(300)
    .Must(CheckService, "前面的区域，以后再来探索吧！") 
    .Subscribe();
```
防抖规则 `Debounce` 为后续的所有规则施加一个防抖操作。

#### 条件语法
```
this.RuleFor(x => x.Name)
    .When(x=> x.Name?.Length > 3) // 当用户填写一定长度后，开始校验可用性
    .Debounce(300) // 按条件防抖
    .Must(CheckService, "前面的区域，以后再来探索吧！") // 按条件防抖验证
    .Subscribe();
```
与 FluentValidation 不同，这个 `When` 为后续操作施加一个条件，作用域延续到下一个 `When`，如果没有下一个 `When` 则延续到结束。这个设定允许多个条件共享同一个条件，从而允许按条件为一个异步验证增加防抖操作。

这里不允许 `When` 嵌套，我认为这样就足以满足绝大部分需求了。

> 那如果连续写了两个 `When` 呢？我会把它合并成一个 `When`，这样比较符合直觉，虽然我不觉得会有人这么写。
{: .prompt-tip }


> 在这种设计下，自定义规则就只需要给 Builder 编写一个扩展方法，应该属于是个人都会做了
{: .prompt-tip }

oK, 我对此表示满意，进度 1/9

### 设计同步/异步/防抖/条件 底层逻辑

上面的语法并非凭空产生，他是经过我几个小时的灵光一黑，才做出来的，它依赖以下假设：
1. 所有验证规则按顺序执行
2. 所有验证规则必须是 条件/防抖/异步 中的一种
3. 条件规则内部可以包含一组规则

这样假设我有一个规则组，我可以使用如下逻辑(伪代码)实现条件/防抖/异步验证的自由组合:
```csharp

async Task Evaluate(IReadOnlyList<IRule<T>> rules, CancellationToken token)
{
    foreach (var rule in rules)
    {
        // 如果有新验证到来，本次验证结束，用来实现防抖
        if (token.IsCancellationRequested)
        {
            break;
        }

        if (rule is DebounceRule<T> debounceRule)
        {
            await Task.Delay(debounceRule.Duration);
        }
        else if (rule is ConditionalRule<T> conditionalRule)
        {
            // 递归遍历所有规则
            if (conditionalRule.Condition(value))
            {
                await Evaluate(conditionalRule.InnerRules, token);
            }
        }
        else if (rule is AsyncTokenFuncRule<T> asyncFuncRule)
        {
            var isValid = await asyncFuncRule.Evaluate(value, token);
            if(!isValid){
                // 收集结果
            }
        }
    }
}
```

把实际执行的规则拆分为这三类，应该已经满足了大部分单属性验证需求。

规则类型定义如下：
```csharp
internal interface IRule<T>
{
}

internal sealed class DebounceRule<T>(TimeSpan duration) : IRule<T>
{
    public TimeSpan Duration { get; } = duration;
}

internal sealed class AsyncTokenFuncRule<T> : IRule<T>
{
    public required Func<T, CancellationToken, Task<bool>> Evaluate { get; init; }

    public required string Message { get; init; }
}

internal sealed class ConditionalRule<T>(Func<T, bool> condition) : IRule<T>
{
    public Func<T, bool> Condition { get; } = condition;

    public List<IRule<T>> InnerRules { get; init; } = [];
}
```

ok，进度 5/9

### 设计：多属性关联验证

上面的设计针对单属性几乎已经完美，单如果一个属性依赖另一个属性怎么办呢？

对此，我决定简化处理，增加一个特定操作符：`DependsOn(Observable<T>)` 这样当其他属性变化时，再次触发一次单属性的验证。至于说依赖其他属性的验证，那就直接在验证操作符内取属性就好了：

```csharp
this.RuleFor(x => x.A)
    .DependsOn(this.ObsereChanged(x=>x.B)) // 这里由于Builder 设计的过于通用，只依赖 Observable<T> 不依赖 VM 而无法简化
    .Must(x=> x < this.B, "A 必须 小于 B") 
    .Subscribe();
```

ok, 进度 6/9

### 设计：与状态相关的需求

仔细看看，其实提示状态的一共就两种：
- 每个属性的状态：未验证/验证中/验证通过/验证失败
- 整个验证器的状态：全部属性是验证通过则验证器视为验证通过

ok，基于以上信息，以及本项目的 Rx 背景，初步设计 `ReactiveValidator` 如下(伪代码)：

```csharp
enum ValidationState
{
    /// <summary>
    /// 初始值，尚未验证
    /// </summary>
    NotValidated,
    /// <summary>
    /// 正在评估
    /// </summary>
    Validating,
    Valid,
    Invalid
}
class ReactiveValidator
{
    // 用来对接 INotifyDataErrorInfo 接口，避免由 VM 管理 Errors 数据
    public bool HasErrors
    public Observable<PropertyErrors> ErrorsChanged { get; }

    // 给 Command 用，用 RxCommand 的话，只需要写 RxCommand.Create(()=>{}, Validator.IsValid) 就可以完成接线
    public Observable<bool> IsValid { get; }

    // 给VM提供一个获取单个属性验证状态方法
    public Observable<ValidationState> GetState(string propertyName)
}
```
设定：当一个属性变化后，立刻更新状态为验证中，验证完成（成功/失败）后调整为对应状态，防抖期间状态为验证中，当所有验证通过之后，设置验证器状态为 Valid，并通过属性 IsValid 暴露。

好，进度 8/9

由于篇幅关系（当时的周末也结束了）这个验证器实现到此为止，跑通了，很成功。

