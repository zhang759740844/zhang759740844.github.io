title: iOS Crash 收集框架 KSCrash 源码解析(上)
date: 2019/10/25 14:07:12  
categories: iOS
tags: 

 - 源码解析
 - 学习笔记
---

KSCrash 是 iOS 上一个知名的 crash 收集框架。包括腾讯刚开源的 APM 框架 Matrix，其中 crash 收集部分也是直接使用的 KSCrash。那么 KSCrash 到底是如何进行 crash 收集的呢？

<!--more-->

## 总体结构

总体结构如下图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/kscrash_1.png?raw=true)

可以看到总共分为三大部分：

- Crash Recording
- Crash Reporting
- Installation

其中 Installation 用来启动 KSCrash，并且指定了 Crash 收集的方式。Crash 收集方式可以包括：邮件发送，向指定服务器发送，向专门的 Crash 收集服务器发送等方式。每种方式对应于一个实现类：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/kscrash_2.png?raw=true)

Crash Reporting 包含了三个子文件夹，分别是 Filter，Sink 和 Tools，主要用来上报 Crash。本文不会对其做具体的研究。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/kscrash_3.png?raw=true)

其中 Filter 包含了将存储在设备上的 Crash 信息的 NSData 转为 NSString，JSON，GZip 等几种方式的实现类。

Sink 则是不同发送方式的真正的处理者，和 Installation 中的几种收集方式一一对应。

Tools 是一些工具类，以及发送请求相关的类的合集。

最最关键的还是要数 Crash Recording，它包含了捕捉各种类型 crash 的方式，将会在下文中做详细介绍。

## 基本使用

以官方 Demo 中的 Simple-Example 为例：

```objc
@implementation AppDelegate

- (BOOL) application:(__unused UIApplication *) application
didFinishLaunchingWithOptions:(__unused NSDictionary *) launchOptions {
    [self installCrashHandler];    
    return YES;
}

- (void) installCrashHandler {
    /// 👇是各种 crash 收集方式，选择一种使用
//    KSCrashInstallation* installation = [self makeStandardInstallation];
    KSCrashInstallation* installation = [self makeEmailInstallation];
//    KSCrashInstallation* installation = [self makeHockeyInstallation];
//    KSCrashInstallation* installation = [self makeQuincyInstallation];
//    KSCrashInstallation *installation = [self makeVictoryInstallation];

    /// 注册 crash handler
    [installation install];

    /// 发送所有的 crash 日志
    [installation sendAllReportsWithCompletion:^(NSArray* reports, BOOL completed, NSError* error) {
         if(completed) {
             NSLog(@"Sent %d reports", (int)[reports count]);
         } else {
             NSLog(@"Failed to send reports: %@", error);
         }
     }];
}

- (KSCrashInstallation*) makeEmailInstallation {
    NSString* emailAddress = @"your@email.here";
    
    KSCrashInstallationEmail* email = [KSCrashInstallationEmail sharedInstance];
    email.recipients = @[emailAddress];
    email.subject = @"Crash Report"
    email.message = @"This is a crash report";
    email.filenameFmt = @"crash-report-%d.txt.gz";
    
    [email addConditionalAlertWithTitle:@"Crash Detected"
                                message:@"The app crashed last time it was launched. Send a crash report?"
                              yesAnswer:@"Sure!"
                               noAnswer:@"No thanks"];
    
    // Uncomment to send Apple style reports instead of JSON.
    [email setReportStyle:KSCrashEmailReportStyleApple useDefaultFilenameFormat:YES];

    return email;
}

@end
```

如上面代码所示，在 AppDelegate 中注册 KSCrash。在注册 handler 期间，通过工厂模式选择一个上传方式的 installation 进行实例化。不同的 installation 会有不同的处理，比如上面的 email 上传方式，那就需要提供邮箱地址。

不同的 installation 有不同的上床日志的方式，但是它们注册监听的方式都是一样的。它们都继承于基类 `KSCrashInstallation`，在基类中有统一的 `install` 方法。

## 开启 Crash 监控

在 `install` 的过程中，installation 创建了一个单例对象 `KSCrash`，并且调用它的 `install` 

方法：

```objc
// KSCrash.m
/// 开启监控
- (BOOL) install {
    _monitoring = kscrash_install(self.bundleName.UTF8String,
                                          self.basePath.UTF8String);
    if(self.monitoring == 0) {
        return false;
    }
    return true;
}
```

它又调用了 c 方法 `kscrash_install`，这个方法在 `KSCrashC.c` 中：

```c
/// 真正开始的方法, 创建一个 monitor
KSCrashMonitorType kscrash_install(const char* appName, const char* const installPath) {
    /// 省略了一些基本设置，包括路径名，文件名等
    ...
    /// 设置状态为已启动
    g_installed = 1;
    /// 给 monitor 设置回调
    kscm_setEventCallback(onCrash);
    /// 设置 monitor
    KSCrashMonitorType monitors = kscrash_setMonitoring(g_monitoring);

    return monitors;
}
```

设置回调函数 `onCrash` 的部分我们也后面再看。先来看设置 monitor 的部分：

```c
/// 设置 monitor
KSCrashMonitorType kscrash_setMonitoring(KSCrashMonitorType monitors)
{
    g_monitoring = monitors;
    
    if(g_installed)
    {
        kscm_setActiveMonitors(monitors);
        return kscm_getActiveMonitors();
    }
    // Return what we will be monitoring in future.
    return g_monitoring;
}
```

由于前面设置了 `g_installed` ，因此，这里就会通过 `kscm_setActiveMonitors()` 这个方法激活 monitor。这是整个激活步骤的核心方法：

