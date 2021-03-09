# Android 全面解析之 Handler 机制：常见问题汇总

### 正文

##### 主线程为什么不能初始化 Looper？

答：因为应用在启动的过程中已经初始化主线程的 Looper 了。

每个 Java 程序都是有一个 main 方法的入口，Android 是基于 Java 的程序也不例外，Android 程序的入口在 ActivityThread 的 main 方法中：

```java
public static void main(String[] args) {
    ...
 // 初始化主线程Looper
    Looper.prepareMainLooper();
    ...
    // 新建一个ActivityThread对象
    ActivityThread thread = new ActivityThread(); //ActivityThread 是一个类，不是线程
    thread.attach(false, startSeq);

    // 获取ActivityThread的Handler，也是他的内部类H
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    ...
    Looper.loop();
 // 如果loop方法结束则抛出异常，程序结束
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

main 方法中先初始化主线程的 Looper，新建 ActivityThread对象，然后再启动 Looper，这样主线程的 Looper在程序启动的时候就跑起来了。我们不需要在去初始化 Looper了。

#### 为什么主线程的 Looper 是一个死循环，却不会导致 ANR？

答：因为当 Looper 处理完所有的消息之后会进入阻塞状态，当有新的消息进来的时候会被唤醒继续执行。

ANR 是 Application not responding。当我发送一个绘制 UI 的消息到主线程执行的时候，经过一定的时间没有绘制，则会导致 ANR 异常，Looper 的死循环，是循环执行各种事务，包括绘制事务。Looper 死循环说明线程没有死亡，如果 Looper 停止循环则线程就结束退出了。Looper 的死循环本身就是保证UI 绘制任务可以被执行的原因之一，同时 UI 绘制有 同步屏障，可以更快速的保障绘制更快执行。

### Handler 如何保证 MessageQueue 并发访问安全

答：循环加锁，配合阻塞唤醒机制

我们可以发现 MessageQueue 其实是“生产者-消费者模型”，Handler 不断的放入消息，Looper 不断的取出，这就涉及到了死锁问题。如果 Looper 拿到锁，但是队列中没有消息，就会一直等待，而 Handler 需要把消息放进去，锁被 Looper 持有却无法入队，这就造成了死锁。Handler 机制的解决方法是 **循环加锁**。在 MessageQueue 的 next 方法中：

```java
Message next() {
   ...
    for (;;) {
  ...
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            ...
        }
    }
}
```

我可以看到他的等待是在锁外的，当队列中没有消息的时候，他会先释放锁，再进行等待，直到被唤醒，这样就不会造成死锁问题了。

那再入队的时候会不会因为队列已经满了然后一边在等待消息一边拿着锁呢？MessageQueue 没有上限，或者他的内存是JVM 给程序分配的内存，如果超出内存会抛出异常，但一般情况下是不会的。

#### Handler 是如何切换线程的？

答：使用不同线程的 Looper 处理消息。

代码的执行过程，并不是代码决定的，而是执行这段代码的环境在哪个线程的决定的，或者说哪个线程的逻辑调用的。每个 Looper 都运行在对应的线程，所以不同的 Looper 调用的 dispatchMessage 方法就运行在所在的线程了。

#### Handler 的阻塞唤醒机制是怎么回事？

答：Handler 的阻塞唤醒机制是基于 Linux 的阻塞唤醒机制

这个机制也是类似于 Handler 机制的模式。在本地创建一个文件描述符，然后需要等待的一方则监听这个文件描述符，唤醒的一方只需要修改这个文件，那么等待的一方就会收到文件从而打破唤醒，和 Looper 监听 MessageQueue，Handler 添加 Message 是比较类似的。

#### 能不能让一个 Message 加急被处理？什么是Handler 同步屏障？

答：可以 。一种使得异步消息可以被更快处理的机制

如果向主线程发动了一个 UI 更新的操作 Message，而此时的消息队列中消息很多，那么这个 Message  的处理就变的十分缓慢，造成界面卡顿，所以通过同步屏障，可以使得 UI 绘制的 Message更快的执行。

什么是同步屏障？这个“屏障”其实是一个 Message，插入在 MessageQueue 的链表头，且其target 是 null。MessageQueue 入队的时候不是判断 target 不能是 null吗？添加屏障是另外一个方法：

```java
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        // 把当前需要执行的Message全部执行
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        // 插入同步屏障
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

可以看到同步屏障就是一个特殊的 Target ，哪里特殊呢？target == null，我们可以看到他 target 属性赋值，那这个 target 有什么用的？看 next 方法：

