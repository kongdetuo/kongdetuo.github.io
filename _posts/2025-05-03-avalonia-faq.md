---
title: Avalonia 常见问题
date: 2025-08-03 11:40:00 +0800
categories: [Avalonia]
tags: [avalonia, ava-faq]     # TAG names should always be lowercase
description: 本文记录 Avalonia 的常见问题
---

> 这里不记录与AI相关的任何问题，鬼知道AI是怎么编的
{: .prompt-info }

## 0x00 我应该选哪个模板？

全都创建一个 Demo 跑一遍试试

## 0x01 Avalonia 可以只能使用 ReactiveUI 和 CommunityToolKit 吗？可以使用别的框架吗？

可以使用别的框架，只是模板里没有而已

## 0x02 ReactiveUI 和 CommunityToolKit 有什么区别吗，选哪个？

问这个问题的，还是选 CommunityToolKit 吧，他简单一些，虽然也没简单到哪去，至少各种视频文章多一些。选 ReactiveUI 的很容易一头钻进 Reactive 中研究半天用不上的东西。

## 0x03 为什么 pointerover 时设置背景色没有生效？

简短回答：因为你的样式优先级比较低，需要写详细一些才能覆盖主题中的样式:

```xml
<Style Selector="Button:pointerover /template/ ContentPresenter#PART_ContentPresenter">
    <Setter Property="Background" Value="Orange" />
</Style>
```

具体原因：Avalonia 开发组认为, 像是背景色之类的属性，是要保证响应性的，如果模板按下面这种写法：
```xml
<ControlTemplate>
    <ContentPresenter x:Name="PART_ContentPresenter"
                        Background="{TemplateBinding Background}"/>
</ControlTemplate>
```
当用户直接设置 `Background = "Red"` 后，鼠标放上去是无法变色的，这就失去了相应能力，是不好的。为了避免这种情况，开发组在自带的样式中写了比较高优先级的选择器，这也就导致了重写样式需要写这么一长串。

> 我个人倒是不喜欢这种设计，因为这属于多加了一条规则。不过呢，不管如何选择，都会有人不满，总归是要定下来的，无所谓了
{: .prompt-info }

## 0x04 为什么 DataGrid 没有显示

请检查 App.axaml 有没有导入样式。

例如：

```xml
<Application.Styles>
    <FluentTheme />
    <StyleInclude Source="avares://Avalonia.Controls.DataGrid/Themes/Fluent.xaml"/>
</Application.Styles>
```

## 0x05 ReactiveUI 可以和 CommunityToolKit 一起用吗？

可以

## 0x06 Avalonia 在 Linux 环境下无法双击打开

因为 .desktop 被识别成后缀名了，删除就好

## 0x07 Avalonia 可以内嵌 CEF 吗

我不知道

## 0x08 为什么 TextBox 的 KeyDown 事件没有触发？

因为 TextBox 会自行处理 KeyDown 事件，如果需要在 TextBox 之前处理事件，需要注册隧道事件：

``` csharp
// 注册隧道事件
target.AddHandler(InputElement.KeyDownEvent, OnPreviewKeyDown, RoutingStrategies.Tunnel);

void OnPreviewKeyDown(object sender, KeyEventArgs e)
{
    // 处理程序代码
}
```

[参考连接](https://docs.avaloniaui.net/zh-Hans/docs/get-started/wpf/tunnelling-events)

## 0x09 Avalonia 中有 Trigger 吗？

没有。下面有一些替代品：

- [选择器语法](https://docs.avaloniaui.net/zh-Hans/docs/reference/styles/style-selector-syntax)

- [样式类绑定](https://docs.avaloniaui.net/zh-Hans/docs/guides/data-binding/binding-classes)

- [基于Behavior的DataTrigger](https://github.com/wieslawsoltes/Xaml.Behaviors)

> 我tm今天就是要用Trigger，没有Trigger我要死了，我要杀人，全世界都应该因为没有这个Trigger爆炸 😂😂😂
{: .prompt-info }


## 0x0a 未完待续