```c
void kscm_setActiveMonitors(KSCrashMonitorType monitorTypes)
{
    /// 进程是否在调试中，调试中的应用只记录 Mach Signal C++ OC 的 crash
    if(ksdebug_isBeingTraced() && (monitorTypes & KSCrashMonitorTypeDebuggerUnsafe))
    {
        static bool hasWarned = false;
        if(!hasWarned)
        {
            hasWarned = true;
            KSLOGBASIC_WARN("    ************************ Crash Handler Notice ************************");
            KSLOGBASIC_WARN("    *     App is running in a debugger. Masking out unsafe monitors.     *");
            KSLOGBASIC_WARN("    * This means that most crashes WILL NOT BE RECORDED while debugging! *");
            KSLOGBASIC_WARN("    **********************************************************************");
        }
        monitorTypes &= KSCrashMonitorTypeDebuggerSafe;
    }
    /// 是否需要线程安全，只有 Mach 和 Signal 异常是线程安全的
    if(g_requiresAsyncSafety && (monitorTypes & KSCrashMonitorTypeAsyncUnsafe))
    {
        KSLOG_DEBUG("Async-safe environment detected. Masking out unsafe monitors.");
        monitorTypes &= KSCrashMonitorTypeAsyncSafe;
    }

    KSLOG_DEBUG("Changing active monitors from 0x%x tp 0x%x.", g_activeMonitors, monitorTypes);

    KSCrashMonitorType activeMonitors = KSCrashMonitorTypeNone;
    for(int i = 0; i < g_monitorsCount; i++)
    {
        Monitor* monitor = &g_monitors[i];
        bool isEnabled = monitor->monitorType & monitorTypes;
        /// 开启相应的 monitor
        setMonitorEnabled(monitor, isEnabled);
        if(isMonitorEnabled(monitor))
        {
            activeMonitors |= monitor->monitorType;
        }
        else
        {
            activeMonitors &= ~monitor->monitorType;
        }
    }

    KSLOG_DEBUG("Active monitors are now 0x%x.", activeMonitors);
    g_activeMonitors = activeMonitors;
}
```

先来看入参，外部传入的 `g_monitoring` 默认值为 `KSCrashMonitorTypeProductionSafeMinimal`，它的定义如下：

```c
#define KSCrashMonitorTypeProductionSafe (KSCrashMonitorTypeAll & (~KSCrashMonitorTypeExperimental))
#define KSCrashMonitorTypeProductionSafeMinimal (KSCrashMonitorTypeProductionSafe & (~KSCrashMonitorTypeOptional))
```

再来看 `KSCrashMonitorTypeAll` 的定义：

```objc
typedef enum
{
    /* Captures and reports Mach exceptions. */
    KSCrashMonitorTypeMachException      = 0x01,
    
    /* Captures and reports POSIX signals. */
    KSCrashMonitorTypeSignal             = 0x02,
    
    /* Captures and reports C++ exceptions.
     * Note: This will slightly slow down exception processing.
     */
    KSCrashMonitorTypeCPPException       = 0x04,
    
    /* Captures and reports NSExceptions. */
    KSCrashMonitorTypeNSException        = 0x08,
    
    /* Detects and reports a deadlock in the main thread. */
    KSCrashMonitorTypeMainThreadDeadlock = 0x10,
    
    /* Accepts and reports user-generated exceptions. */
    KSCrashMonitorTypeUserReported       = 0x20,
    
    /* Keeps track of and injects system information. */
    KSCrashMonitorTypeSystem             = 0x40,
    
    /* Keeps track of and injects application state. */
    KSCrashMonitorTypeApplicationState   = 0x80,
    
    /* Keeps track of zombies, and injects the last zombie NSException. */
    KSCrashMonitorTypeZombie             = 0x100,
} KSCrashMonitorType;

#define KSCrashMonitorTypeAll              \
(                                          \
    KSCrashMonitorTypeMachException      | \
    KSCrashMonitorTypeSignal             | \
    KSCrashMonitorTypeCPPException       | \
    KSCrashMonitorTypeNSException        | \
    KSCrashMonitorTypeMainThreadDeadlock | \
    KSCrashMonitorTypeUserReported       | \
    KSCrashMonitorTypeSystem             | \
    KSCrashMonitorTypeApplicationState   | \
    KSCrashMonitorTypeZombie               \
)
```

可以看到，all 代表的就是所有异常方式：

1. Mach 异常
2. Signal 异常
3. C++ 异常
4. OC 异常
5. 死锁
6. 用户抛出的异常

设置 monitor 的时候，会判断是否在被调试，这是通过 `sysctl` 获取进程的信息的方式来进行判断的：

```c
/// 是否在被调试
bool ksdebug_isBeingTraced(void)
{
    /// 查询进程信息结果的结构体
    struct kinfo_proc procInfo;
    size_t structSize = sizeof(procInfo);
    int mib[] = {CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()};
    
    /// 通过 sysctl 获取进程信息
    if(sysctl(mib, sizeof(mib)/sizeof(*mib), &procInfo, &structSize, NULL, 0) != 0)
    {
        KSLOG_ERROR("sysctl: %s", strerror(errno));
        return false;
    }
    
    /// 通过进程信息判断是否被调试
    return (procInfo.kp_proc.p_flag & P_TRACED) != 0;
}
```

最终通过 for 循环，遍历启动每一个 monitor。启动某个 monitor 的方法如下：

```c
/// 启动某个 monitor
static inline void setMonitorEnabled(Monitor* monitor, bool isEnabled)
{
    KSCrashMonitorAPI* api = getAPI(monitor);
    /// 调用各个 monitor 的 enable 方法，启动 monitor
    if(api != NULL && api->setEnabled != NULL)
    {
        api->setEnabled(isEnabled);
    }
}
```

`Monitor` 是一个结构体，它保存了 Monitor 的类型以及启动 monitor 的方法：

```c
/// monitor 的数据结构
typedef struct
{
    KSCrashMonitorType monitorType;
    KSCrashMonitorAPI* (*getAPI)(void);
} Monitor;
```

也就是说，每一种异常都会有一个自己的 `setEnabled()` 方法，用于启动自身。

### Mach 异常

#### 开启捕获

Mach 异常在 `KSCrashMonitor_MachException.c` 中被处理外部调用 `setEnabled()` 开启 Mach 异常的监听：

```c
static void setEnabled(bool isEnabled)
{
    if(isEnabled != g_isEnabled)
    {
        g_isEnabled = isEnabled;
        if(isEnabled)
        {
            ksid_generate(g_primaryEventID);
            ksid_generate(g_secondaryEventID);
            if(!installExceptionHandler())
            {
                return;
            }
        }
        else
        {
            uninstallExceptionHandler();
        }
    }
}
```

