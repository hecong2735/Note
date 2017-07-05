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
| `performSelector:onThread:withObject:waitUntilDone:`<br>`performSelector:onThread:withObject:waitUntilDone:modes:` | 在拥有NSThread对象的任何线程上执行指定的selector。这些方法使你可以选择阻止当前的线程直到执行完selector。 |
| `performSelector:withObject:afterDelay:`<br>`performSelector:withObject:afterDelay:inModes:` | 经过可选择的一段延迟后在当前线程的下个run loop循环中执行指定的selector。因为这要等到下个run loop循环执行selector， 这些方法提供了从当前执行代码的自动微型延迟。多个排队的selector会按照队列顺序逐个执行。 |
| `cancelPreviousPerformRequestsWithTarget:`<br>`cancelPreviousPerformRequestsWithTarget:selector:object:` | 用`performSelector:withObject:afterDelay`或者`performSelector:withObject:afterDelay:inModes:`方法取消一个送往当前线程的信息。 |



#### Timer Sources

Timer Sources在预定的时间内将事件同步传递给线程。Timer是线程通知自己处理事件的方式。例如在用户连续的按键之间经过一段时间，就可以使用timer来启动自动搜索。延迟时间的运用让用户能在开始搜索之前输入尽可能多的搜索字段。

尽管timer是基于时间的通知，但是它并不是实时机制。和input source一样，timer和run loop中的指定mode相关联。如果timer不在当前run loop监视的mode中，它并不会起作用直到你在一个支持timer的mode中运行。类似的，如果在run loop处于执行处理例程的中间timer触发，则timer需要等待直到下一次通过run loop来调用处理例程。如果run loop没有运行，那么timer也不会触发。

你可以为产生的事件单次或重复地配置timer。一个重复的timer会根据预定的触发时间而不是实际的触发时间自动的重新触发。例如一个timer安排在一个特定的时间触发并且过后的每5秒触发一次，那么预定的触发时间总是会在初始时间的5秒间隔上，即使实际上的触发时间延迟了。如果触发时间延迟太久导致timer错过了一次或多次的预定触发时间，那么timer只会为错过的这段时间触发一次。在为错过的时间触发完后，timer会在下一个预定的触发时间重新触发。

#### Run Loop Observers

和在一个合适的同步或异步事件发生时触发的source相反，run loop observers在run loop执行期间的特殊地点触发。你可以使用run loop observers来为你的线程处理事件做准备，或者做线程休眠之前的准备工作。你可以在run loop中与下面的事件关联run loop observers：

- run loop的入口。
- 当run loop即将处理一个timer。
- 当run loop即将处理一个input source。
- 当run loop即将休眠。
- 当run loop被唤醒，但是在它处理唤醒它的事件之前。
- 退出run loop时。

你可以使用Core Foundation将run loop observers加入到应用程序中。你可以创造`CFRunLoopObserverRef`不透明类型的一个新的实例来创建run loop observer。这种类型会跟踪你自定义的回调函数和塔关注的活动。

和timer一样，run-loop observers可以单次或重复使用。一个一次性的observers在触发后会将自己从run loop中移除，而重复的observers仍然在这个run loop中。你可以在创建的时候指定run loop运行一次还是多次。

#### Run Loop的事件顺序

每当你运行线程的run loop，它都会处理即将发生的事件并且为依附的observers产生通知。它处理事件的顺序如下：

1. 通知observers run loop已经进入。
2. 通知observers所有准备触发的timer。
3. 通知observers所有准备出发的非基于端口的input source。
4. 触发所有非基于端口的input source。
5. 如果一个基于端口的input source等待触发，那么立即执行事件并跳转到步骤9。
6. 通知observers线程准备休眠。
7. 使线程休眠知道如下的事件发生：
   - 一个基于端口的input source事件到达。
   - 一个timer触发。
   - run loop到达过期时间。
   - run loop被显式的唤醒。
8. 通知observers线程刚刚被唤醒。
9. 处理即将到来的事件。
   - 如果一个用户定义的timer被触发，处理timer事件并重启循环。跳到步骤2。
   - 如果一个input source被触发，传递事件。
   - 如果run loop被显式的唤醒并且没有超时，重启循环。跳到步骤2。
10. 通知observers run loop已经退出。

因为observer给timer和input source的通知在这些事件发生之前就已经被传递了，所以在通知的时间和事件的真实发生时间之间可能会有一段间隔。如果事件之间的时间是关键的，可以使用休眠和从休眠中唤醒通知来帮助你将实际事件之间的时间关联起来。

