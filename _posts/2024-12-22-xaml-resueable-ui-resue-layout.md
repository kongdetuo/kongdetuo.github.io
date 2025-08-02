---
title: 【Xaml 可复用 UI】 复用布局
date: 2024-12-22 14:00:00 +0800
categories: [UI, 可复用UI]
tags: [可复用UI, xaml, avalonia]
---

> 本文使用 `Avalonia` 作为示例平台，但是 `WPF` 等平台同样可以使用

## 问题
如果我们很多界面都很类似，那么是否在每个界面上都应该去定义一次布局呢？

思考如下页面：

![一个简单的数据维护页面](image/blog/20241222_data_page.png)

这是一个十分典型的数据展示页面，基本就只是提供一个增删改查。相信每个人都能在几分钟之内写出完整的布局代码，比如：
```xml
<Grid RowDefinitions="auto,*,auto,auto" Margin="4">
    <!--搜索条件，工具栏-->
    <DockPanel LastChildFill="False">
        <!--搜索条件栏-->
        <DockPanel>

        </DockPanel>
        <!--工具栏-->
        <DockPanel DockPanel.Dock="Right">

        </DockPanel>
    </DockPanel>

    <!--表格-->
    <DataGrid Grid.Row="1" Name="grid">

    </DataGrid>

    <!--分页控件-->
    <Panel Grid.Row="2">

    </Panel>

    <!--详细面板-->
    <Panel Grid.Row="3">

    </Panel>
</Grid>
```

好的，现在有一个问题，如果工程中有很多页面都有类似的布局，如何保证每一个页面整体风格是一致的？比如是搜索栏高度、页面边距、Grid边距、各种装饰品比如Logo如何摆放、位置大小、统一的动画效果等等。

首先很容易想到，可以使用静态资源来统一，比如工具栏高度，可以定义一个 `toolBarHeight` 的资源，每个页面都引用一下这个资源。从实战来说，这种方法确实可以达到效果，但是有一个问题：如何保证每一个页面都正确引用了资源呢？

> 题外话，我现在参与维护的系统中存在 4000 多个 xaml 文件，采用的正是这种方法，怎么说呢，显示效果大致是相同的，但是总有几个页面有些问题。

上面的是如何正确编写的问题，现在思考第二个问题：如果页面布局要做一定的修改，比如把详细信息面板隐藏了，或者放到其他位置，又或者每个页面增加一个统一的Logo，又该如何做？注意，可能有几百个页面。

## 实现