其中包含两个 c 函数，`installExceptionHandler()` 和 `uninstallExceptionHandler()` 分别用于开启和关闭监听。

> 在说如何捕捉 Mach 异常前，先说一下什么是 Mach。Mach:[mʌk]，是一种操作系统微内核，是许多新操作系统的设计基础。
>
>  Mach微内核中有几个基础概念：
>  - Tasks，拥有一组系统资源的对象，允许”thread”在其中执行。**每一个BSD 进程都在底层关联了一个Mach 任务对象**(BSD 层在 Mach 之上，提供了更高层次的功能，比如 UNIX 进程模型，POSIX 线程模型等)
>  - Threads，执行的基本单位，拥有task的上下文，并共享其资源。
>  - Ports，task之间通讯的一组受保护的消息队列；task可对任何port发送/接收数据。
>  - Message，有类型的数据对象集合，只可以发送到port。
>
> 关于什么是微内核？微内核把系统服务，比如文件管理、虚拟内存、设备I/O，单独包装为一个个模块。微内核作为底层使用进程间通信收发消息。这一来大量的内核代码可以转移到用户空间，使内核变得更小。这种方式拓展性强，但是通信有效率损耗。宏内核正相反，把所有系统服务放在一起。
>
> UNIX 下，触发到内核态需要通过系统调用触发陷阱。Mach 可以通过申请 port，然后利用IPC机制向这个port发送消息。
>
> 陷阱是软中断，是主动触发的，立刻同步处理。异常是当前指令执行出现问题，是被动的，立刻同步处理。中断是外部硬件触发的，是异步的。

回到 Mach 异常的注册方法中，我们要知道，当异常发生时，内核会向当前 task 的某个专门处理异常的 port 发消息，该消息会依次被转为 signal，NSException 抛出。如果我们要在 Mach 层捕获异常，就需要注册自己的 port，来接收这个异常：

```c
/// 创建 mach 捕捉者
static bool installExceptionHandler()
{
    KSLOG_DEBUG("Installing mach exception handler.");

    bool attributes_created = false;
    pthread_attr_t attr;

    kern_return_t kr;
    int error;

    /// 获取当前进程的 task
    const task_t thisTask = mach_task_self();
    exception_mask_t mask = EXC_MASK_BAD_ACCESS |
    EXC_MASK_BAD_INSTRUCTION |
    EXC_MASK_ARITHMETIC |
    EXC_MASK_SOFTWARE |
    EXC_MASK_BREAKPOINT;

    KSLOG_DEBUG("Backing up original exception ports.");
    /// 保存之前的异常处理端口到 g_previousExceptionPorts 中
    kr = task_get_exception_ports(thisTask,
                                  mask,
                                  g_previousExceptionPorts.masks,
                                  &g_previousExceptionPorts.count,
                                  g_previousExceptionPorts.ports,
                                  g_previousExceptionPorts.behaviors,
                                  g_previousExceptionPorts.flavors);
    if(kr != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_get_exception_ports: %s", mach_error_string(kr));
        goto failed;
    }

    /// 如果自己的异常处理端口 g_exceptionPort 是空的，那么创建
    if(g_exceptionPort == MACH_PORT_NULL)
    {
        KSLOG_DEBUG("Allocating new port with receive rights.");
        /// 创建新的异常处理端口
        kr = mach_port_allocate(thisTask,
                                MACH_PORT_RIGHT_RECEIVE,
                                &g_exceptionPort);
        if(kr != KERN_SUCCESS)
        {
            KSLOG_ERROR("mach_port_allocate: %s", mach_error_string(kr));
            goto failed;
        }

        KSLOG_DEBUG("Adding send rights to port.");
        /// 申请端口权限
        kr = mach_port_insert_right(thisTask,
                                    g_exceptionPort,
                                    g_exceptionPort,
                                    MACH_MSG_TYPE_MAKE_SEND);
        if(kr != KERN_SUCCESS)
        {
            KSLOG_ERROR("mach_port_insert_right: %s", mach_error_string(kr));
            goto failed;
        }
    }

    KSLOG_DEBUG("Installing port as exception handler.");
    /// 把异常设置为自己的 port
    kr = task_set_exception_ports(thisTask,
                                  mask,
                                  g_exceptionPort,
                                  (int)(EXCEPTION_DEFAULT | MACH_EXCEPTION_CODES),
                                  THREAD_STATE_NONE);
    if(kr != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_set_exception_ports: %s", mach_error_string(kr));
        goto failed;
    }

    KSLOG_DEBUG("Creating secondary exception thread (suspended).");
    /// 以下整个部分用来创建读取异常端口数据的线程,设置异常端口的处理函数
    pthread_attr_init(&attr);
    attributes_created = true;
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    /// 创建第二条处理crash的线程，以防主z处理crash的线程crash了
    error = pthread_create(&g_secondaryPThread,
                           &attr,
                           &handleExceptions,
                           kThreadSecondary);
    if(error != 0)
    {
        KSLOG_ERROR("pthread_create_suspended_np: %s", strerror(error));
        goto failed;
    }
    g_secondaryMachThread = pthread_mach_thread_np(g_secondaryPThread);
    ksmc_addReservedThread(g_secondaryMachThread);

    KSLOG_DEBUG("Creating primary exception thread.");
    /// 创建主处理crash的线程
    error = pthread_create(&g_primaryPThread,
                           &attr,
                           &handleExceptions,
                           kThreadPrimary);
    if(error != 0)
    {
        KSLOG_ERROR("pthread_create: %s", strerror(error));
        goto failed;
    }
    pthread_attr_destroy(&attr);
    g_primaryMachThread = pthread_mach_thread_np(g_primaryPThread);
    ksmc_addReservedThread(g_primaryMachThread);

    KSLOG_DEBUG("Mach exception handler installed.");
    return true;


failed:
    KSLOG_DEBUG("Failed to install mach exception handler.");
    if(attributes_created)
    {
        pthread_attr_destroy(&attr);
    }
    uninstallExceptionHandler();
    return false;
}
```

