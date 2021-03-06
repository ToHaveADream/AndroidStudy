# Android 全面解析之 Handler 机制（五）：再认识 Handler



### 工作流程

看下整体的流程：

![](/picture/data-08.image)

1. Handler 设置一系列的 api 供给开发者可以使用 Handler 发送各种类型的消息，最终调用到了 enqueueMessage 方法来入队
2. 调用 MessageQueue 的 enqueueMessage 方法把消息插入到 MessageQueue 的链表中，等待被 Looper 获取处理
3. Looper 获取到 Message 之后，调用 Message 对应的 Handler 处理 Message

Handler 机制使用的是多线程的思路，主线程不断地等待消息，然后从别的线程发送过来消息让主线程进行执行，这称为：事务驱动型设计，主线程地逻辑都是通过 Message 来驱动的。

看一下 Android 应用程序的 main 方法：

```java
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
    AndroidOs.install();
    CloseGuard.setEnabled(false);
    Environment.initForCurrentUser();
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);
    Process.setArgV0("<pre-initialized>");
    // 初始化Looper
    Looper.prepareMainLooper();
    long startSeq = 0;
    if (args != null) {
        for (int i = args.length - 1; i >= 0; --i) {
            if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                startSeq = Long.parseLong(
                        args[i].substring(PROC_START_SEQ_IDENT.length()));
            }
        }
    }
    // 创建ActivityThread
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    // 启动Looper
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

这里的逻辑主要就是启动了 ActivityThread 和 初始化了 Looper之后没有启动了其他的逻辑，那Activity 是怎么被调用并执行逻辑呢？通过 Handler，Android 是事务驱动型的设计，通过不断地分发事务让整个程序运行起来。AMS 通过 binder 机制和程序联系起来，然后 binder 进程发送消息给 主线程，主线程再执行相应地逻辑。它们地关系可以用下面的图表示：

![](/picture/data-09.image)

**当应用进程被创建的时候，只是创建了主线程的 Looper 和 Handler，以及 Binder 线程等，之后 AMS 通过 binder与应用程序通信，给主线程发送消息，让程序执行创建 Activity等的操作。**这样的设计我们就不用了写死循环去等待用户的输入逻辑，应用程序就可以跑起来且不会结束。之后程序会开启其他的线程来接收用户的触摸输入，然后把这些包装成一个 Message 发送到主线程更新 UI。























