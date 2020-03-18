# Runloop系列


> 本文很大一部分内容来源[ibireme-深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)。


## 入门

runloop顾名思义"运行循环"，不退出线程的情况下循环处理一些事件（触摸事件、定时器事件等）。其实 runloop 的概念不仅仅出现在 iOS 中，NodeJS中也存在类似的`Event Loop`。

runloop 的应用主要包括：

- 定时器（Timer）、PerformSelector
- GCD Async Main Queue
- 事件响应、手势识别、界面刷新
- 网络请求
- 自动释放池 AutoreleasePool


### 没有runloop

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
		NSLog(@"Hello World");
    }
    return 0;
}
```
在命令行的项目中没有runloop的代码，main函数在打印完"Hello World"之后返回（return）0就已经结束了。

### 有runloop

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}

```

在平常的 `iOS` 项目中，UIApplicationMain 函数内部会开启主线程的runloop，不停的循环接受处理事件。运行循环的伪代码我们可以简单的理解为 `While`循环。

```
///  伪代码
int main(int argc, char * argv[]) {
    @autoreleasepool {
        int returnValue = 0;
        do {
            int message = sleep_and_wait();
            returnValue = process_message(message);
        }while (0 == returnValue);
        return 0;
    }
}

```

## Runloop 对象

iOS 中有2套 API 都可以获得 Runloop 对象，分别是：

- Foundation：NSRunLoop
- Core Foundation：CFRunLoopRef


```
// 获取当前的runloop对象
NSRunLoop *runloop = [NSRunLoop currentRunLoop];
CFRunLoopRef runloop2 = CFRunLoopGetCurrent();
```

`NSRunLoop`和`CFRunLoopRef`都是RunLoop对象，区别在于`NSRunLoop`是基于`CFRunLoopRef`的一层OC包装，虽然`NSRunloop`没有源码开放，但是我们可以通过开源的`CFRunLoopRef `来对`Runloop`一探究竟，[CFRunLoopRef开源代码](https://opensource.apple.com/tarballs/CF/)目前最新版本是`CF-1153.18.tar.gz`。


### runloop与线程

每条线程都有唯一的一个与之对应的 `RunLoop` 对象，而且线程刚创建时并没有`RunLoop`对象，只有在第一次获取 `RunLoop` 时才会创建。`RunLoop`创建出来后会保存在一个全局的`Dictionary`里，以线程作为key，`RunLoop`作为value。在线程结束时 `RunLoop` 会被销毁。

**ps:主线程的RunLoop已经自动获取（创建），子线程默认没有开启RunLoop,需要开发者自己去获取创建。**

在`CFRunLoop.c`的源码中可以验证下上面的结论：

```
CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}
```
调用`CFRunLoopGetCurrent`函数获取当前`runloop`时，调用函数 `_CFRunLoopGet0 `传入当前的线程作为参数。

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/runloop/cfrunloop_getcurrent.png?raw=true)

### Runloop 结构

先来认识下 `Runloop` 的内部结构：

```
typedef struct __CFRunLoop * CFRunLoopRef; 
struct __CFRunLoop {
    //---- __CFRunLoop主要成员------
    pthread_t _pthread; // runloop 对应的 线程
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    //---- __CFRunLoop主要成员------
   // others..
};
```

runloop 中包含若干个 `Mode`， 其中集合`CFMutableSetRef _modes;`存放`CFRunLoopModeRef`对象, `_currentMode`也是`CFRunLoopModeRef`对象，代表 runloop 当前是在哪个`Mode`下。`_commonModeItems` 和 `_commonModes ` 通过kCFRunLoopCommonModes (NSRunLoopCommonModes)标记分别存放的`item(Source/Observer/Timer)`和`Mode`(主线程的 RunLoop 里会有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode)。

每当 RunLoop 的内容发生变化时，RunLoop 都会自动将` _commonModeItems` 里的 `Source/Observer/Timer` 同步到具有 `Common` 标记的所有Mode里。


```
typedef struct __CFRunLoopMode *CFRunLoopModeRef;
struct __CFRunLoopMode {
	// ----__CFRunLoopMode主要成员----
    CFStringRef _name; // mode的名字
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;	
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
	// ----__CFRunLoopMode主要成员----
	
	// others..
};
```