代码很长，关键地方做了注释：

1. 通过 `mach_task_self()` 获取当前进程对应的 task。
2. 通过 `task_get_exception_ports()` 获取原本处理异常的 port，并保存在 `g_previousExceptionPorts` 中。
3. 通过 `mach_port_allocate()` 创建新的异常处理端口。
4. 通过 `mach_port_insert_right()` 给这个新创建的端口申请权限
5. 通过 `task_set_exception_ports()` 把异常接收的 port 设置为自己新创建的 port
6. 创建好 port 之后，就可以创建自己的线程去一直读取 port 上的消息了。通过 `pthread_create()` 创建自己的线程，以及设置好执行的方法为 `handleExceptions`。

#### 处理异常

现在就是关键的处理方法 `handleExceptions()` 了。这也是一个非常长的方法：

```objc
/// 处理 Exception
static void* handleExceptions(void* const userData)
{
    MachExceptionMessage exceptionMessage = {{0}};
    MachReplyMessage replyMessage = {{0}};
    char* eventID = g_primaryEventID;

    const char* threadName = (const char*) userData;
    pthread_setname_np(threadName);
    if(threadName == kThreadSecondary)
    {
        KSLOG_DEBUG("This is the secondary thread. Suspending.");
        thread_suspend((thread_t)ksthread_self());
        eventID = g_secondaryEventID;
    }

    for(;;)
    {
        KSLOG_DEBUG("Waiting for mach exception");

        // Wait for a message.
        /// 不断调用 mach_msg 接收消息，从异常端口中读取信息到 exceptionMessage 中
        kern_return_t kr = mach_msg(&exceptionMessage.header,
                                    MACH_RCV_MSG,
                                    0,
                                    sizeof(exceptionMessage),
                                    g_exceptionPort,
                                    MACH_MSG_TIMEOUT_NONE,
                                    MACH_PORT_NULL);
        /// 上面一直循环读取，直到读取成功了，进入后面的处理函数中
        if(kr == KERN_SUCCESS)
        {
            break;
        }

        // Loop and try again on failure.
        KSLOG_ERROR("mach_msg: %s", mach_error_string(kr));
    }

    KSLOG_DEBUG("Trapped mach exception code 0x%llx, subcode 0x%llx",
                exceptionMessage.code[0], exceptionMessage.code[1]);
    if(g_isEnabled)
    {
        thread_act_array_t threads = NULL;
        mach_msg_type_number_t numThreads = 0;
        /// 暂停所有非当前线程以及白名单线程的线程
        ksmc_suspendEnvironment(&threads, &numThreads);
        g_isHandlingCrash = true;
        /// 捕捉到异常之后清除所有的 monitor
        kscm_notifyFatalExceptionCaptured(true);

        KSLOG_DEBUG("Exception handler is installed. Continuing exception handling.");


        // Switch to the secondary thread if necessary, or uninstall the handler
        // to avoid a death loop.
        /// 捕捉到 exception 后，恢复原来的 port
        if(ksthread_self() == g_primaryMachThread)
        {
            KSLOG_DEBUG("This is the primary exception thread. Activating secondary thread.");
// TODO: This was put here to avoid a freeze. Does secondary thread ever fire?
            restoreExceptionPorts();
            if(thread_resume(g_secondaryMachThread) != KERN_SUCCESS)
            {
                KSLOG_DEBUG("Could not activate secondary thread. Restoring original exception ports.");
            }
        }
        else
        {
            KSLOG_DEBUG("This is the secondary exception thread.");// Restoring original exception ports.");
//            restoreExceptionPorts();
        }

        // Fill out crash information
        /// 设置 crash 信息的 context
        KSLOG_DEBUG("Fetching machine state.");
        /// 创建一个 machineContext 用来保存异常信息
        KSMC_NEW_CONTEXT(machineContext);
        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        crashContext->offendingMachineContext = machineContext;
        /// 创建一个遍历调用栈的 cursor
        kssc_initCursor(&g_stackCursor, NULL, NULL);
        /// 把线程信息附加到 machineContext 上
        if(ksmc_getContextForThread(exceptionMessage.thread.name, machineContext, true))
        {
            kssc_initWithMachineContext(&g_stackCursor, 100, machineContext);
            KSLOG_TRACE("Fault address %p, instruction address %p",
                        kscpu_faultAddress(machineContext), kscpu_instructionAddress(machineContext));
            if(exceptionMessage.exception == EXC_BAD_ACCESS)
            {
                crashContext->faultAddress = kscpu_faultAddress(machineContext);
            }
            else
            {
                crashContext->faultAddress = kscpu_instructionAddress(machineContext);
            }
        }

        KSLOG_DEBUG("Filling out context.");
        crashContext->crashType = KSCrashMonitorTypeMachException;
        crashContext->eventID = eventID;
        crashContext->registersAreValid = true;
        crashContext->mach.type = exceptionMessage.exception;
        crashContext->mach.code = exceptionMessage.code[0] & (int64_t)MACH_ERROR_CODE_MASK;
        crashContext->mach.subcode = exceptionMessage.code[1] & (int64_t)MACH_ERROR_CODE_MASK;
        if(crashContext->mach.code == KERN_PROTECTION_FAILURE && crashContext->isStackOverflow)
        {
            // A stack overflow should return KERN_INVALID_ADDRESS, but
            // when a stack blasts through the guard pages at the top of the stack,
            // it generates KERN_PROTECTION_FAILURE. Correct for this.
            crashContext->mach.code = KERN_INVALID_ADDRESS;
        }
        /// 将 mach 异常转为对应的 signal
        crashContext->signal.signum = signalForMachException(crashContext->mach.type, crashContext->mach.code);
        crashContext->stackCursor = &g_stackCursor;

        /// context 交给 kscrashmonitor 处理
        kscm_handleException(crashContext);

        KSLOG_DEBUG("Crash handling complete. Restoring original handlers.");
        g_isHandlingCrash = false;
        /// 结束了捕获恢复所有线程
        ksmc_resumeEnvironment(threads, numThreads);
    }

    KSLOG_DEBUG("Replying to mach exception message.");
    // Send a reply saying "I didn't handle this exception".
    replyMessage.header = exceptionMessage.header;
    replyMessage.NDR = exceptionMessage.NDR;
    replyMessage.returnCode = KERN_FAILURE;

    /// 发消息告知没有处理这个异常
    mach_msg(&replyMessage.header,
             MACH_SEND_MSG,
             sizeof(replyMessage),
             0,
             MACH_PORT_NULL,
             MACH_MSG_TIMEOUT_NONE,
             MACH_PORT_NULL);

    return NULL;
}
```