因为在运行run loop的时候会传递timer和其他周期性的事件，规避该循环会影响这些事件的传递。典型的例子就是无论何时当你通过进入一个循环并重复的从应用程序获取事件来实现一个鼠标跟踪例程。因为你的代码是直接抓取事件而不是让应用程序正常的发送事件，活动的timer将会失效直到鼠标跟踪例程退出并且返回到应用程序的控制中。

一个run loop可以用过run loop对象显式的唤醒。其他事件也可以导致run loop被唤醒。例如添加一个非基于端口的input source会唤醒run loop这样这个input source才会立即被处理而不是等到其他的事件发生时。

## 何时使用Run Loop

只要创建次要线程的时候才需要显式的运行run loop。对主线程而言，run loop是其基础的重要组成部分。app框架提供了运行主程序循环的代码并且会自动开始该循环。iOS中UIApplication（OS X的NSApplication）的`run`方法会启动应用程序的主循环作为正常启动步骤的一部分。如果你使用Xcode的模板工程来创建应用程序，你不应该显式的调用这个例程。

对于次要线程而言，你需要决定run loop对它是否是必要的。如果是，你需要自己配置并启动它。没有必要在所有的情况下都启动run loop。例如你在使用线程运行一些长期和预先安排的任务，你可能要避免使用run loop。Run loop是给你想要更多互动的线程准备的，比如你计划做如下的事情时，你需要启动run loop：

- 使用port或者自定义的input source来和其他线程交流。
- 在线程中使用timer。
- 在Cocoa应用程序中使用任何`performSelector...`方法。
- 保持线程执行周期任务。

如果你确实要使用run loop，那么建立和配置很简单。和所有的多线程编程一样，你应该有一个次要线程退出的合适时机。退出而不是强制终止结束总是能更好的关闭一个线程。

## 使用Run Loop对象

一个run loop对象提供了添加input source、timers和run-loop observers的主要接口并且运行它。每个线程都有一个run loop对象与之相关联。在Cocoa中，这个对象是NSRunLoop的实例。在低等应用程序（*low-level application*）中，它是指向CFRunLoopRef不透明类型的指针。

### 获取一个Run Loop对象

给当前线程获取run loop，你可以按下面的方式：

- 在Cocoa应用程序中，使用NSRunLoop的`currentRunLoop`方法来取回一个NSRunLoop对象。
- 使用CFRunLoopGetCurrent函数。

尽管这不是免费的桥接类型，你可以从NSRunLoop对象中获取一个CFRunLoopRef不透明对象。NSRunLoop类定义了一个返回CFRunLoopRef类型的getCFRunLoop方法使你可以通过Core Foundation例程。因为这两种对象都是关联的同一个run loop，你可以随意的使用NSRunLoop对象或者CFRunLoopRef不透明类型。

### 配置Run Loop

在次要线程运行run loop之前，你必须添加至少一个input source或者timer。如果一个run loop没有任何source监视，在你试图运行的时候，就会立即退出。

除了安装source之外，你还可以安装run loop observers并使用它们来检测run loop运行的不同阶段。要安装run loop observers，你可以创建一个CGRunLoopObservrRef不透明对象，然后使用CFRunLoopAddObserver方法来添加到run loop中。Run loop observers必须使用Core Foundation创建，即使是在Cocoa应用程序中。

清单3-1展示将一个run loop observers添加到一个线程的run loop中的主要例程。这个例子的目的是给你展示如何创建一个run loop observer，因此这个代码只需简单的设置一个run loop observer来监视所有的run loop活动。在处理timer请求时，基本的处理例程（未显示）仅仅记录run loop活动。

**清单3-1** 创建run loop observer

```
- (void)threadMain
{
    // 应用程序使用垃圾回收，所以没有必要使用autorelease pool
    NSRunLoop* myRunLoop = [NSRunLoop currentRunLoop];
 
    // 创建一个run loop observer并把它添加到run loop中
    CFRunLoopObserverContext  context = {0, self, NULL, NULL, NULL};
    CFRunLoopObserverRef    observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
            kCFRunLoopAllActivities, YES, 0, &myRunLoopObserver, &context);
 
    if (observer)
    {
        CFRunLoopRef    cfLoop = [myRunLoop getCFRunLoop];
        CFRunLoopAddObserver(cfLoop, observer, kCFRunLoopDefaultMode);
    }
 
    // 创建并安排timer
    [NSTimer scheduledTimerWithTimeInterval:0.1 target:self
                selector:@selector(doFireTimer:) userInfo:nil repeats:YES];
 
    NSInteger    loopCount = 10;
    do
    {
        // 运行run loop10次以触发timer
        [myRunLoop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:1]];
        loopCount--;
    }
    while (loopCount);
}
```

