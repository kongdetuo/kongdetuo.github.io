---
title: 【DataGrid】如何为自动生成的列设置 CellTemplate
date: 2025-08-02 23:00:00 +0800
categories: [UI, Xaml]
tags: [avalonia, wpf, datagrid]     # TAG names should always be lowercase
---

`DataGrid` 是展示表格的控件，设置`AutoGenerateColumns="True"`后，`DataGrid` 会自动生成列。有时候这很方便，但是自动生成的列，很死板，基本全都是 `DataGridTextColumn，如果是` `int` `string` 可能还好，但稍微复杂一些的数据类型，比如 `DateTime`, 无论是展示还是编辑，效果都很难受。这里记录如何给自动生成的列设置模板。

## 0x01 AutoGeneratingColumn

当 DataGrid 自动生成列时，会触发 `AutoGeneratingColumn` 事件，在这里，可以控制生成的列。事件参数 `e` 中有一个 `DataGridColumn` 类型的 `Column` 属性，是可读可写的，构造一个 `DataGridTemplateColumn`，赋值给 `Column` 属性，就可以实现自定义列。

WPF 代码：

```csharp
private void DataGrid_AutoGeneratingColumn(object sender, DataGridAutoGeneratingColumnEventArgs e)
{
    if(e.PropertyType == typeof(DateTime))
    {
        DataTemplate cellTemplate = new DataTemplate();
        FrameworkElementFactory fatcory = new FrameworkElementFactory(typeof(TextBlock));
        Binding binding = new Binding();
        binding.Path = new PropertyPath(e.PropertyName);
        binding.StringFormat = "{0:yyyy-MM-dd}";
        fatcory.SetBinding(TextBlock.TextProperty, binding);
        cellTemplate.VisualTree = fatcory;

        DataTemplate cellEditingTemplate = new DataTemplate();
        FrameworkElementFactory factory1 = new FrameworkElementFactory(typeof(DatePicker));
        Binding binding1 = new Binding();
        binding1.Path = new PropertyPath(e.PropertyName);
        factory1.SetBinding(DatePicker.SelectedDateProperty, binding);
        cellEditingTemplate.VisualTree = factory1;
        var column = new DataGridTemplateColumn()
        {
            Header = e.PropertyName,
            CellTemplate = cellTemplate,
            CellEditingTemplate = cellEditingTemplate,
        };
        e.Column = column;
    }
}
```

> WPF 中这种简单的模板可以用 C# 实现，就懒得写 xaml 了，实际环境建议还是写 xaml
{: .prompt-tip }

Avalonia 的写法:
```csharp
private void DataGrid_AutoGeneratingColumn(object? sender, Avalonia.Controls.DataGridAutoGeneratingColumnEventArgs e)
{
    if (e.PropertyType == typeof(DateTimeOffset))
    {
        // 这里的 object 是 DataGridRow.DataContext，这里直接写 Object 走反射
        var cellTemplate = new FuncDataTemplate<object>((n, s) => new TextBlock()
        {
            VerticalAlignment = Avalonia.Layout.VerticalAlignment.Center,
            [!TextBlock.TextProperty] = new Binding(e.PropertyName)
            {
                StringFormat = "{0:yyyy-MM-dd}"
            }
        });

        var cellEditingTemplate = new FuncDataTemplate<object>((n, s) => new DatePicker()
        {
            VerticalAlignment = Avalonia.Layout.VerticalAlignment.Center,
            [!DatePicker.SelectedDateProperty] = new Binding(e.PropertyName)
        });

        var column = new DataGridTemplateColumn()
        {
            Header = e.PropertyName,
            CellTemplate = cellTemplate,
            CellEditingTemplate = cellEditingTemplate,
        };
        e.Column = column;
    }
}
```

## 0x02 Behavior

这种东西，如果直接写事件订阅，总感觉有些不 MVVM，建议封装成 Behavior 方便使用。