整体的处理方式如下：

1. 不停循环通过 `mach_msg()` 读取 port 中传来的消息。
2. 读取成功后挂起所有线程。
3. 清除所有的 monitor，恢复原来的 port
4. 抓取所有线程的信息保存到 `KSMachineContext` 结构体中
5. 将各种信息交给 `crashContext`
6. 把 `crashContext` 抛出给外部处理方法
7. 恢复所有的线程
8. 通过 `mach_msg()` 再发出一个消息告知没有处理这个异常

通过方法 `task_threads()` 获取当前 task 的所有线程，通过 `thread_suspend()` 方法挂起某个线程：

```c
/// suspend 当前 task 内非处理 crash 或者白名单内的线程。
void ksmc_suspendEnvironment(thread_act_array_t *suspendedThreads, mach_msg_type_number_t *numSuspendedThreads)
{
#if KSCRASH_HAS_THREADS_API
    KSLOG_DEBUG("Suspending environment.");
    kern_return_t kr;
    const task_t thisTask = mach_task_self();
    const thread_t thisThread = (thread_t)ksthread_self();
    
    /// 获取当前task的所有thread
    if((kr = task_threads(thisTask, suspendedThreads, numSuspendedThreads)) != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_threads: %s", mach_error_string(kr));
        return;
    }
    
    /// 遍历所有 thread，如果不是当前处理 crash 的 thread，以及要保留的 thread 就直接 suspend 它们
    for(mach_msg_type_number_t i = 0; i < *numSuspendedThreads; i++)
    {
        thread_t thread = (*suspendedThreads)[i];
        if(thread != thisThread && !isThreadInList(thread, g_reservedThreads, g_reservedThreadsCount))
        {
            if((kr = thread_suspend(thread)) != KERN_SUCCESS)
            {
                // Record the error and keep going.
                KSLOG_ERROR("thread_suspend (%08x): %s", thread, mach_error_string(kr));
            }
        }
    }
    
    KSLOG_DEBUG("Suspend complete.");
#endif
}
```

通过 `task_threads()` 方法获取所有的 thread：

```c
/// 获取当前 task 所有的 thread，把它保存到 context 中
static inline bool getThreadList(KSMachineContext* context)
{
    const task_t thisTask = mach_task_self();
    KSLOG_DEBUG("Getting thread list");
    kern_return_t kr;
    thread_act_array_t threads;
    mach_msg_type_number_t actualThreadCount;

    /// task_thread 获取所有的 thread
    if((kr = task_threads(thisTask, &threads, &actualThreadCount)) != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_threads: %s", mach_error_string(kr));
        return false;
    }
    KSLOG_TRACE("Got %d threads", context->threadCount);
    int threadCount = (int)actualThreadCount;
    int maxThreadCount = sizeof(context->allThreads) / sizeof(context->allThreads[0]);
    if(threadCount > maxThreadCount)
    {
        KSLOG_ERROR("Thread count %d is higher than maximum of %d", threadCount, maxThreadCount);
        threadCount = maxThreadCount;
    }
    for(int i = 0; i < threadCount; i++)
    {
        /// 把 thread 保存到 context 中
        context->allThreads[i] = threads[i];
    }
    context->threadCount = threadCount;

    for(mach_msg_type_number_t i = 0; i < actualThreadCount; i++)
    {
        mach_port_deallocate(thisTask, context->allThreads[i]);
    }
    vm_deallocate(thisTask, (vm_address_t)threads, sizeof(thread_t) * actualThreadCount);

    return true;
}
```

恢复线程的时候，使用相应的 `thread_resume()` 方法即可：

```c
/// 恢复所有线程
void ksmc_resumeEnvironment(thread_act_array_t threads, mach_msg_type_number_t numThreads)
{
#if KSCRASH_HAS_THREADS_API
    KSLOG_DEBUG("Resuming environment.");
    kern_return_t kr;
    const task_t thisTask = mach_task_self();
    const thread_t thisThread = (thread_t)ksthread_self();
    
    if(threads == NULL || numThreads == 0)
    {
        KSLOG_ERROR("we should call ksmc_suspendEnvironment() first");
        return;
    }
    
    for(mach_msg_type_number_t i = 0; i < numThreads; i++)
    {
        thread_t thread = threads[i];
        if(thread != thisThread && !isThreadInList(thread, g_reservedThreads, g_reservedThreadsCount))
        {
            if((kr = thread_resume(thread)) != KERN_SUCCESS)
            {
                // Record the error and keep going.
                KSLOG_ERROR("thread_resume (%08x): %s", thread, mach_error_string(kr));
            }
        }
    }
    
    for(mach_msg_type_number_t i = 0; i < numThreads; i++)
    {
        mach_port_deallocate(thisTask, threads[i]);
    }
    vm_deallocate(thisTask, (vm_address_t)threads, sizeof(thread_t) * numThreads);
    
    KSLOG_DEBUG("Resume complete.");
#endif
}
```

所有 Mach 异常都在 BSD 层被 `ux_exception` 转换为相应的 Unix 信号，并通过 `threadsignal` 将信号投递到出错的线程。在 Mach 层，我们可以直接通过 Mach 异常推测出 Signal 类型：

