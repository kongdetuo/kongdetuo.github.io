---
title: 【RxMvvmLight】0x06 数据验证与 ViewModel
date: 2026-09-05 13:30:00 +0800
categories: [RxMvvmLight]
tags: [mvvm, rx, reactive, r3, data-validation]     # TAG names should always be lowercase
description: 这明明是一件非常简单的事情，为什么会变成这样！
---

上一篇实现了验证引擎核心，但一个合理的框架不是一堆零件，还是需要一些胶水把他们串联起来。

## IValidationObservableObject

这其实就是一个超级中转站，让 `RuleFor` 方法可以构造验证器所需的 `Observable<T>`，让验证器可以通知 View。

代码很短我就直接贴在这里

```csharp
public interface IValidationObservableObject : INotifyDataErrorInfo
{
    ReactiveValidator Validator { get; }
}

public class ValidationObservableObject : ObservableObject, IValidationObservableObject
{
    public ReactiveValidator Validator {get;} = new();

    public bool HasErrors => Validator.HasErrors;

    public event EventHandler<DataErrorsChangedEventArgs>? ErrorsChanged;

    public IEnumerable GetErrors(string? propertyName) => Validator.GetErrors(propertyName) : Enumerable.Empty<string>();

    protected ValidationObservableObject()
    {
        Validator!.ErrorsChanged.Subscribe(errors =>
        {
            ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(errors.PropertyName));
            OnPropertyChanged(nameof(HasErrors));   
        });
    }
}

public static class ValidationObservableObjectExtensions
{
    public static PropertyValidationBuilder<TValue> RuleFor<TVM, TValue>(
        this TVM viewModel,
        Func<TVM, TValue> expression,
        [CallerArgumentExpression(nameof(expression))] string propertyName = "")
        where TVM : IValidationObservableObject, INotifyDataErrorInfo, INotifyPropertyChanged
    {
        var name = RxMvvmLight.Helpers.PropertyNameHelper.ExtractNames(propertyName)[0];
        return new(viewModel.Validator, name, viewModel.ObserveChanged(expression, propertyName));
    }

    public static PropertyValidationBuilder<TValue> RuleFor<TVM, TValue>(
        this TVM viewModel,
        Observable<TValue> observable,
        string propertyName)
        where TVM : IValidationObservableObject, INotifyDataErrorInfo
    {
        return new(viewModel.Validator, propertyName, observable);
    }
}

```

## 这么简单的代码能出问题吗？

能的兄弟，能的，考虑以下VM
```csharp
public class LoginViewModel : ValidationObservableObject{

    public string UserName{get; set=>this.SetProperty(ref field, value);} = "";

    public LoginViewModel(){
        this.RuleFor(x => x.UserName)
            .NotBlank("用户名不可为空")
            .Subscribe();
    }
}
```

用户启动APP，会看到什么？明晃晃几个大字：用户名不可为空！

用户体验这块，谁会希望我一进去就一片红？

## 让我来解决这个问题，不过，先绕个圈子吧

我肯定不是第一个遇到这个问题的，让我看看前辈怎么解决的

### 学习 R3 的 `BindableReactiveProperty`

R3 其实自带了一个 `BindableReactiveProperty` 并且附带了验证功能，只是这种包装在 MVVM 里实在是难用，所以我一开始就没考虑过它。好看看它怎么解决的：

默认跳过第一个值，如果要强行验证第一个值，需要调用 `ForceValidate` 方法。

```csharp
Weight = new BindableReactiveProperty<double>()
    .EnableValidation(() => Weight)
    .ForceValidate();
```

好思路，我也这么做，在 `Subscribe` 方法中判断一下有没有调用 `ForceValidate` 方法，如果没有，就跳过第一个。

在多次尝试中，我发现，如果是不可为空的属性，我总是需要跳过第一条，这是符合预期的，但对于可空的属性，比如说可选的邮箱地址之类的东西，我就必须让它走一遍验证管道。

为什么？因为我的 `ReactiveValidator` 还提供了一个 `IsValid` 属性，如果它一直不走验证管道，那状态就一直是“未验证”，但问题是，默认的空值总是有效的，所以应该让它默认走一下验证管道，刷新一下状态。

那问题来了，这不是要求用户分别处理可空不可空的属性吗？不可空的，必须写一个 `Required` 可空的必须写一个 `ForceValidate`，呵，无用的仪式增加了。

## 分离是一切的解药

我其实绕了很多圈子，绕了有好几天，想过的思路比如说要用户显示设置验证器初始化完成之类的，反正都需要仪式，我不喜欢，用户可以搞他们的仪式，我就别给他们加仪式了。

突然灵光一黑：**我为什么非要把状态 和 消息展示 绑定在一起呢？**

你看啊，状态要求过一下验证管道，但过一下验证管道不等于我要展示错误啊对不对。

思路打开，我只需要阻断View展示消息就可以了，让验证管线始终运行，但只有属性真正变化之后才展示。

思路有了，经过一些尝试，我实现了新的 `ValidationObservableObject` 如下
```csharp
public class ValidationObservableObject : ObservableObject, IValidationObservableObject
{
    private bool errorDisplayActivated = false;

    public ReactiveValidator Validator => field ??= new();

    public bool HasErrors => errorDisplayActivated && Validator.HasErrors;

    public event EventHandler<DataErrorsChangedEventArgs>? ErrorsChanged;

    public IEnumerable GetErrors(string? propertyName) => errorDisplayActivated ? Validator.GetErrors(propertyName) : Enumerable.Empty<string>();

    protected ValidationObservableObject()
    {
        Validator!.ErrorsChanged.Subscribe(errors =>
        {
            if (this.errorDisplayActivated)
            {
                ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(errors.PropertyName));
                OnPropertyChanged(nameof(HasErrors));
            }
        });

        // 忽略注册验证器之前的变化
        this.Changed
            .Where(p => p.PropertyName != null)
            .Where(p => Validator.ContainsProperty(p.PropertyName!))
            .Take(1).Subscribe(x =>
            {
                this.errorDisplayActivated = true;
                ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(x.PropertyName));
            });
    }
}

```
说白了就是，默认验证器和 View 是断开的，如果有哪个注册过验证器的属性变化了，就放开限制。

这足够灵活了，而且用户啥都不用干，一切总是恰到好处，如果用户想一进去就一片红怎么办呢？好说，你注册早一点，在初始化属性之前注册验证器，或者注册完之后手动通知一下修改，就行了。

当然了，如果你并不认同我的所谓恰到好处，我推荐抄一份代码做一个自己的基类，这样不受我的影响。