当为一个长期运行的线程配置run loop时，最好添加至少一个input source来接收信息。尽管你可以仅用附加一个timer来进入run loop，一旦timer触发，它通常会被销毁，这回导致run loop退出。附加一个重复的timer可以使run loop运行更长的一段时间，但是需要周期性的触发timer以唤醒线程，这实际上是另一种形式的轮询（*polling*）。相反的，一个input source会等待一个事件的发生，使你的线程休眠。

### 启动Run Loop

只有在次要线程中才有必要启动run loop。一个run loop必须至少含有一个input source或者timer来监控。如果一个都没有添加，run loop会立即退出。

有如下几种方法可以启动run loop：

- 无条件的（*unconditionally*）。
- 设定时限。
- 在特定的mode中。

无条件的进入run loop是最简单的选择，但是也是最不可取的。无条件的运行run loop会使线程进入一个永久循环中，这使无法对循环机进行控制。你可以添加或移除input sources和timers，但是停止run loop的唯一方法是终止它。并且也没有办法在自定义的mode中运行run loop。

和无条件的运行run loop不同，运行run loop的时候最好添加一个时限。当你添加了一个时限，run loop会运行到事件到达或者时限到期。如果一个事件到达，这个事件会发送给处理程序处理然后run loop退出。你的代码可以重新启动run loop并且解决下一个事件。如果是到期了，你可以重启run loop或者使用时间来进行内务管理（*housekeeping*）。

除了添加时限之外，你还可以使用特定的mode来运行run loop。mode和时限不是互斥的并且都可以用来启动run loop。mode限制了传递事件给run loop的source类型。详情可以查看Run Loop Modes。

清单3-2展示了线程主要整体例程的框架。这个例子的关键部分展示了run loop的基本结构。本质上是通过添加input source和timer到run loop中然后重复调用例程中的一个来启动run loop。每次run loop例程返回，都检查一遍是否有满足退出线程的条件。这个例子使用Core Foundation run loop例程，这样就能检查返回结果并且查明run loop为何退出。如果你在使用Cocoa并且你不需要检查返回结果，你也可以在类似的管理器使用NSRunLoop类中的方法来运行run loop。（调用 NSRunLoop类中方法的run loop例子可以查看清单3-14）

**清单 3-2** 运行一个run loop

```
- (void)skeletonThreadMain
{
    // 创建一个autorelease pool，如果没有使用垃圾回收的话。
    BOOL done = NO;
 
    // 添加source或timer和一些其他操作
 
    do
    {
        // Start the run loop but return after each source is handled.
        SInt32    result = CFRunLoopRunInMode(kCFRunLoopDefaultMode, 10, YES);
 
        // 如果一个source显式的停止了run loop，或者
        // 没有source和timers，继续并退出
        if ((result == kCFRunLoopRunStopped) || (result == kCFRunLoopRunFinished))
            done = YES;
 
        // 这里检查其他的退出条件并设置done变量的值。
    }
    while (!done);
 
    // 这里是清楚代码，保证释放了所有分配的autorelease pool
}
```

可以递归的运行run loop。换句话说，你可以调用CFRunLoopRun，CFRunLoopInMode或者其他的NSRunLoop方法在input source或者timer的处理例程内启动run loop。当你这么做的时候，你可以使用任何mode来运行嵌套的run loop，包括外部run loop运行的mode。

### 退出Run Loop

在run loop处理事件之前退出有两种方法：

- 配置run loop的运行时限。
- 让run loop停止。

使用一个时限无疑是更好的方法，只要有较好的管理。指定一个时限能够让run loop在退出之前结束所有正常的处理活动，包括传递信息给run loop observers。

使用CFRunLoopStop函数显式的停止run loop可以起到一个和设定时限类似的作用。run loop会发送所有仍然存在的run-loop通知然后退出。两者之间的区别在于你可以使用这个方法来停止无条件启动的run loop。

尽管移除run loop的input source和timer也可能导致run loop退出，但这并不是一个可靠的方法。一些系统例程会添加input source到run loop中来解决一些事件。因为你的代码可能并没有注意到这些input source，所以你也就无法移除他们使你的run loop退出。