```c
// 将 mach 转为 signal
static int signalForMachException(exception_type_t exception, mach_exception_code_t code)
{
    switch(exception)
    {
        case EXC_ARITHMETIC:
            return SIGFPE;
        case EXC_BAD_ACCESS:
            return code == KERN_INVALID_ADDRESS ? SIGSEGV : SIGBUS;
        case EXC_BAD_INSTRUCTION:
            return SIGILL;
        case EXC_BREAKPOINT:
            return SIGTRAP;
        case EXC_EMULATION:
            return SIGEMT;
        case EXC_SOFTWARE:
        {
            switch (code)
            {
                case EXC_UNIX_BAD_SYSCALL:
                    return SIGSYS;
                case EXC_UNIX_BAD_PIPE:
                    return SIGPIPE;
                case EXC_UNIX_ABORT:
                    return SIGABRT;
                case EXC_SOFT_SIGNAL:
                    return SIGKILL;
            }
            break;
        }
    }
    return 0;
}
```

#### 取消监听

在捕获到异常后就要取消原本的监听，主要分为两步：

1. 将原本用作监听异常的 port 恢复
2. 结束自己创建的用于处理异常的线程

```c
static void uninstallExceptionHandler()
{
    KSLOG_DEBUG("Uninstalling mach exception handler.");
    
    // NOTE: Do not deallocate the exception port. If a secondary crash occurs
    // it will hang the process.
    
    /// 恢复原本的mach处理端口
    restoreExceptionPorts();
    
    thread_t thread_self = (thread_t)ksthread_self();
    
    /// 当前不是 primary 处理 crash 的线程，那么终止线程
    if(g_primaryPThread != 0 && g_primaryMachThread != thread_self)
    {
        KSLOG_DEBUG("Canceling primary exception thread.");
        if(g_isHandlingCrash)
        {
            thread_terminate(g_primaryMachThread);
        }
        else
        {
            pthread_cancel(g_primaryPThread);
        }
        g_primaryMachThread = 0;
        g_primaryPThread = 0;
    }
    /// 当前不是备用处理 crash 的线程，那么终止线程
    if(g_secondaryPThread != 0 && g_secondaryMachThread != thread_self)
    {
        KSLOG_DEBUG("Canceling secondary exception thread.");
        if(g_isHandlingCrash)
        {
            thread_terminate(g_secondaryMachThread);
        }
        else
        {
            pthread_cancel(g_secondaryPThread);
        }
        g_secondaryMachThread = 0;
        g_secondaryPThread = 0;
    }
    
    g_exceptionPort = MACH_PORT_NULL;
    KSLOG_DEBUG("Mach exception handlers uninstalled.");
}

/// 恢复原本的 mach 处理端口
static void restoreExceptionPorts(void)
{
    KSLOG_DEBUG("Restoring original exception ports.");
    if(g_previousExceptionPorts.count == 0)
    {
        KSLOG_DEBUG("Original exception ports were already restored.");
        return;
    }

    const task_t thisTask = mach_task_self();
    kern_return_t kr;

    // Reinstall old exception ports.
    for(mach_msg_type_number_t i = 0; i < g_previousExceptionPorts.count; i++)
    {
        KSLOG_TRACE("Restoring port index %d", i);
        /// 将 port 设置为原来的
        kr = task_set_exception_ports(thisTask,
                                      g_previousExceptionPorts.masks[i],
                                      g_previousExceptionPorts.ports[i],
                                      g_previousExceptionPorts.behaviors[i],
                                      g_previousExceptionPorts.flavors[i]);
        if(kr != KERN_SUCCESS)
        {
            KSLOG_ERROR("task_set_exception_ports: %s",
                        mach_error_string(kr));
        }
    }
    KSLOG_DEBUG("Exception ports restored.");
    g_previousExceptionPorts.count = 0;
}

```

### Signal 异常

前面说过，Mach 异常会在 BSD 层转化为相应的 UNIX 信号，投递到相应的线程中。我们同样可以捕捉相应的 Signal。

#### 开启捕获

同样是调用 `setEnabled()` 方法，它会执行到 `installSignalHandler()` 方法中。相关的参数设置比较多，坦白来说确实不太好理解它们的作用，不过其实粗略的看下来也不影响对于主流程的理解。总的来说就是通过 `sigaction()` 方法记录下某个 siganl 对应的处理方法，并且保存先前的处理方法：

```c
/// 创建 signal 的捕捉者
static bool installSignalHandler()
{
    KSLOG_DEBUG("Installing signal handler.");

#if KSCRASH_HAS_SIGNAL_STACK

    if(g_signalStack.ss_size == 0)
    {
        KSLOG_DEBUG("Allocating signal stack area.");
        g_signalStack.ss_size = SIGSTKSZ;
        g_signalStack.ss_sp = malloc(g_signalStack.ss_size);
    }

    KSLOG_DEBUG("Setting signal stack area.");
    if(sigaltstack(&g_signalStack, NULL) != 0)
    {
        KSLOG_ERROR("signalstack: %s", strerror(errno));
        goto failed;
    }
#endif

    /// 需要监听的 signal 数组
    const int* fatalSignals = kssignal_fatalSignals();
    /// 需要监听的 signal 数组大小
    int fatalSignalsCount = kssignal_numFatalSignals();

    if(g_previousSignalHandlers == NULL)
    {
        KSLOG_DEBUG("Allocating memory to store previous signal handlers.");
        g_previousSignalHandlers = malloc(sizeof(*g_previousSignalHandlers)
                                          * (unsigned)fatalSignalsCount);
    }

    struct sigaction action = {{0}};
    action.sa_flags = SA_SIGINFO | SA_ONSTACK;
#if KSCRASH_HOST_APPLE && defined(__LP64__)
    action.sa_flags |= SA_64REGSET;
#endif
    sigemptyset(&action.sa_mask);
    action.sa_sigaction = &handleSignal;

    for(int i = 0; i < fatalSignalsCount; i++)
    {
        KSLOG_DEBUG("Assigning handler for signal %d", fatalSignals[i]);
        /// 设置该 signal 对应的处理方法，并且保存原始的处理方法
        if(sigaction(fatalSignals[i], &action, &g_previousSignalHandlers[i]) != 0)
        {
            /// 设置失败的时候走下面的方法
            char sigNameBuff[30];
            const char* sigName = kssignal_signalName(fatalSignals[i]);
            if(sigName == NULL)
            {
                snprintf(sigNameBuff, sizeof(sigNameBuff), "%d", fatalSignals[i]);
                sigName = sigNameBuff;
            }
            KSLOG_ERROR("sigaction (%s): %s", sigName, strerror(errno));
            // Try to reverse the damage
            for(i--;i >= 0; i--)
            {
                sigaction(fatalSignals[i], &g_previousSignalHandlers[i], NULL);
            }
            goto failed;
        }
    }
    KSLOG_DEBUG("Signal handlers installed.");
    return true;

failed:
    KSLOG_DEBUG("Failed to install signal handlers.");
    return false;
}
```