```java
Message next() {
    ...

    // 阻塞时间
    int nextPollTimeoutMillis = 0;
    for (;;) {
        ...
        // 阻塞对应时间 
        nativePollOnce(ptr, nextPollTimeoutMillis);
  // 对MessageQueue进行加锁，保证线程安全
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            /**
            *  1
            */
            if (msg != null && msg.target == null) {
                // 同步屏障，找到下一个异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 下一个消息还没开始，等待两者的时间差
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获得消息且现在要执行，标记MessageQueue为非阻塞
                    mBlocked = false;
                    /**
              *  2
              */
                    // 一般只有异步消息才会从中间拿走消息，同步消息都是从链表头获取
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 没有消息，进入阻塞状态
                nextPollTimeoutMillis = -1;
            }

            // 当调用Looper.quitSafely()时候执行完所有的消息后就会退出
            if (mQuitting) {
                dispose();
                return null;
            }
            ...
        }
        ...
    }
}
```

这个方法，看一下同步屏障的部分，看注释1 的地方的代码：

```java
if (msg != null && msg.target == null) {
    // 同步屏障，找到下一个异步消息
    do {
        prevMsg = msg;
        msg = msg.next;
    } while (msg != null && !msg.isAsynchronous();
}
```

如果遇到同步屏障，那么会循环遍历整个链表找到标记为异步消息的 Message，即 isAsynchronous 为 true，其他的消息就会直接忽视，那么这样异步消息，就会被提前执行了。

注意，同步屏障不会自动移除，使用完成之后不会自己移除，使用完成之后需要手动进行移除，不然会造成同步消息无法被处理。

有了这个同步屏障，那么唤醒的判断条件需要加一个：**MessageQueue 中有同步屏障且处于阻塞中，此时插入在所有异步消息前插入新的异步消息**。这个也好理解，跟同步消息是一样的。如果把所有的同步消息先忽略，就是插入新的链表头且处于阻塞状态，这个时候就需要被唤醒了。看下：

```java
boolean enqueueMessage(Message msg, long when) {
    ...

    // 对MessageQueue进行加锁
    synchronized (this) {
        ...
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            /**
            * 1
            */
            // 当线程被阻塞，且目前有同步屏障，且入队的消息是异步消息
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                /**
                * 2
                */
                // 如果找到一个异步消息，说明前面有延迟的异步消息需要被处理，不需要被唤醒
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; 
            prev.next = msg;
        }
  
        // 如果需要则唤醒队列
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

如果插入的消息是异步消息，且有同步屏障，同时 MessageQueue 正处于阻塞状态，那么需要被唤醒，**如果插入的异步消息的插入位置不是所有异步消息之前，那么不需要唤醒**。它前面没有异步消息，后面有异步消息，那么久需要他被先执行，when 比较小，所以需要唤醒。前面有异步消息，说明 when 比较大，不需要先执行。

那么如何发送一个异步类型的消息呢？

- 使用异步类型的 Handler 发送的全部 Message 是异步的
- 给Message 标记异步

> 要理解这个方法的含义，我们要先了解一下Handler的同步屏障机制。通常我们使用Handler发消息的时候，都是用的默认的构造方法生成Handler，然后用send方法来发送消息，其实这时候我们发送的都是同步消息，发出去之后就会在消息队列里面排队处理。
>

> 我们都知道，Android系统16ms会刷新一次屏幕，如果主线程的消息过多，在16ms之内没有执行完，必然会造成卡顿或者掉帧。那怎么才能不排队，没有延时的处理呢？这个时候就需要异步消息，在处理异步消息的时候，我们就需要同步屏障，让异步消息不用排队等候处理。可以理解为同步屏障是一堵墙，把同步消息队列拦住，先处理异步消息，等异步消息处理完了，这堵墙就会取消，然后继续处理同步消息。

Handler 有一系列带Boolean 类型的参数的构造器，这个参数就是决定是否是 异步 Handler。

```java
public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    // 这里赋值
    mAsynchronous = async;
}
```

但是异步类型的异步类型的Handler 构造器标记为 hide，我们无法使用，所以我们使用异步消息只有通过给 Message 设置异步标志：

```java
public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
}
```

**BUT！！！**，其实同步屏障对我们平常的使用是没有多大用处的。因为设置同步屏障和创建异步 Handler 的方法都标记为 hide，说明 Google 不想要我们去使用他。

#### 移除同步屏障

我们可以通过 removeSyncBarrier 方法来移除同步屏障。

```java
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        // 找到 target 为 null 且 token 相同的消息
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();
        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
