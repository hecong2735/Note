# Thread Programming Guide中Run Loop部分的翻译

Run loops是线程的基本组成部分之一。一个*run loop*是一个用于安排工作并协调收到事件的进程循环。run loop的目标是使线程在用工作的时候持续工作，在没有工作的时候休眠。

Run loop的管理并不是完全自动的。你必须设计你的线程代码在合适的时间启动run loop并处理事件。Cocoa和Core Foundation都提供了*run loop*对象来帮助你配置和管理线程的run loop。你的程序并不需要显示的创建这些对象，包括程序的主线程在内的每个线程，都有一个与之相关联的run loop对象。但是只有次要线程（*secondary threads*）需要显示的运行它们的run loop。应用程序的框架已经自动的在主线程设置并运行run loop作为启动程序的一部分。

接下来的章节将提供关于run loops的更多信息以及如何在你的程序中配置run loop。获取关于run loop的额外信息，可以查看[NSRunLoop Class Reference](https://developer.apple.com/documentation/foundation/nsrunloop)和[CFRunLoop Reference](https://developer.apple.com/documentation/corefoundation/cfrunloopref?language=objc)。

## Run Loop剖析

Run loop就和它的名字一样，是一个线程进入并运用的循环，来运行事件处理程序响应传入事件。你的代码提供了用于实现run loop的实际循环部分的控制语句。换句话说，你的代码提供了操作run loop的*while*或者*for*循环。在循环中，你可以使用一个run loop对象“运行”事件处理代码来接收事件并调用已安装的处理程序。

一个run loop接收来自两种不同类型sources的事件。*Input sources*传递异步事件，通常消息来自另一个线程或者来自另一个应用程序。*Timer sources*传递同步事件，发生在一个预定的时间或者重复间隔。两种source都使用一个应用程序特定的处理程序来处理到达的事件。

3-1展示了run loop的概念结构以及sources的种类。Input sources传递异步事件到相应的处理程序中，并调用`runUntilDate:`方法（在线程关联的`NSRunLoop`对象中调用）退出。Timer sources传递事件到它们的处理程序例程（*handler routines*）中，但不会导致run loop退出。
