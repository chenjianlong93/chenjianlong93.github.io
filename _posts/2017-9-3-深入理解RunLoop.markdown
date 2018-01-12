RunLoop是iOS开发中非常重要的一个概念，这篇文章会介绍RunLoop的概念、实现原理以及苹果对RunLoop的应用以及其他我们常用的应用场景。帮助自己更好地理解RunLoop。
### RunLoop的概念
对于一个程序来说，最小的执行任务的单位就是线程，而线程执行完任务之后一般来讲就会退出。但显然我们使用程序的时候希望任何时间我们唤醒程序的时候它都能正常工作。所以我们需要一个机制，让线程可以随时处理事件但并不退出。这种机制在很多系统和框架都有实现，例如Windows程序的消息循环，而对于OSX/iOS来说这种机制叫RunLoop。

实现这种机制的关键在于：1.有消息时立马被唤醒正常工作 2.没消息时休眠避免资源占用。

所以，RunLoop实际上就是一个对象，这个对象管理了需要处理的事件和消息，并提供了一个函数执行Event Loop的逻辑。线程执行完这个函数后，就会一直处于函数内部“接受->等待->处理”的循环中,直到循环结束,函数返回.

OSX/iOS系统提供了两个对象:NSRunLoop和CFRunLoopRef两个对象.其中前者是对后者的封装,提供的API完全面向对象,但这些API都不是线程安全的;后者则是CF框架,纯C的API,线程安全.
### RunLoop与线程的关系
线程和RunLoop是一一对应的。每个线程（包括主线程）都有一个对应的 Runloop 对象。我们并不能自己创建 Runloop 对象，但是可以获取到系统提供的 Runloop 对象。
主线程的RunLoop会在应用启动后自动启动，其他线程默认不会启动，需要我们手动启动。并且只提供了两个获取的函数：CFRunLoopGetMain() 和 CFRunLoopGetCurrent()。函数实现伪代码如下：

```
/// 全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
/// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;
 
/// 获取一个 pthread 对应的 RunLoop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
    OSSpinLockLock(&loopsLock);
    
    if (!loopsDic) {
        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
        loopsDic = CFDictionaryCreateMutable();
        CFRunLoopRef mainLoop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
    }
    
    /// 直接从 Dictionary 里获取。
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));
    
    if (!loop) {
        /// 取不到时，创建一个
        loop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, thread, loop);
        /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
    }
    
    OSSpinLockUnLock(&loopsLock);
    return loop;
}
 
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```
可以看出，线程创建时并不会有RunLoop，你不主动获取的话一直都不会有，RunLoop的创建发生在第一次获取时，销毁是线程结束时。
### RunLoop中的Mode
一个RunLoop包含若干个Mode，每个Mode又包含不同的source/timer/observer。每次调用RunLoop时只能指定一个Mode，称为currentMode。如果需要切换不同的Mode，只能退出RunLoop再重新指定一个进入。这样做是为了区分不同的source/timer/observer，让其互不影响。

如果用一个东西来比喻的话，就像一个电视机（RunLoop），每次只能放一个台（mode），每个台的内容不同（source/timer/observer），但是每次切换台的时候要先关掉电视，直接跳到另一个台。

RunLoop的Mode有一下这些：
1.  kCFRunLoopDefaultMode: App的默认 Mode，通常主线程是在这个 Mode 下运行的。
2. UITrackingRunLoopMode: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。
3.  UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。
4. GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到。
5. kCFRunLoopCommonModes: 这是一个占位的 Mode，没有实际作用。

> 每个Mode又包含不同的source/timer/observer

**CFRunLoopSourceRef**是事件产生的地方。Source有两个版本：Source0 和 Source1。
* Source0 处理的是App内部的事件、App自己负责管理，如按钮点击事件等。
* Source1 用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程。

**CFRunLoopTimerRef**基于时间的触发器。当加入到RunLoop中时，会注册一个时间点，当时间点到时，RunLoop会被唤醒执行回调。

**CFRunLoopObserverRef**是观察者，每个观察者都有一个回调（函数指针），当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。可以观测的时间点有以下几个：

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 进入休眠
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```

上面的 Source/Timer/Observer 被统称为 mode item，一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。如果一个 mode 中一个 item 都没有，则 RunLoop 会直接退出，不进入循环。

> CFRunLoopMode 和 CFRunLoop 的结构大致如下：

```
struct __CFRunLoopMode {
    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;    // Set
    CFMutableSetRef _sources1;    // Set
    CFMutableArrayRef _observers; // Array
    CFMutableArrayRef _timers;    // Array
    ...
};
 
struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set
    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set
    ...
};
```

主线程RunLoop里有两个预设的Mode：`kCFRunLoopDefaultMode`和`UITrackingRunLoopMode `。**这两个 Mode 都已经被标记为”Common”属性。**

DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。

**这样就会造成滑动的时候timer失效的现象。**

**这里有个概念叫 “CommonModes”**：一个 Mode 可以将自己标记为”Common”属性（通过将其 ModeName 添加到 RunLoop 的 “commonModes” 中）。每当 RunLoop 的内容发生变化时，RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 “Common” 标记的所有Mode里。

**也就是创建Mode时，如果标记为Common，那么添加Source/Observer/Timer这些itmes时如果指定NSRunLoopCommonModes时，会将这些items自动同步到所有标记为”Common”属性的Mode。**

解决方法有两个，第一是在两个Mode中都加入timer这个mode item。第二就是将timer加入NSRunLoopCommonModes.这两者所做的事情其实是一样的，只不过第二种它会将timer自动同步到所有标记为”Common”属性的Mode。

### RunLoop的内部实现
RunLoop内部逻辑大致如下：
![](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_1.png)
内部实现伪代码：

```
/// 用DefaultMode启动
void CFRunLoopRun(void) {
    CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
}
 
/// 用指定的Mode启动，允许设置RunLoop超时时间
int CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean stopAfterHandle) {
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
 
/// RunLoop的实现
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {
    
    /// 首先根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
    /// 如果mode里没有source/timer/observer, 直接返回。
    if (__CFRunLoopModeIsEmpty(currentMode)) return;
    
    /// 1. 通知 Observers: RunLoop 即将进入 loop。
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);
    
    /// 内部函数，进入loop
    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {
        
        Boolean sourceHandledThisLoop = NO;
        int retVal = 0;
        do {
 
            /// 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
            /// 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
            /// 4. RunLoop 触发 Source0 (非port) 回调。
            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
 
            /// 5. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            if (__Source0DidDispatchPortLastTime) {
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
                if (hasMsg) goto handle_msg;
            }
            
            /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
            if (!sourceHandledThisLoop) {
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
            }
            
            /// 7. 调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
            /// • 一个基于 port 的Source 的事件。
            /// • 一个 Timer 到时间了
            /// • RunLoop 自身的超时时间到了
            /// • 被其他什么调用者手动唤醒
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
            }
 
            /// 8. 通知 Observers: RunLoop 的线程刚刚被唤醒了。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
            
            /// 收到消息，处理消息。
            handle_msg:
 
            /// 9.1 如果一个 Timer 到时间了，触发这个Timer的回调。
            if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 
 
            /// 9.2 如果有dispatch到main_queue的block，执行block。
            else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 
 
            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }
            
            /// 执行加入到Loop的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
 
            if (sourceHandledThisLoop && stopAfterHandle) {
                /// 进入loop时参数说处理完事件就返回。
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout) {
                /// 超出传入参数标记的超时时间了
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(runloop)) {
                /// 被外部调用者强制停止了
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
                /// source/timer/observer一个都没有了
                retVal = kCFRunLoopRunFinished;
            }
            
            /// 如果没超时，mode里没空，loop也没被停止，那继续loop。
        } while (retVal == 0);
    }
    
    /// 10. 通知 Observers: RunLoop 即将退出。
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}
```

### 苹果对RunLoop的一些应用
> AutoreleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和\_objc\_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用  \_objc \_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

> 定时器

NSTimer 其实就是 CFRunLoopTimerRef，当一个NSTimer注册到RunLoop后，RunLoop会为其在重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。**但是，为了节省资源，RunLoop并不会在很准的时间回调这个Timer，Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。**

如果RunLoop因为执行一个很耗时的任务，从而错过了某个时间点的话，就会跳过那个时间点的回调不会执行。这就跟坐公交，错过了10：10的，就只好坐10：20的。

**所以，在对时间准确性要求很高的地方，不能用NSTimer，而应该用CADisplayLink。CADisplayLink 是一个和屏幕刷新率一致的定时器。**

> 其他应用

还有一些事件的响应，例如硬件事件(触摸/锁屏/摇晃等)通过注册source1到RunLoop实现。还有手势识别、按钮点击事件之类的app内部的时间通过注册source0到RunLoop实现。

### RunLoop在实际项目中的应用
这一部分我会另写一篇文章进行介绍。