```

这里主要是将同步屏障从 MessageQueue 中移除，一般执行完了异步消息后就会通过该方法将同步屏障移除。

最后若需要唤醒，调用了 nativeWake 方法。

# 阻塞唤醒机制

我们知道不断进行着循环时非常消耗资源的，有时我们的 消息是不需要马上就执行的，而是需要过一段时间，此时如果 Looper 仍然不断地循环是一种资源地浪费。

因此 Handler 设计了这样一种阻塞唤醒机制使得在当下没有需要执行的消息时，将将 Looper 的 loop阻塞，直到下一个任务的执行时间到达或者一些特殊的情况下再将其唤醒，从而避免了上述的资源浪费。

#### epoll

这个阻塞唤醒机制时基于 Linux IO 多路复用机制 epoll 实现的，它可以同时监控多个资源描述符，当某个文件描述符就绪时，会通知对应程序进行读/写 操作。

epoll 主要有三个方法，分别时

epoll_create、epoll_ctl、epoll_wait

##### epoll_create

```java
int epoll_create(int size)
```

其功能主要是创建一个 epoll 句柄并返回，传入的 size 代表监听的描述符的个数。

##### epoll_ctl

```java
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
```

其功能是对 epoll 事件进行注册，会对该 fd 执行对应的 op 操作，参数含义如下：

epfd: epoll 的句柄值（也就是 epoll_create 的返回值）

op：对 fd 执行的操作

- EPOLL_CTL_ADD:注册 fd 到 epfd
- EPOLL_CTL_DEL:从 epfd 中删除 fd
- EPOLL_CTL_MOD:修改已注册的 fd 的监听事件

fd:需要监听的文件描述符

epoll_event：需要监听的事件

epoll_event 是一个结构体，里面的 events 代表了对应文件操作符的操作。而 data 代表了用户可用的数据

其中 events 可取下面几个值：

- EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
- EPOLLOUT：表示对应的文件描述符可以写；
- EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外部数据来）；
- EPOLLERR：表示对应的文件描述符发生错误；
- EPOLLHUP：表示对应的文件描述符被挂断；
- EPOLLET：将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
- EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

##### epoll_wait

```java
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

其功能是等待事件的上报，参数含义如下：

- epfd:epoll 的句柄值
- events:从内核中得到的事件集合
- maxevents：events 数量，不能超过 create 时的 size
- timeout：超时时间

当调用了该方法后，会进入阻塞状态，等待 epfd 上 IO 事件，若 epfd 监听的某个文件描述符发生前面指定的  event 时，就会进行回调，从而使得 epoll 被唤醒并返回需要处理的事件个数，若超过了 设定的超时时间，同样也会被唤醒并返回 0 避免一直阻塞。

而 Handler 的阻塞唤醒机制就是基于上面的 epoll 的阻塞特性。

#### native 初始化

在 Java 中 的 MessageQueue 创建时会调用到 nativeInit 方法，在 native 层会创建 NativeMessageQueue 并返回其地址，之后都是通过这个地址来与该 NativeMessageQueue 进行通信（也就是 MessageQueue 中的 mPtr),而在 NativeMessageQueue 创建时又会创建 Native 层下的 Looper，我们看到 Native 下 的 Looper 的构造函数。

```c++
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK); //构造唤醒事件的fd
    AutoMutex _l(mLock);
    rebuildEpollLocked(); 
}
```



可以看到，它调用了 rebuildEpollLocked 方法对 epoll 进行初始化，看下其实现：

```java
void Looper::rebuildEpollLocked() {
    if (mEpollFd >= 0) {
        close(mEpollFd); 
    }
    mEpollFd = epoll_create(EPOLL_SIZE_HINT); 
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event));
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeEventFd;
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);

    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
    }
}
```

可以看到，这里首先关闭了旧的 epoll 描述符，之后又调用了 epol_create 创建了新的 epoll 描述符，然后进行了一些初始化后，讲 mWakeEventFd 及 mRequests 中的 fd 都注册到了 epoll 的描述符中，注册的事件都是 EPOLLIN。

这意味着当这些文件描述符其中一个 发生了 IO之后，就会通知 epoll_wait 使其唤醒，那么我们猜测 Handler 的阻塞唤醒机制就是通过 epoll_wait 实现的。

同时可以发现，Native 层也是存在 MessageQueue 和 Looper 的，也就是说 native 层实际上也有一套消息机制的。

#### Native 阻塞实现

我们看看阻塞，他的实现就在我们看到的 MessageQueue:next 方法中，当发现要返回的消息将来才会执行，则会计算出当下距离其要执行的时间还差多少毫秒，并调用 nativePollOnce 方法将返回的过程阻塞到指定的时间。

NativePollOnce 显然是一个 native 方法，最后调用了到了 Looper 这个 native 层类的 pollOnce 方法。

```java
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }
        if (result != 0) {
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }
        result = pollInner(timeoutMillis);
    }
}
```

主要是一些对 Native 层消息机制的处理，我们暂时不关心，最后调用到了 pollInner 方法：

```java
int Looper::pollInner(int timeoutMillis) {
    // ...
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;
    mPolling = true; 
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    // 1
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // ...
    return result;
}
```

