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
| Default        | NSDefaultRunLoopMode（Cocoa）<br>kCFRunLoopDefaultMode<br>（Core Foundation） | 默认mode是大多数操作使用的mode。大多情况下，你应该使用这个mode来启动run loop并配置你的input sources |
| Connection     | NSConnectionReplyMode（Cocoa）             | Cocoa使用这种mode与NSConnection对象一起监视回复。你自己会很少使用到这个mode。 |
| Modal          | NSModalPanelRunLoopMode（Cocoa）           | Cocoa使用这种mode来分辨modal面板想要的事件。            |
| Event tracking | NSEventTrackingRunLoopMode（Cocoa）        | Cocoa使用这种mode在mouse-dragging循环和其他类型的用户交互循环限制传入事件 |
| Common modes   | NSRunLoopCommonModes（Cocoa）<br>kCFRunLoopCommonModes<br>（CoreFoundation） | 这是一个通常使用mode的配置集合。与这个mode关联的input source也与这个集合中每一个mode相关联。对Cocoa应用程序来说，这个集合包含了default、modal和默认的事件跟踪mode。Core Foundation仅包括初始的默认mode。你可以通过CFRunLoopAddCommonMode函数来添加自定义mode。 |



### Input Sources

Input sources异步的将事件传递给线程，事件的source依赖于input source的类型，input source大体上分为两类。基于端口的input source监视应用程序的Mach ports。自定义input source监视事件的自定义sources。就run loop而言，它并不在乎一个input source是基于端口的还是自定义的。系统通常会将两种类型的input sources都实现。两种source之间的唯一区别就是它们发出信号的方式。基于端口的source由系统核心自动的发出信号，而自行一的source必须由另一个线程手动的发出信号。

创建一个input source时，将它分配给run loop中一个或多个mode。Mode会影响在任何给定时刻哪个input source会被监视。大多数时间会以默认mode运行run loop，但也可以指定特定的自定义mode。如果一个input mode不在当前被监视的mode中，它产生的任何事件都会被保留直到run loop在正确的mode中运行。

下面的章节将介绍一些input source。

#### 基于端口的source

Cocoa和Core Foundation提供内置端口相关对象和函数来创建基于端口的input source。例如在Cocoa中，你永远不用直接的创建一个input source。你只需创建一port对象并且使用NSPort方法将port添加到run loop中。这个port对象会完成所需要input source的创建和配置。

在Core Foundation中，你必须手动的创建port和它的run loop source。在这两种情况下，使用与port不透明类型（*opaque type*）（CFMachPortRef， CFMessagePortRef或者CFSocketRef）来创建适当的对象。

#### 自定义input source

为了创建一个自定义input source，你必须使用在Core Foundation中与CFRunLoopSourceRef不透明类型相关的方法。可以使用几个回调方法来配置自定义input source。Core Foundation在不同的时间点调用这些方法来配置source，处理即将到来的事件以及在source从run loop中移除时拆除它。

除了当事件到来时定义自定义source行为之外，你也必须定义事件传递机制。source的这部分运行在一个单独的线程上，负责向input source提供其数据，并且在数据准备好进行处理时发出信号。事件传递机制取决于你但是不必太过复杂。

#### Cocoa Perform Selector Sources

除了基于端口的source之外，Cocoa还定义了一种自定义input source使你可以在任何线程上执行方法（selector）。如同基于端口的source一样，perform selector需要在目标线程上序列化，来缓解在一个线程上运行多种方法可能发生的许多同步问题。和基于端口的source不同，一个perform selector source在它执行完selector后将自己从run loop中移除。

**Note**：在OS X v10.5之前，perform selector source大多用来给主线程发送信息，但是在OS X v10.5之后和iOS中，你可以用它们给任何线程发送信息。

当在另一个线程上执行selector时，目标线程必须有一个活跃的run loop。对于你自己创建的线程而言，这意味着你需要等到代码显式的启动run loop。因为主线程会启动它自己的run loop，但你也可以在应用程序调用`applicationDidFinishLaunching：`委托方法时立即开始在该线程上发出调用。

表3-2列出了NSObject中可以在其他线程执行selector的方法。因为这些方法是在NSObject中声明的，你能在可以进入Objectiv-C对象的任何线程中使用，包括POSIX线程。这些方法实际中不会创建一个新的线程来执行selector。

**表3-2** 可以在其他线程中执行的selector

| Method                                   | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| `performSelectorOnMainThread:withObject:waitUntilDone:`<br>`performSelectorOnMainThread:withObject:waitUntilDone:modes:` | 在应用程序的主线程的下个run loop循环中执行指定的selector。这些方法使你可以选择阻止当前线程，直到执行selector。 |
| `performSelector:onThread:withObject:waitUntilDone:`<br>`performSelector:onThread:withObject:waitUntilDone:modes:` | 在拥有NSThread对象的任何线程上执行指定的selector。这些方法使你可以选择阻止当前的线程知道执行完selector。 |
| `performSelector:withObject:afterDelay:`<br>`performSelector:withObject:afterDelay:inModes:` | 经过可选择的一段延迟后在当前线程的下个run loop循环中执行指定的selector。因为这要等到下个run loop循环执行selector， 这些方法提供了从当前执行代码的自动微型延迟。多个排队的selector会按照队列顺序逐个执行。 |
| `cancelPreviousPerformRequestsWithTarget:`<br>`cancelPreviousPerformRequestsWithTarget:selector:object:` | 用`performSelector:withObject:afterDelay`或者`performSelector:withObject:afterDelay:inModes:`方法取消一个送往当前线程的信息。 |



