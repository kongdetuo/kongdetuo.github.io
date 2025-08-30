---
title: 【Avalonia】如何将 Avalonia 作为插件UI
date: 2025-08-30 16:00:00 +0800
categories: [Avalonia]
tags: [avalonia, cad, plugin-ui]     # TAG names should always be lowercase
description: 在大型软件（如CAD）的二次开发中，经常使用 WPF/WinForm 作为 UI，Avalonia 可以吗？
---

## 结论

如果想要用Avalonia 做二次开发，只需要：
1. 先构造一次 Application：
    ```csharp
    if (Application.Current == null){
        AppBuilder.Configure<Application>()
            .UsePlatformDetect()
            .WithInterFont()
            .LogToTrace()
            .SetupWithoutStarting();
    }
    ```
1. 使用 `Application.Current.Run(new Window())` 显示窗口

## 可能存在的问题

1. 如何托管控件而不是显示弹窗？
1. 需要处理好 Avalonia 窗口与宿主窗口的关系。
1. 如果有多个插件提供商，Application 和样式可能会冲突，需要小心处理。
1. 主线程上同时运行两个事件循环，会不会出问题。

## 过程

很早以前我就想过这个问题，当时通过简单尝试发现：

1. 必须构造一个 Application，不可以直接 new Window()
2. Application 不可以重复构造，这样就不能每次创建一次 App 用来显示
3. Application 会独占线程，如果另开一个线程用起来就相当不方便了

看起来是相当没有前途，但是刚好，LinqPad 支持嵌入 Avalonia，这证明是嵌入 Avalonia 是可行的，在他的示例中有这么一个方法：

```csharp
// The OnInit() hook method in LINQPad executes once when your process starts.
// This is where we need to initialize the Avalonia subsystem.

void OnInit()
{
	AppBuilder
		.Configure<Application>()
		.UsePlatformDetect()
		.SetupWithoutStarting();

	// Edit the following line to change the theme. After editing, press Shift+F5 to kill the cached process.
	Application.Current.Styles.Add (new FluentTheme());
	
	if (Util.IsDarkThemeEnabled) 
		Application.Current.RequestedThemeVariant = ThemeVariant.Dark;
}
```
这个 `SetupWithoutStarting`就很可疑，原来 Avalonia 可以只初始化不启动吗？那如果不启动Application，又该如何显示窗口呢？

 `new Window().Show()` ? 不行，这么写只能显示一个空窗口，然后这个窗口还是死的。
 
 App 呢？费这么大劲就为了构造一个 App，应该有用吧？发现 App 有一个 `RunWithMainWindow` 的扩展方法，尝试一下可以用。

 但是为什么呢？F12 看一眼源码：
 ```
    /// <summary>
    /// On desktop-style platforms runs the application's main loop until main window is closed
    /// </summary>
    /// <remarks>
    /// Consider using StartWithDesktopStyleLifetime instead, see https://github.com/AvaloniaUI/Avalonia/wiki/Application-lifetimes for details
    /// </remarks>
    public static void Run(this Application app, Window mainWindow)
    {
        if (mainWindow == null)
        {
            throw new ArgumentNullException(nameof(mainWindow));
        }
        var cts = new CancellationTokenSource();
        mainWindow.Closed += (_, __) => cts.Cancel();
        if (!mainWindow.IsVisible)
        {
            mainWindow.Show();
        }
        app.Run(cts.Token);
    }
    
    /// <summary>
    /// On desktop-style platforms runs the application's main loop with custom CancellationToken
    /// without setting a lifetime.
    /// </summary>
    /// <param name="app">The application.</param>
    /// <param name="token">The token to track.</param>
    public static void Run(this Application app, CancellationToken token)
    {
        Dispatcher.UIThread.MainLoop(token);
    }

    public static void RunWithMainWindow<TWindow>(this Application app)
        where TWindow : Avalonia.Controls.Window, new()
    {
        var window = new TWindow();
        window.Show();
        app.Run(window);
    }
 ```

 所以其实就是显示窗口，显示完了启动一个 MainLoop 就可以了, 例如下面这段代码可以同时显示4个窗口：

 ```
    AppBuilder.Configure<App>()
        .UsePlatformDetect()
        .WithInterFont()
        .LogToTrace()
        .SetupWithoutStarting();

    new Window().Show();
    new Window().Show();
    new Window().Show();
    new Window().Show();
    Dispatcher.UIThread.MainLoop(CancellationToken.None);
 ```
> 这感觉就相当灵活，我是不是可以在主窗口显示之前先显示一个窗口，比如登录窗口/启动图之类的
{: .prompt-tip }