### 线程安全和Run Loop对象

线程安全的差异取决于你使用哪一个API来操作run loop。Core Foundation里的方法大多是线程安全并且可以从任意的线程里调用。但是，如果你打算改变run loop的配置，从run loop所在的线程进行操作仍是最佳选择。

Cocoa的NSRunLoop类并没有和Core Foundation一样内置线程安全。如果你在使用NSRunLoop类来修改run loop，你应该在run loop所在的线程上进行操作。添加input source或者timer到属于另一个线程的run loop上可能导致程序崩溃或者运行异常。

## 配置Run Loop Sources

接下来的章节将展示一些在Cocoa和Core Foundation中如何设置不同类型source的例子。

### 定义一个自定义input source

创建一个自定义input source需要定义如下属性：

- 你希望input source处理的信息。
- 一个调度程序，让感兴趣的客户端知道如何联系input source。
- 一个执行任何客户端发送请求的处理程序。
- 一个无效input source的终止程序。

因为你创建了一个自定义input source来处理自定义的信息，所以实际配置被设计为灵活的。调度、处理和取消程序是自定义input source的关键程序。然而，剩下大多数的input source行为，都发生在这些程序之外。例如自定义input source的数据传送机制并且将input source的存在传递给其他线程。

图片3-2展示了配置自定义input source的例子。在这个例子中，应用程序的主线程维持对input source、对该input source的自定义命令缓冲区（*custom command buffer*）和这个input source安装的run loop的引用。当主线程有一个任务想要在工作线程上完成，他传递一个命令和其他工作线程开始任务所需要的信息给命令缓冲区（因为主线程和工作线程的input source都可以访问命令缓冲区，访问必须是同步的）。命令被传达后，主线程就通知input source，并且将工作线程的run loop唤醒。一旦run loop获得唤醒命令，他就会调用input source的处理程序，处理命令缓冲区的命令。

**图3-2** 自定义input source操作

![3-2](https://github.com/hecong2735/ThreadingProgrammingGuide/raw/master/img/3-2.jpg)

接下来的章节将解释图中自定义input source的实现并且展示你需要实现的关键代码。

### 定义input source

定义一个自定义input source需要使用Core Foundation来配置run loopsource并且将它添加到run loop中。尽管基本的处理程序是基于C的函数，但你仍然可以使用Objective-C或者C++来封装他们。

图3-2介绍的input  source使用了一个Objective-C对象来管理命令缓冲区和在run loop之间进行协调。清单3-3展示了这个对象的定义。RunLoopSource对象管理一个命令缓冲区并且使用它缓冲从其他线程得到的信息。这个清单也展示了RunLoopContext对象的定义，RunLoopContext对象仅仅是一个容器对象，用于将RunLoopSource对象和run loop的引用传递给主线程。

**清单3-3** 自定义input source对象的定义

```
@interface RunLoopSource : NSObject
{
    CFRunLoopSourceRef runLoopSource;
    NSMutableArray* commands;
}
 
- (id)init;
- (void)addToCurrentRunLoop;
- (void)invalidate;
 
// Handler method
- (void)sourceFired;
 
// Client interface for registering commands to process
- (void)addCommand:(NSInteger)command withData:(id)data;
- (void)fireAllCommandsOnRunLoop:(CFRunLoopRef)runloop;
 
@end
 
// These are the CFRunLoopSourceRef callback functions.
void RunLoopSourceScheduleRoutine (void *info, CFRunLoopRef rl, CFStringRef mode);
void RunLoopSourcePerformRoutine (void *info);
void RunLoopSourceCancelRoutine (void *info, CFRunLoopRef rl, CFStringRef mode);
 
// RunLoopContext is a container object used during registration of the input source.
@interface RunLoopContext : NSObject
{
    CFRunLoopRef        runLoop;
    RunLoopSource*        source;
}
@property (readonly) CFRunLoopRef runLoop;
@property (readonly) RunLoopSource* source;
 
- (id)initWithSource:(RunLoopSource*)src andLoop:(CFRunLoopRef)loop;
@end
```

尽管objective-C代码管理input source的自定义数据，将input source添加到run loop中仍然需要基于C的回调函数。这个函数的第一个会在你将run loop source添加到run loop上时被调用，这在清单3-4中被展示。因为这个input source只有一个客户端（主线程），他使用调度函数来发送信息，以便在该线程上使用应用程序委托注册自己。当委托想和input source联系的时候，就使用RunLoopContext对象中的信息。