其中`_sources0 ` 和 `_sources1 `的集合存放`CFRunLoopSourceRef`对象、`_observers` 数组存放 `CFRunLoopObserverRef`对象、`_timers`数组存放`CFRunLoopTimerRef`对象。

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/runloop/runloop_struct.png?raw=true)

runloop 中包含多个 `Mode`，而每个`Mode`都有自己的`_sources0 `、`_sources1`、`_observers`、`_timers`，目前常用的 Mode 主要有两种：

- kCFRunLoopDefaultMode（NSDefaultRunLoopMode）：App的默认Mode，通常主线程是在这个Mode下运行。
- UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。
- [其它Mode](http://iphonedevwiki.net/index.php/CFRunLoop)


`_currentMode`在这些`Mode`中不断的切换。处理Mode下的`_sources0 `、`_sources1`、`_observers`、`_timers`。切换`Mode`时会退出当前的`Mode`,重新进入新的`Mode`。

## Runloop 运行逻辑

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/runloop/runloop_logic.png?raw=true)

以下代码摘自：[
深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)

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

## runloop的休眠
`runloop` 进入休眠时调用的函数是 `mach_msg()`,`mach_msg()` 函数实际上是调用了一个 Mach 陷阱 (trap)，即函数mach_msg_trap()，陷阱这个概念在 Mach 中等同于系统调用。当你在用户态调用 mach_msg_trap() 时会触发陷阱机制，切换到内核态；内核态中内核实现的 mach_msg() 函数会完成实际的工作，如下图：

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/runloop/runloop_mach_msg.png?raw=true)

## runloop 的唤醒

### source0

- 触摸事件处理

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/runloop/runloop_source0_touches.png?raw=true)

- `performSelector:onThread:`

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/runloop/runloop_source0_perform.png?raw=true)


### Source1 

Source1主要是基于 port 的线程间通信。


### timers

- NSTimer的唤醒

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/runloop/runloop_timer.png?raw=true)

- performSelector:withObject:afterDelay:

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/runloop/runloop_afeterDealy.png?raw=true)

>小结一下：以下代码来源[ibireme-深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)

```
{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {
 
        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
 
        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();
        
 
        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
 
        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
 
        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
 
        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
 
    } while (...);
 
    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```

## Observers

Observers 主要用于监听RunLoop的状态，当runloop即将进入等待休眠的时候会有UI刷新、Autorelease pool释放。

`RunLoop`的状态主要有以下几种：

```
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),          // 即将进入 runloop
    kCFRunLoopBeforeTimers = (1UL << 1),   // 即将处理 Timer 
    kCFRunLoopBeforeSources = (1UL << 2),  // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5),  // 即将休眠
    kCFRunLoopAfterWaiting = (1UL << 6),   // 从休眠中唤醒
    kCFRunLoopExit = (1UL << 7),           // 退出 runloop
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

我们可以通过打印查看以系列的 runloop 状态：

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/runloop/runloop_state.png?raw=true)

## mainRunloop

通过 `po [NSRunLoop mainRunLoop]` 查看 runloop 的内部以下简化代码同样摘自[ibireme-深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)	
:

```
po [NSRunLoop mainRunLoop]

