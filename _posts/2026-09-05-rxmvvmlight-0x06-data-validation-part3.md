---
title: 【RxMvvmLight】0x07 数据验证与本地化
date: 2026-09-05 15:30:00 +0800
categories: [RxMvvmLight]
tags: [mvvm, rx, reactive, r3, data-validation]     # TAG names should always be lowercase
description: 函数式才是真正的救赎之路，请无视其他信仰。160回合 |+3 时代得分
---

我讨厌无谓的仪式，从出生起，便是如此。

## 起因

一切要展示文本的地方，最终都需要本地化，验证消息同样如此。

现在问题来了，像下面这个规则，如何才能本地化呢？

```csharp
    extension(PropertyValidationBuilder<string> builder)
    {
        public PropertyValidationBuilder<string> MaxLength(int length, string message = "长度不得超过{0}")
        {
            return builder.Must(v => v?.Length <= length, string.Format(message, length));
        }
    }
```

这是我期待的形式，因为这样非常容易定义规则。只需要一个扩展方法，只需要三行代码。

那本地化怎么支持？
- 至少要支持硬编码吧，有的是项目不用本地化。
- 至少要支持一个 lambda 吧，没有 lambda，本地化没法搞。
- 我这个项目是用的错误码，你能支持吗
- 我这个项目是用的 `IObservable<string>` 本地化的，你能支持吗
- 我这个项目是用的字典，你能支持吗？
- 我这是用数据库管理的，你能支持吗？
- 我这...

头大了。

## 掉坑

我发现我丢掉 `WithMessage` 之后，没有一个合理的位置放这些重载，因为给不同的资源类型都提供一份重载是不现实的，这才是 FluentValidation 选择这种高噪音语法的原因：只需要定义少量的 `WithMessage` 就可以给无数的规则统一设置资源。

但我依旧不喜欢 `WithMessage`，因为如果这样：
- 每个规则几乎必须是一个类，而无法像我这样，写一个扩展就行
- 语法噪音
- 语法利用率-我造了多个方法，但是你总是只用其中一种？
  
所以我决定，主动给自己穿小鞋：在现有设计的基础上继续完善。

这几乎是不可能的，因为留给我的只有一个参数的位置，而我需要用这一个位置提供几乎无限的扩展，同时，要限制重载的数量，不能让它指数级增长。

### 尝试：只保留 Lambda

这个不用试，肯定行，但是它强制要求所有类型都走lambda，不清真，实在没办法了再考虑用这个。

### 尝试：自定义操作符

新版本 C# 允许在扩展中定义操作符，所以我可以给 string 定义一个一元加法运算，返回一个 resourceKey：
```csharp
    public record Key(string Name);

    extension(string message)
    {
        public static Key operator +(string name) => new (name);
    }


    // 使用：
    this.RuleFor(x => x.Name)
        .Required(+"required")
        .Subscribe();
```

这疑似有些太隐晦了。

而且这看起来就不好扩展。


### 尝试：扩展索引器

正在预览中的 C# 版本里，允许用户扩展索引器，所以理论上我可以写下面的代码
```csharp
this.RuleFor(x => x.Name)
    .Required()["required"]
    .Required()[()=>"required"]
    .Subscribe();
```
这本质上就是 `WithMessage` 优点就是短，缺点与 `WithMessage` 是几乎一致的，而且，索引器不是这么用的，很别扭。

### 尝试：隐式转换

我来定义一个 `MessageProvider` 类，把 `string` 和 `Func<string>` , 如果用户有自己的类型，也可以定义一个隐式转换，理论上这个方案的扩展性是无限的。

死于 lambda 无法一步到位隐式转换为 `MessageProvider`，此外，接口也不能隐式转换。


## 等等

你刚刚说 lambda 无法一次转换为 `MessageProvider` 对吧？那如果，我把一个lambda 传递给一个泛型方法，它是什么类型的？

```csharp
void Foo<T>(T value){

}

Foo(() => 1); // 这个lambda 是什么类型？
```
答案是 `Func<int>`

这是一个冷知识，来自 C# 10 的改进，现在 Lambda 可以有一个自然类型。

于是我有了一个大胆的想法：使用泛型消息，这应该可以提供无限的可能性，至少，它可以兼容 `string` 和 `Func<string>`

## 实现

只要有了方向，一切就顺理成章，首先，现在我的参数是 `TMessage`, 但我需要的是 `string`，所以，我需要一个注册中心，让我知道 `TMessage` 如何转换成 `string`:
```csharp
public static class MessageConverter
{
    class ConverterRegistry<T>
    {
        public static Func<T, string> Converter = null!;
    }

    static MessageConverter()
    {
        ConverterRegistry<string>.Converter = message => message;
        ConverterRegistry<Func<string>>.Converter = messageFunc => messageFunc();
    }

    public static void Register<T>(Func<T, string> converter)
    {
        ConverterRegistry<T>.Converter = converter;
    }

    public static string Convert<T>(T value)
    {
        if(ConverterRegistry<T>.Converter == null)
        {
            throw new InvalidOperationException($"No message converter registered for type '{nameof(T)}'.");
        }
        return ConverterRegistry<T>.Converter(value);
    }
}
```