> 参考了 Asp.net mvc [Layout](https://learn.microsoft.com/zh-cn/aspnet/core/mvc/views/layout) 的概念。

在 Xaml 平台，要实现这种效果其实很容易，只需要一个自定义控件即可，定义十分简单，只需要几个依赖属性：

```csharp
class DataDetailLayout : TemplatedControl
{
    /// <summary>
    /// 条件区
    /// </summary>
    public static readonly StyledProperty<object> SearchContentProperty =
        AvaloniaProperty.Register<DataDetailLayout, object>(nameof(SearchContent));

    /// <summary>
    /// 工具栏
    /// </summary>
    public static readonly StyledProperty<object> ToolbarContentProperty =
        AvaloniaProperty.Register<DataDetailLayout, object>(nameof(ToolbarContent));

    /// <summary>
    /// 数据区
    /// </summary>
    public static readonly StyledProperty<object> DataContentProperty =
        AvaloniaProperty.Register<DataDetailLayout, object>(nameof(DataContent));

    /// <summary>
    /// 分页控件区
    /// </summary>
    public static readonly StyledProperty<object> PaginationContentProperty =
        AvaloniaProperty.Register<DataDetailLayout, object>(nameof(PaginationContent));

    /// <summary>
    /// 数据详情区
    /// </summary>
    public static readonly StyledProperty<object> DetailContentProperty =
        AvaloniaProperty.Register<DataDetailLayout, object>(nameof(DetailContent));

    public object SearchContent
    {
        get => this.GetValue(SearchContentProperty);
        set => SetValue(SearchContentProperty, value);
    }

    public object ToolbarContent
    {
        get => this.GetValue(ToolbarContentProperty);
        set => SetValue(ToolbarContentProperty, value);
    }

    public object DataContent
    {
        get => this.GetValue(DataContentProperty);
        set => SetValue(DataContentProperty, value);
    }

    public object PaginationContent
    {
        get => this.GetValue(PaginationContentProperty);
        set => SetValue(PaginationContentProperty, value);
    }

    public object DetailContent
    {
        get => this.GetValue(DetailContentProperty);
        set => SetValue(DetailContentProperty, value);
    }
} 
```
为他定义一个样式：
```xml
<Style Selector="layout|DataDetailLayout">
    <Setter Property="Template">
        <ControlTemplate>
            <Grid RowDefinitions="auto,*,auto,auto" Margin="4">
                <!--搜索条件，工具栏-->
                <DockPanel LastChildFill="False">
                    <!--搜索条件栏-->
                    <ContentPresenter Content="{TemplateBinding SearchContent}"/>

                    <!--工具栏-->
                    <ContentPresenter DockPanel.Dock="Right" Content="{TemplateBinding ToolbarContent}"/>
                </DockPanel>

                <!--表格-->
                <ContentPresenter Grid.Row="1" Content="{TemplateBinding DataContent}"/>

                <!--分页控件面板-->
                <ContentPresenter Grid.Row="2" Content="{TemplateBinding PaginationContent}"/>

                <!--详细面板-->
                <ContentPresenter Grid.Row="3" Content="{TemplateBinding DetailContent}"/>
            </Grid>
        </ControlTemplate>
    </Setter>
</Style>
```
在使用的地方只需要填充内容即可自动获得统一外观：
```xml
<layout:DataDetailLayout>
    <!--搜索条件栏-->
    <layout:DataDetailLayout.SearchContent>
        <DockPanel>
            <TextBox Watermark="姓名" Width="100" VerticalAlignment="Center" Margin="0,4,4,4"/>
            <Button Margin="4" Content="搜索"/>
        </DockPanel>
    </layout:DataDetailLayout.SearchContent>

    <!--工具栏-->
    <layout:DataDetailLayout.ToolbarContent>
        <DockPanel>
            <Button Content="新增" Margin="4"/>
            <Button Content="编辑" Margin="4"/>
            <Button Content="删除" Margin="4"/>
            <Button Content="刷新" Margin="4"/>
        </DockPanel>
    </layout:DataDetailLayout.ToolbarContent>

    <!--表格-->
    <layout:DataDetailLayout.DataContent>
        <DataGrid Name="grid">
            <DataGrid.Columns>
                <DataGridTextColumn Header="姓名" Binding="{Binding Name}"/>
                <DataGridTextColumn Header="年龄" Binding="{Binding Age}"/>
                <DataGridTextColumn Header="联系方式" Binding="{Binding Contact}"/>
            </DataGrid.Columns>
        </DataGrid>
    </layout:DataDetailLayout.DataContent>

    <!--分页控件面板-->
    <layout:DataDetailLayout.PaginationContent>
        <Panel>
            <DockPanel HorizontalAlignment="Right">
                <Button Content="1"/>
                <Button Content="2"/>
                <Button Content="3"/>
                <Button Content="4"/>
                <Button Content="..."/>
            </DockPanel>
        </Panel>
    </layout:DataDetailLayout.PaginationContent>

    <!--详细面板-->
    <layout:DataDetailLayout.DetailContent>
        <DockPanel VerticalAlignment="Top">
            <Panel Height="80" Width="65" Background="LightGray">
                <TextBlock Text="外貌不详"/>
            </Panel>
            <StackPanel Margin="8,4">
                <TextBlock Text="姓名：张三"/>
                <TextBlock Text="荣誉称号：法外狂徒"/>
            </StackPanel>
        </DockPanel>
    </layout:DataDetailLayout.DetailContent>
</layout:DataDetailLayout>
```
> 这个就是示例图的代码

> 每当用布局的时候都会觉得 Xaml 的语法是真的长

## 其他应用

通过 `Layout` 也可以做一些别的操作：

1. 作为样式容器，在模板中编写的样式同样会应用到子元素中，这样我们可以让子元素有统一的默认外观
1. 把一部分内容显示到别的位置，比如说，可以提供一个标题属性，显示到标题栏上，又或者可以把详情内容放到弹窗中

## 改进

在上面的示例中，所有属性都是 `object` 类型，这样一般是足够的，但是有些情况，比如工具栏，其实可以提供一个 `IEnumerable` 类型来接收多个子元素，这样用起来会更加简单。

如果某个区域可能会显示多种内容，在 `Avalonia` 平台，还可以通过 `DataTemplates` 来展示不同的 VM 数据，在`WPF`平台可能需要提供一个 `DataTemplateSelector` 来选择不同的模板。

## 调试
> 这一段仅适用于 `Avalonia`

上面的模板在正常运行时，如果按 F12 查看逻辑树，会发现根本没有子节点，这其实是因为`Avalonia`不认识这个控件，但是这个我没看出来应该怎么弄，等后面再说吧。