fatal_signal 包括如下：

```c
static const int g_fatalSignals[] =
{
    SIGABRT,
    SIGBUS,
    SIGFPE,
    SIGILL,
    SIGPIPE,
    SIGSEGV,
    SIGSYS,
    SIGTRAP,
};
```

#### 处理异常

处理异常的回调方法中会返回 signal 信息，以及一个 context：

```c
static void handleSignal(int sigNum, siginfo_t* signalInfo, void* userContext)
{
    KSLOG_DEBUG("Trapped signal %d", sigNum);
    if(g_isEnabled)
    {
        thread_act_array_t threads = NULL;
        mach_msg_type_number_t numThreads = 0;
        /// 暂停线程
        ksmc_suspendEnvironment(&threads, &numThreads);
        /// 通知已经捕获到异常了
        kscm_notifyFatalExceptionCaptured(false);

        KSLOG_DEBUG("Filling out context.");
        KSMC_NEW_CONTEXT(machineContext);
        /// 保存 context 到 machineContext 中，并且获取 thread 信息
        ksmc_getContextForSignal(userContext, machineContext);
        /// 把 machineContext 放到  g_stackCursor 中
        kssc_initWithMachineContext(&g_stackCursor, 100, machineContext);

        /// 生成真正的 context
        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        memset(crashContext, 0, sizeof(*crashContext));
        crashContext->crashType = KSCrashMonitorTypeSignal;
        crashContext->eventID = g_eventID;
        crashContext->offendingMachineContext = machineContext;
        crashContext->registersAreValid = true;
        crashContext->faultAddress = (uintptr_t)signalInfo->si_addr;
        crashContext->signal.userContext = userContext;
        crashContext->signal.signum = signalInfo->si_signo;
        crashContext->signal.sigcode = signalInfo->si_code;
        crashContext->stackCursor = &g_stackCursor;

        /// 把 context 传给外部处理函数
        kscm_handleException(crashContext);
        /// 恢复原来的环境
        ksmc_resumeEnvironment(threads, numThreads);
    }

    KSLOG_DEBUG("Re-raising signal for regular handlers to catch.");
    // This is technically not allowed, but it works in OSX and iOS.
    /// 重新抛出 signal
    raise(sigNum);
}
```

整个流程和 Mach 异常还是非常类似的，先暂停线程，然后读取线程信息，再把 signal 信息线程信息保存到 context 中，传递给外部的处理函数。最后恢复原来的环境。

#### 取消监听

```c
/// 取消捕捉 signal
static void uninstallSignalHandler(void)
{
    KSLOG_DEBUG("Uninstalling signal handlers.");

    const int* fatalSignals = kssignal_fatalSignals();
    int fatalSignalsCount = kssignal_numFatalSignals();

    for(int i = 0; i < fatalSignalsCount; i++)
    {
        KSLOG_DEBUG("Restoring original handler for signal %d", fatalSignals[i]);
        sigaction(fatalSignals[i], &g_previousSignalHandlers[i], NULL);
    }
    
#if KSCRASH_HAS_SIGNAL_STACK
    g_signalStack = (stack_t){0};
#endif
    KSLOG_DEBUG("Signal handlers uninstalled.");
}

```

取消捕捉的方式和启动捕捉类似，都是通过 `sigaction()` 方法，不同的是，现在将原本的处理方法传回。

### C++ 异常

#### 开启捕获

通过 `set_terminate()` 方法设置自己的捕获函数：

```c++
static void setEnabled(bool isEnabled)
{
    if(isEnabled != g_isEnabled)
    {
        g_isEnabled = isEnabled;
        if(isEnabled)
        {
            initialize();

            ksid_generate(g_eventID);
            /// 保存原始的 c++ 处理 handler，设置自己的处理 handler
            g_originalTerminateHandler = std::set_terminate(CPPExceptionTerminate);
        }
        else
        {
            /// 恢复原始的 c++ 处理 handler
            std::set_terminate(g_originalTerminateHandler);
        }
        g_captureNextStackTrace = isEnabled;
    }
}
```

#### 处理异常