ok，有了这个 `MessageConverter`，需要`string` 的地方只需要调用一下 `MessageConverter.Convert` 就可以获取字符串，比如开始举的这个例子：
```csharp
    extension(PropertyValidationBuilder<string> builder)
    {
        public PropertyValidationBuilder<string> MaxLength<TMessage>(int length, TMessage message)
        {
            // 这里入参是 TMessage
            // 但是 Must 的 TMessage 是 Func<string>
            return builder.Must(v => v?.Length <= length, () => string.Format(MessageConverter.Convert(message), length));
        }
    }
```

然后，我在最终注册规则的地方，统一把他们封装成 `Func<string>`。
```csharp
    public PropertyValidationBuilder<T> RegisterAsyncRule<TMessage>(Func<T, CancellationToken, Task<bool>> evaluate, TMessage messageProvider)
    {
        var rule = new AsyncTokenFuncRule<T> 
        { 
            Evaluate = evaluate, 
            MessageProvider = () => MessageConverter.Convert(messageProvider)
        };
        return AddRule(rule);
    }
```
这样，验证器面对的就是统一的 `Func<string>` 而不是乱七八糟的 `TMessage`。如此就实现了内置的两种消息类型。

## 增加第三种内置消息

作为一个框架，我当然是需要内置一些消息的，再简陋也得至少放一门语言进去，我这里选择使用枚举：
```csharp
public enum BuiltInMessage
{
    Required = 0,
    TextNotBlank = 100,
    TextMinLength = 101,
    TextMaxLength = 102,
    TextLength = 103,
    InvalidFormat = 201,
    InvalidEmailFormat = 202,
    InvalidUrlFormat = 203,
}
```
然后我修改一下内置消息注册：
```csharp
    static MessageConverter()
    {
        ConverterRegistry<string>.Converter = message => message;
        ConverterRegistry<Func<string>>.Converter = messageFunc => messageFunc();
        ConverterRegistry<BuiltInMessage>.Converter = builtInMessage => builtInMessage switch
        {
            BuiltInMessage.Required => "此字段为必填项",
            BuiltInMessage.TextNotBlank => "此字段不能为空",
            BuiltInMessage.TextMinLength => "长度不能少于{0}个字符",
            BuiltInMessage.TextMaxLength => "长度不能超过{0}个字符",
            BuiltInMessage.TextLength => "长度必须在{0}到{1}个字符之间",
            BuiltInMessage.InvalidFormat => "格式不正确",
            BuiltInMessage.InvalidEmailFormat => "邮箱格式不正确",
            BuiltInMessage.InvalidUrlFormat => "URL格式不正确",
            _ => throw new ArgumentOutOfRangeException("Invalid built-in message")
        };
    }
```
再给内置操作符加一些无参重载，此时缺点其实就暴露出来了，再怎么泛型自动推导，也得有参数才行，无参的只能写重载。好在重载很容易写：
```csharp
    extension(PropertyValidationBuilder<string> builder)
    {
        public PropertyValidationBuilder<string> MaxLength(int length) => builder.MaxLength(length, BuiltInMessage.TextMaxLength);

        public PropertyValidationBuilder<string> MaxLength<TMessage>(int length, TMessage message)
        {
            // 这里入参是 TMessage
            // 但是 Must 的 TMessage 是 Func<string>
            return builder.Must(v => v?.Length <= length, () => string.Format(MessageConverter.Convert(message), length));
        }
    }
```

## 演示一下如何集成第三方本地化库

这是对接第三方库 [Irihi.Lingua](https://github.com/irihitech/Irihi.Lingua) 的示例，它使用 `IObservable<string?>` 驱动 Avalonia View 本地化：
```csharp
MessageConverter.Register<IObservable<string?>>(x =>
{
    return ((LinguaObservableString)x).CurrentValue!;
});

MessageConverter.Register<BuiltInMessage>(x =>
{
   // 这里写内置消息的映射
});
```

是的，只需要这么注册一下就够了。

## 做一下收尾工作

本地化支持还需要支持刷新，这个十分简单，在 `ReactiveValidator` 的构造函数里把它放进一个静态弱引用列表里，然后提供一个静态刷新方法，用来遍历所有`ReactiveValidator`重新刷新一次消息，然后在你框架的语言变更事件中调用它就可以了。

这个属于脏活累活，我就不贴代码了。

## 延申

不知道你有没有注意到：我其实没有限制你注册 `string` 和 `Func<string>` 的转换器，这其实也是允许的。你可以注册一个 `string` 转换器，然后把它作为一个资源键处理，如果你使用资源键管理消息的话。

至于 `Func<string>` 就不太建议重写了，因为它才是真正的核心路线，当然你非要重写也不是不行，只是需要注意一点点细节罢了。

## 结语
最终，这个验证库成功的把 C# 用出了 F# 的风格：类型推导/闭包/自动转换，不得不说，像这种小东西，用函数式来处理是最方便的，不然为了封装一个判断建一个类还是太麻烦了。