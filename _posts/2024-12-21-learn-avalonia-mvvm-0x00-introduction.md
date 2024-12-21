---
title: 【学习 Avalonia MVVM 模式】 0x00 MVVM 介绍
date: 2024-12-21 12:00:00 +0800
categories: [Avalonia, MVVM]
tags: [avalonia, mvvm, 教程]
---

本系列文章面向从未使用过类似框架的 ``Avalonia`` 新人，旨在快速构建 ``MVVM`` 核心认知，掌握主流 ``MVVM`` 框架前置知识，降低框架使用门槛。

## 如何理解``MVVM``模式

``MVVM`` 是一种实现 UI 层的模式，M-V-VM 共同构成 UI 本身。关于UI，百科是这么说的：

> 用户界面（简称UI，亦称使用者界面）是系统和用户之间进行交互和信息交换的媒介，它实现信息的内部形式与人类可以接受形式之间的转换

有些文邹邹的，简单说就是 UI 接收用户的输入并向用户输出一些内容，而 ``MVVM`` 模式需要处理的也是这些。M-V-VM 各自负责 UI 的一个侧面，其中：

* `View`是 UI 的展示效果
* `ViewModel`是抽象的`View`， 是描述一个`View` 可以接收哪些用户输入，可以输出什么数据
* `Model`则是 UI 的数据，用来驱动`View`

必须注意 `MVVM` 仅仅是 UI 层模式，不属于 UI 层的逻辑同样不在 `MVVM` 模式应用范围之内。比如数据的增删改查就应该单独封装成服务供 `ViewModel` 调用。

### 举个例子

考虑一个简单的例子，一个搜索服务（比如百度）需要接收哪些输入，又要输出什么？

显而易见，接收的输入有：

* 一个搜索词
* 搜索（通常由回车键或者单独的搜索按钮触发）
* 打开指定结果（如果有结果）
    
而输出的部分则有：
* ~~广告~~
* 搜索结果列表

依据这些信息，我们可以定义一下 `ViewModel`（伪代码）：

```csharp
class SearchViewModel
{
    /// <summary>
    /// 搜索文本
    /// </summary>
    public string SearchText { get; set; }

    /// <summary>
    /// 搜索
    /// </summary>
    public void Search() { } 

    /// <summary>
    /// 搜索结果
    /// </summary>
    public List<SearchResultItem> Results { get; set; }

    /// <summary>
    /// 打开指定结果
    /// </summary>
    public void OpenResult(SearchResultItem result) { }
}
```
> 在这里，先使用 `方法` 代表用户的操作。

再考虑每个搜索结果具备哪些信息：
* 图标
* 标题
* 摘要
* 跳转链接信息

定义 ``Model`` 如下：

```charp
class SearchResultItem
{
    /// <summary>
    /// 图标
    /// </summary>
    public object Icon { get; set; }

    /// <summary>
    /// 标题
    /// </summary>
    public string Title { get; set; }

    /// <summary>
    /// 摘要
    /// </summary>
    public string Description { get; set; }

    /// <summary>
    /// 链接
    /// </summary>
    public Uri Uri { get; set; }
}
```

至于 `View` 就看各自的审美了，这里放一个最简单的代码结构：

```xml
	<DockPanel>
		<DockPanel DockPanel.Dock="Top">
			<Button Content="搜索" DockPanel.Dock="Right"/><!--搜索按钮-->
			<TextBox/><!--搜索文本-->
		</DockPanel>
		<ListBox><!--这里放搜索结果-->
			<ListBox.ItemTemplate>
				<DataTemplate>
					<DockPanel>
						<Image/><!--这里放图标-->
						<DockPanel>
							<TextBlock/><!--这里放标题-->
							<TextBlock/><!--这里放描述-->
						</DockPanel>
					</DockPanel>
				</DataTemplate>
			</ListBox.ItemTemplate>
		</ListBox>
	</DockPanel>
```

这就是 `MVVM` 拆分 UI 的逻辑。在我看来，`ViewModel` 应该是 ``MVVM`` 模式绝对的核心，因为他描述 UI 的本质目标，即与用户交互，`View` 负责展示具体的样式，`Model` 则是 UI 要显示的数据。

## 数据驱动与 MVVM 模式

ChatGPT 这么描述数据驱动：
> 数据驱动是一种编程和设计范式，它的核心思想是让数据来驱动程序的行为，而不是让代码来控制数据的状态。在数据驱动的程序中，数据的变化会自动反映到UI上，而用户对UI的操作也会自动反映到数据上。

可以看到，数据驱动的概念与双向绑定及其相似，而双向绑定是 `MVVM` 的重要构成部分，`MVVM`天然适合使用数据驱动范式，在使用 `MVVM` 模式时应该使用 `ViewModel` 的数据来驱动 `View` 状态变化，而不是手动编写代码修改 `View`。

## 关于 `MVVM` 框架

`MVVM` 框架本身并不神秘，只使用最核心的三五个简单类型就可以实现 `MVVM` 模式。当然常见的 `MVVM` 框架提供了更多的基础设施，比如 `Ioc`/`Messenger`/`EventBus`/`ReactiveX`/源生成器等，这些基础设施可以在各个环节中提高开发体验，这些内容将会在主要部分完成之后简单介绍。