```c++
/// c++ 处理函数
static void CPPExceptionTerminate(void)
{
    thread_act_array_t threads = NULL;
    mach_msg_type_number_t numThreads = 0;
    /// 挂起非处理现场和白名单线程的其他所有线程
    ksmc_suspendEnvironment(&threads, &numThreads);
    KSLOG_DEBUG("Trapped c++ exception");
    const char* name = NULL;
    std::type_info* tinfo = __cxxabiv1::__cxa_current_exception_type();
    if(tinfo != NULL)
    {
        name = tinfo->name();
    }
    
    if(name == NULL || strcmp(name, "NSException") != 0)
    {
        /// 捕捉到 crash 后，清空 KSCrash 的所有 monitor
        kscm_notifyFatalExceptionCaptured(false);
        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        memset(crashContext, 0, sizeof(*crashContext));

        char descriptionBuff[DESCRIPTION_BUFFER_LENGTH];
        const char* description = descriptionBuff;
        descriptionBuff[0] = 0;

        KSLOG_DEBUG("Discovering what kind of exception was thrown.");
        g_captureNextStackTrace = false;
        try
        {
            throw;
        }
        catch(std::exception& exc)
        {
            strncpy(descriptionBuff, exc.what(), sizeof(descriptionBuff));
        }
#define CATCH_VALUE(TYPE, PRINTFTYPE) \
catch(TYPE value)\
{ \
    snprintf(descriptionBuff, sizeof(descriptionBuff), "%" #PRINTFTYPE, value); \
}
        CATCH_VALUE(char,                 d)
        CATCH_VALUE(short,                d)
        CATCH_VALUE(int,                  d)
        CATCH_VALUE(long,                ld)
        CATCH_VALUE(long long,          lld)
        CATCH_VALUE(unsigned char,        u)
        CATCH_VALUE(unsigned short,       u)
        CATCH_VALUE(unsigned int,         u)
        CATCH_VALUE(unsigned long,       lu)
        CATCH_VALUE(unsigned long long, llu)
        CATCH_VALUE(float,                f)
        CATCH_VALUE(double,               f)
        CATCH_VALUE(long double,         Lf)
        CATCH_VALUE(char*,                s)
        catch(...)
        {
            description = NULL;
        }
        g_captureNextStackTrace = g_isEnabled;

        // TODO: Should this be done here? Maybe better in the exception handler?
        KSMC_NEW_CONTEXT(machineContext);
        ksmc_getContextForThread(ksthread_self(), machineContext, true);

        KSLOG_DEBUG("Filling out context.");
        crashContext->crashType = KSCrashMonitorTypeCPPException;
        crashContext->eventID = g_eventID;
        crashContext->registersAreValid = false;
        crashContext->stackCursor = &g_stackCursor;
        crashContext->CPPException.name = name;
        crashContext->exceptionName = name;
        crashContext->crashReason = description;
        crashContext->offendingMachineContext = machineContext;

        /// 处理异常
        kscm_handleException(crashContext);
    }
    else
    {
        KSLOG_DEBUG("Detected NSException. Letting the current NSException handler deal with it.");
    }
    /// 恢复线程
    ksmc_resumeEnvironment(threads, numThreads);

    KSLOG_DEBUG("Calling original terminate handler.");
    /// 触发原本的  handler
    g_originalTerminateHandler();
}

```

### NSException 异常

#### 开启捕获

到了 NSException 部分，很多文章中也都有提到，先通过 `NSGetUncaughtExceptionHandler()` 获取原先的异常处理函数，然后再通过  `NSSetUncaughtExceptionHandler()` 方法设置自己的处理函数。

```c
static void setEnabled(bool isEnabled)
{
    if(isEnabled != g_isEnabled)
    {
        g_isEnabled = isEnabled;
        if(isEnabled)
        {
            KSLOG_DEBUG(@"Backing up original handler.");
            /// 拿到原来的 handler
            g_previousUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
            
            KSLOG_DEBUG(@"Setting new handler.");
            /// 设置新的 handler
            NSSetUncaughtExceptionHandler(&handleUncaughtException);
            KSCrash.sharedInstance.uncaughtExceptionHandler = &handleUncaughtException;
            KSCrash.sharedInstance.currentSnapshotUserReportedExceptionHandler = &handleCurrentSnapshotUserReportedException;
        }
        else
        {
            KSLOG_DEBUG(@"Restoring original handler.");
            /// 设置回原来的 handler
            NSSetUncaughtExceptionHandler(g_previousUncaughtExceptionHandler);
        }
    }
}
```

#### 处理异常

关于调用堆栈的获取都不需要通过 task 了，直接从 NSException 就可以获取到：

```c
/// 处理方法
static void handleException(NSException* exception, BOOL currentSnapshotUserReported) {
    KSLOG_DEBUG(@"Trapped exception %@", exception);
    if(g_isEnabled)
    {
        thread_act_array_t threads = NULL;
        mach_msg_type_number_t numThreads = 0;
        ksmc_suspendEnvironment(&threads, &numThreads);
        kscm_notifyFatalExceptionCaptured(false);

        KSLOG_DEBUG(@"Filling out context.");
        /// 调用堆栈的地址
        NSArray* addresses = [exception callStackReturnAddresses];
        NSUInteger numFrames = addresses.count;
        uintptr_t* callstack = malloc(numFrames * sizeof(*callstack));
        /// 转为堆栈
        for(NSUInteger i = 0; i < numFrames; i++)
        {
            callstack[i] = (uintptr_t)[addresses[i] unsignedLongLongValue];
        }

        char eventID[37];
        ksid_generate(eventID);
        KSMC_NEW_CONTEXT(machineContext);
        ksmc_getContextForThread(ksthread_self(), machineContext, true);
        KSStackCursor cursor;
        kssc_initWithBacktrace(&cursor, callstack, (int)numFrames, 0);

        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        memset(crashContext, 0, sizeof(*crashContext));
        crashContext->crashType = KSCrashMonitorTypeNSException;
        crashContext->eventID = eventID;
        crashContext->offendingMachineContext = machineContext;
        crashContext->registersAreValid = false;
        crashContext->NSException.name = [[exception name] UTF8String];
        crashContext->NSException.userInfo = [[NSString stringWithFormat:@"%@", exception.userInfo] UTF8String];
        crashContext->exceptionName = crashContext->NSException.name;
        crashContext->crashReason = [[exception reason] UTF8String];
        crashContext->stackCursor = &cursor;
        crashContext->currentSnapshotUserReported = currentSnapshotUserReported;

        KSLOG_DEBUG(@"Calling main crash handler.");
        kscm_handleException(crashContext);

        free(callstack);
        if (currentSnapshotUserReported) {
            ksmc_resumeEnvironment(threads, numThreads);
        }
        if (g_previousUncaughtExceptionHandler != NULL)
        {
            KSLOG_DEBUG(@"Calling original exception handler.");
            g_previousUncaughtExceptionHandler(exception);
        }
    }
}
```

### DeadLock 

死锁部分我就不细说了，原理和在前一篇 “iOS开发高手课” 笔记中有说明，BeeHive 中也有类似的的实现。主要是通过子线程循环往复的判断一个标志位是否在主线程中被修改过。来达到监听的效果。

## 总结

至此，捕获异常相关的几个方法都已经分析完了。其实过程还是非常类似的：替换原来的捕获处理，捕获到异常后保存 context 信息，暂停所有线程获取所有线程信息，恢复原本的捕获处理方法，调用统一的异常处理函数。

下一篇将介绍异常捕获后，如何分析 context 以及线程信息。





