CFRunLoop {
    current mode = kCFRunLoopDefaultMode
    common modes = {
        UITrackingRunLoopMode
        kCFRunLoopDefaultMode
    }
 
    common mode items = {
 
        // source0 (manual)
        CFRunLoopSource {order =-1, {
            callout = _UIApplicationHandleEventQueue}}
        CFRunLoopSource {order =-1, {
            callout = PurpleEventSignalCallback }}
        CFRunLoopSource {order = 0, {
            callout = FBSSerialQueueRunLoopSourceHandler}}
 
        // source1 (mach port)
        CFRunLoopSource {order = 0,  {port = 17923}}
        CFRunLoopSource {order = 0,  {port = 12039}}
        CFRunLoopSource {order = 0,  {port = 16647}}
        CFRunLoopSource {order =-1, {
            callout = PurpleEventCallback}}
        CFRunLoopSource {order = 0, {port = 2407,
            callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_}}
        CFRunLoopSource {order = 0, {port = 1c03,
            callout = __IOHIDEventSystemClientAvailabilityCallback}}
        CFRunLoopSource {order = 0, {port = 1b03,
            callout = __IOHIDEventSystemClientQueueCallback}}
        CFRunLoopSource {order = 1, {port = 1903,
            callout = __IOMIGMachPortPortCallback}}
 
        // Observer
        CFRunLoopObserver {order = -2147483647, activities = 0x1, // Entry
            callout = _wrapRunLoopWithAutoreleasePoolHandler}
        CFRunLoopObserver {order = 0, activities = 0x20,          // BeforeWaiting
            callout = _UIGestureRecognizerUpdateObserver}
        CFRunLoopObserver {order = 1999000, activities = 0xa0,    // BeforeWaiting | Exit
            callout = _afterCACommitHandler}
        CFRunLoopObserver {order = 2000000, activities = 0xa0,    // BeforeWaiting | Exit
            callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv}
        CFRunLoopObserver {order = 2147483647, activities = 0xa0, // BeforeWaiting | Exit
            callout = _wrapRunLoopWithAutoreleasePoolHandler}
 
        // Timer
        CFRunLoopTimer {firing = No, interval = 3.1536e+09, tolerance = 0,
            next fire date = 453098071 (-4421.76019 @ 96223387169499),
            callout = _ZN2CAL14timer_callbackEP16__CFRunLoopTimerPv (QuartzCore.framework)}
    },
 
    modes ＝ {
        CFRunLoopMode  {
            sources0 =  { /* same as 'common mode items' */ },
            sources1 =  { /* same as 'common mode items' */ },
            observers = { /* same as 'common mode items' */ },
            timers =    { /* same as 'common mode items' */ },
        },
 
        CFRunLoopMode  {
            sources0 =  { /* same as 'common mode items' */ },
            sources1 =  { /* same as 'common mode items' */ },
            observers = { /* same as 'common mode items' */ },
            timers =    { /* same as 'common mode items' */ },
        },
 
        CFRunLoopMode  {
            sources0 = {
                CFRunLoopSource {order = 0, {
                    callout = FBSSerialQueueRunLoopSourceHandler}}
            },
            sources1 = (null),
            observers = {
                CFRunLoopObserver >{activities = 0xa0, order = 2000000,
                    callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv}
            )},
            timers = (null),
        },
 
        CFRunLoopMode  {
            sources0 = {
                CFRunLoopSource {order = -1, {
                    callout = PurpleEventSignalCallback}}
            },
            sources1 = {
                CFRunLoopSource {order = -1, {
                    callout = PurpleEventCallback}}
            },
            observers = (null),
            timers = (null),
        },
        
        CFRunLoopMode  {
            sources0 = (null),
            sources1 = (null),
            observers = (null),
            timers = (null),
        }
    }
}
```


## runloop 的应用

### AutoreleasePool
在 mainRunloop 中有两个关于 AutoreleasePool 的 CFRunLoopObserver,其回调都是函数`_wrapRunLoopWithAutoreleasePoolHandler`。

```
        CFRunLoopObserver {order = -2147483647, activities = 0x1, // Entry
            callout = _wrapRunLoopWithAutoreleasePoolHandler}
        CFRunLoopObserver {order = 2147483647, activities = 0xa0, // BeforeWaiting | Exit
            callout = _wrapRunLoopWithAutoreleasePoolHandler}
```

分别监听了进入 runloop 、即将休眠和退出 runloop。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 `_objc_autoreleasePoolPush()` 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用`_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()` 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 `_objc_autoreleasePoolPop()` 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。


### 事件响应

苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 `__IOHIDEventSystemClientQueueCallback()`。当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考[这里](http://iphonedevwiki.net/index.php/IOHIDFamily)。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

我试过直接在按钮的`action`方法中断点，看到的是source0 触发的 `_UIApplicationHandleEventQueue()`的。作者也在评论中补充了下:

> 首先是由那个Source1 接收IOHIDEvent，之后在回调 `__IOIDEventSystemClientQueueCallback()`内触发的 Source0，Source0 再触发的 `_UIApplicationHandleEventQueue()`。所以UIButton事件看到是在 Source0 内的。


### 手势识别

当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。

当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

### 界面更新

当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：
`_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()`。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

### GCD

当调用 `dispatch_async(dispatch_get_main_queue(), block)`时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调 `__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()` 里执行这个 block。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

### PerformSelecter

当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。
