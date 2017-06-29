# Thread Programming Guide中Run Loop部分的翻译

Run loops是线程的基本组成部分之一。一个*run loop*是一个用于安排工作并协调收到事件的进程循环。run loop的目标是使线程在用工作的时候持续工作，在没有工作的时候休眠。

Run loop的管理并不是完全自动的。你必须设计你的线程代码在合适的时间启动run loop并处理事件。Cocoa和Core Foundation都提供了*run loop*对象来帮助你配置和管理线程的run loop。你的程序并不需要显示的创建这些对象，包括程序的主线程在内的每个线程，都有一个与之相关联的run loop对象。但是只有次要线程（*secondary threads*）需要显示的运行它们的run loop。应用程序的框架已经自动的在主线程设置并运行run loop作为启动程序的一部分。

接下来的章节将提供关于run loops的更多信息以及如何在你的程序中配置run loop。获取关于run loop的额外信息，可以查看[NSRunLoop Class Reference](https://developer.apple.com/documentation/foundation/nsrunloop)和[CFRunLoop Reference](https://developer.apple.com/documentation/corefoundation/cfrunloopref?language=objc)。

## Run Loop剖析

Run loop就和它的名字一样，是一个线程进入并运用的循环，来运行事件处理程序响应传入事件。你的代码提供了用于实现run loop的实际循环部分的控制语句。换句话说，你的代码提供了操作run loop的*while*或者*for*循环。在循环中，你可以使用一个run loop对象“运行”事件处理代码来接收事件并调用已安装的处理程序。

一个run loop接收来自两种不同类型sources的事件。*Input sources*传递异步事件，通常消息来自另一个线程或者来自另一个应用程序。*Timer sources*传递同步事件，发生在一个预定的时间或者重复间隔。两种source都使用一个应用程序特定的处理程序来处理到达的事件。

3-1展示了run loop的概念结构以及sources的种类。Input sources传递异步事件到相应的处理程序中，并调用`runUntilDate:`方法（在线程关联的`NSRunLoop`对象中调用）退出。Timer sources传递事件到它们的处理程序例程（*handler routines*）中，但不会导致run loop退出。

![3-1](https://github.com/hecong2735/ThreadingProgrammingGuide/raw/master/img/3-1.jpg)

除了处理传入的sources之外，run loop也产生关于run loop行为的通知。已注册的*run-loop observers*可以获取这些通知并且用它们在进程中做一些额外的处理。可以使用Core Foundation在线程中安装run-loop observers。

接下来的章节将提供关于run loop组成和它们运行模式的更多信息。也会讲述一下在处理事件的不同时期产生的通知。

### Run Loop Modes

一个*run loop mode*是一个被观察的input sources和timers还有被通知的run loop observers的集合。每一次你运行你的run loop，你都要指定（显示或隐式的）一个要运行的特定“mode”。在run loop运行期间，只有与指定mode相关联的sources会被监视并且允许传递事件（同样的，只有与指定mode相关联的observers会被通知run loop的进度）。与其他mode相关联的sources保留所有的新事件直到后续通过循环中合适的mode。

在代码中，可以靠name来区分modes。Cocoa和Core Foundation都定义了一个默认mode和几种常用的mode，以及指定这些mode的字符串。可以通过为mode name指定一个自定义字符串来定义自定义mode。尽管你指定给自定义mode的name是随意的，但是这些mode的内容并不是。为了保证你的创建的mode可以使用，你必须给mode添加一个或多个input sources、timers或者run-loop observers。

使用mode在run loop的过程中过滤不需要sources的事件。大多数时候，你只要以系统默认的“默认（*default*）”mode运行run loop。然而一个modal面板（*modal panel*）可能以“modal”mode运行。在这种mode中，只有和modal面板相关的sources会传递事件给线程。对次要线程来说，你可以使用自定义mode来阻止低优先级的sources在时间紧迫的操作期间传递事件。

**Note**：mode之间的区别是建立在事件的source上而不是事件的类型上的。例如：你不能使用mode来只匹配mouse-down事件或者只匹配键盘事件。你可以使用mode来监听不同的端口集合，临时暂停定时器，或者以其他方式改变source和当前正在监视的run loop observers。

列表3-1列出了Cocoa和Core Foundation定义的几种标准mode以及他们的描述。name列列出了你在实际使用时各个mode的常数。

**列表3-1** 预定义的run loop modes

| Mode           | Name                                     | Description                              |
| -------------- | :--------------------------------------- | ---------------------------------------- |
| Default        | NSDefaultRunLoopMode（Cocoa）<br>kCFRunLoopDefaultMode（Core Foundation | 默认mode是大多数操作使用的mode。大多情况下，你应该使用这个mode来启动run loop并配置你的input sources |
| Connection     | NSConnectionReplyMode（Cocoa）             | Cocoa使用这种mode与NSConnection对象一起监视回复。你自己会很少使用到这个mode。 |
| Modal          | NSModalPanelRunLoopMode（Cocoa）           | Cocoa使用这种mode来分辨modal面板想要的事件。            |
| Event tracking | NSEventTrackingRunLoopMode（Cocoa）        | Cocoa使用这种mode在mouse-dragging循环和其他类型的用户交互循环限制传入事件 |
| Common modes   | NSRunLoopCommonModes（Cocoa）<br>kCFRunLoopCommonModes（CoreFoundation） | 这是一个通常使用mode的配置集合。与这个mode关联的input source也与这个集合中每一个mode相关联。对Cocoa应用程序来说，这个集合包含了default、modal和默认的事件跟踪mode。Core Foundation仅包括初始的默认mode。你可以通过CFRunLoopAddCommonMode函数来添加自定义mode。 |