可以发现，这里 1 处调用了 epoll_wait 方法，并传入我们之前在nativePollOnce 方法传入的当前时间距下个任务执行时间的差值。

这就是我们的阻塞功能的核心实现了，调用该方法之后，会一直阻塞，直到到达我们设定的时间或之前我们在 epoll 中的 fd 注册的几个 fd 发生了 IO。这里可以猜到，nativeWake 方法就是通过对注册的 nWakeEventFd 进行操作从而实现的唤醒。

#### Native 唤醒

nativeWake 方法最后通过 NativeMessageQueue 的 wake 方法调用到了 Native 下 Looper 的 wake 方法。

```java
void Looper::wake() {
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
```

这里其实就是调用了 write 方法，对 mWakeEventfd 中写入了 1，从而使得监听该 fd 的 pollOnce 方法被唤醒，从而使得 Java 中的 next 方法继续执行。

那我们再回去看看，在什么情况下，Java 层会调用 natvieWake 方法进行唤醒呢？

**MessageQueue 类中调用 nativeWake 方法主要有下列几个时机：**

- **调用 MessageQueue 的 quit 方法进行退出时，会进行唤醒（让其退出）**
- **消息入队时，若插入的消息在链表最前端（最早将执行）或者有同步屏障时插入的是最前端的异步消息（最早被执行的异步消息）**
- **移除同步屏障时，若消息列表为空或者同步屏障后面不是异步消息时**

可以发现，主要是在可能不再需要阻塞的情况下进行唤醒。（比如加入了一个更早的任务，那继续阻塞显然会影响这个任务的执行）









### 什么是 IdleHandler?

答：当 MessageQueue 为空或者目前没有需要执行的 Message 时会回调的接口对象。

IdleHandler 看起来好像是一个 Handler，但是他其实只是只有一个单方法的借口，也称为函数型接口：

```
public static interface IdleHandler {
    boolean queueIdle();
}
```

在 MessageQueue 中有一个 List 存储了 Idlehandler  对象，当 MessageQueue 没有需要被执行的 Messagee 的时候会遍历回调所有的 Idlehandler。所以 IdleHandler 主要用于在消息队列为空闲的时候处理一些**轻量级** 的工作。

IdleHandler 的调用是在 next 方法中：

```java
Message next() {
    // 如果looper已经退出了，这里就返回null
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    // IdleHandler的数量
    int pendingIdleHandlerCount = -1; 
    // 阻塞时间
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        // 阻塞对应时间 
        nativePollOnce(ptr, nextPollTimeoutMillis);
  // 对MessageQueue进行加锁，保证线程安全
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // 同步屏障，找到下一个异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 下一个消息还没开始，等待两者的时间差
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获得消息且现在要执行，标记MessageQueue为非阻塞
                    mBlocked = false;
                    // 一般只有异步消息才会从中间拿走消息，同步消息都是从链表头获取
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 没有消息，进入阻塞状态
                nextPollTimeoutMillis = -1;
            }

            // 当调用Looper.quitSafely()时候执行完所有的消息后就会退出
            if (mQuitting) {
                dispose();
                return null;
            }

            // 当队列中的消息用完了或者都在等待时间延迟执行同时给pendingIdleHandlerCount<0
            // 给pendingIdleHandlerCount赋值MessageQueue中IdleHandler的数量
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            // 没有需要执行的IdleHanlder直接continue
            if (pendingIdleHandlerCount <= 0) {
                // 执行IdleHandler，标记MessageQueue进入阻塞状态
                mBlocked = true;
                continue;
            }

            // 把List转化成数组类型
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // 执行IdleHandler
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // 释放IdleHandler的引用
            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            // 如果返回false，则把IdleHanlder移除
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // 最后设置pendingIdleHandlerCount为0，防止再执行一次
        pendingIdleHandlerCount = 0;

        // 当在执行IdleHandler的时候，可能有新的消息已经进来了
        // 所以这个时候不能阻塞，要回去循环一次看一下
        nextPollTimeoutMillis = 0;
    }
}
```

- 当调用 next 方法的时候，会给 pendingIdleHandlerCount 赋值为 -1
- 如果队列中没有消息需要处理，就会判断是否为 <0,若是则把存储的 idlehandler 长度给它
- 把list 中所有的 Idlehandler 放到数组中。这一步是为了不让在执行 IdleHandler 的时候 List 被插入到新的 IdleHandler，造成逻辑混乱
- 然后遍历整个数组执行所有的 IdleHandler
- 最后给 他赋值为0，然后再回去看一下这个期间有没有新的消息插入。因为他为0不是 -1，所以Idlehandler 只会在空闲的时候执行一次。
- 同时注意，如果 IdleHandler 返回了 false，那么执行一次之后就被丢弃了。













```

```









