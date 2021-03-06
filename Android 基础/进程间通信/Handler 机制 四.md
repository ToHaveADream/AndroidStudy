# Android 全面解析之 Handler 机制（四）：内部关键类

### 前言

本文内容主要介绍Handler 的内部关键类：Handler、以及 HandlerThread。

### 正文

## Handler

#### 概述

我们整个消息机制称为Handler 机制就可以知道 Handler 的使用频率之高，一般情况下我们的使用也是围绕着 Handler 来展开。**Handler 是做为整个消息机制中的发起者和处理者**，消息在不同的线程通过 Handler 发送到目标线程的 MessageQueue 中，然后目标线程的 Looper 再调用 Handler 的 dispatchMessage 方法来处理消息。

#### 创建 Handler

一般情况下我们使用的 Handler 有两种方式：继承 Handler 并重写 handleMessage 方法；直接创建 Handler 对象并传入 callBack。

需要注意一点的是：创建 Handler 必须显示指明 Looper 参数，并不能直接使用无参构造函数，如：

```java
Handler handler = new Handler(); //1
Handler handler = new Handler(Looper.myLooper())//2
```

1是错的，2是错的。避免在  Handler 创建过程中 Looper 已经退出的情况。

#### 发送消息

Handler 发送消息有两种系列方法：postXXX 和 sendXXX。如下：

```java
public final boolean post(@NonNull Runnable r);
public final boolean postDelayed(@NonNull Runnable r, long delayMillis);
public final boolean postAtTime(@NonNull Runnable r, long uptimeMillis);
public final boolean postAtFrontOfQueue(@NonNull Runnable r);

public final boolean sendMessage(@NonNull Message msg);
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis);
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis);
public final boolean sendMessageAtFrontOfQueue(@NonNull Message msg)
```

这里我只列出了比较常用的两类方法。除了插在队列头的两个方法，其他方法最终都调用到了 sendMessageAtTime。我们从 post 方法跟源码分析一下:

```java
public final boolean post(@NonNull Runnable r) {
    return  sendMessageDelayed(getPostMessage(r), 0);
}
```

post 方法把 runnable 对象封装成一个 Message，再调用 sendMessageDelayed 方法，我们看看他是如何封装的：

```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

可以看到逻辑很简单，把runnable 对象直接赋值给 callback 属性。接下来回去继续看 sendMessageDelayed。

```java
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

sendMessageDelayed 把小于 0 的延迟时间改成0，然后调用 sendMessageAtTime。这个方法主要是判断 MessageQueue 是否已经初始化了，然后再调用 enqueueMessage 方法进行入队操作。

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    // 这里把target设置成自己
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();
 // 异步handler设置标志位true，后面会讲到同步屏障
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    // 最后调用MessageQueue的方法入队
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

可以看到 Handler 的入队操作也是很简单，把 Message 的 target 设置成本身，这样这个 Message 最后就是由自己来处理。最后调用 MessageQueue 的入队方法来入队。

#### 处理消息

上面讲Looper 处理消息的时候，最后就是调用 Handler 的 dispatchMessage 方法来处理。我们来看一下这个方法：

```java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}
```

逻辑也不是很复杂。首先判断 Message 是否有 callback，有的话就直接执行 callBack 的逻辑，这个 callBack 就是我们调用 Handler 的 post 系列传进去的 runnable 对象；否则判断 Handler 是否有 callBack，有的话就执行他的方法。如果返回false 直接调用 Handler 本身的 handleMessage 方法。这个过程用下面的图表示一下：

![](/picture/data-06.image)

#### 内存泄漏问题

当我们使用继承Handler方法来使用Handler的时候，要注意使用静态内部类，而不要用非静态内部类。因为非静态内部类会持有外部类的引用，而从上面的分析我们知道Message在被入队之后他的target属性是指向了Handler，如果这个Message是一个延迟的消息，那么这一条引用链的对象就迟迟无法被释放，造成内存泄露。

一般这种泄露现象在于：我们在Activity中发送了一个延迟消息，然后退出了activity，但是由于无法释放，这样activity就无法被回收，造成内存泄露

## HandlerThread

#### 概述

有时候我们需要开辟一个线程来执行一些耗时的任务。一般情况下可以通过新建一个 Thread，然后在它 的 run  方法里初始化该线程的 Looper，这样就可以用他的 Looper 来切换线程处理消息了。如下所示：

```java
val thread = object : Thread(){
    lateinit var mHandler: Handler
    override fun run() {
        super.run()
        Looper.prepare()
        mHandler = Handler(Looper.myLooper()!!)
        Looper.loop()
    }
}
thread.start()
thread.mHandler.sendMessage(Message.obtain())
```

但是运行一下，崩溃了：

![](/picture/data-07.image)

Handler 还未初始化。线程的启动需要一定的时间，如果在 Thread 的 run  方法还没被调用之前获取Handler，就会出现 Handler 未初始化的问题。那简单，等待一下就可以了，上代码：

```java
val thread = object : Thread(){
    lateinit var mHandler: Handler
    override fun run() {
        super.run()
        Looper.prepare()
        mHandler = Handler(Looper.myLooper()!!)
        Looper.loop()
    }
}
thread.start()
Thread(){
    Thread.sleep(10000)
    thread.mHandler.sendMessage(Message.obtain())
}.start()
```

执行以下，没有报错了。但是这样写代码太过臃肿了，还要开启一个线程来延迟处理。这里有一个方法：HandlerThread。

HandlerThread 本身是一个 thread，继承自 Thread，他的代码并不复杂：

```java
public class HandlerThread extends Thread {
    // 依次是：线程优先级、线程id、线程looper、以及内部handler
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    private @Nullable Handler mHandler;

    // 两个构造器。name是线程名字，priority是线程优先级
    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    // 在Looper开始运行前的方法
    protected void onLooperPrepared() {
    }

    // 初始化Looper
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            // 通知初始化完成
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
    // 获取当前线程的Looper
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        // 如果尚未初始化则会一直阻塞知道初始化完成
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    // 利用Object对象的wait方法
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    // 获取handler，该方法被标记为hide，用户无法获取
    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }

    // 两种不同类型的退出，前面讲过不再赘述
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    // 获取线程id
    public int getThreadId() {
        return mTid;
    }
}
```

整个类的代码不是很多，重点在 run() 和 getLooper() 方法。首先看到 getLooper 方法：

```java
public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }
    // 如果尚未初始化则会一直阻塞知道初始化完成
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
                // 利用Object对象的wait方法
                wait();
            } catch (InterruptedException e) {
            }
        }
    }
    return mLooper;
}
```

和我们前面写得不同，他又一个 wait()，这个是 Java 中 Object类提供的一个方法，等到 Looper初始化完成之后就会唤醒他，就可以顺利返回了，不会造成 Looper 尚未初始化完成的情况，然后再看到 run  方法：

```java
// 初始化Looper
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        // 通知初始化完成
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}

```

常规的 Looper 初始化，完成之后调用了 notifyAll() 方法进行了唤醒，对应了上面的 getlooper 方法。

#### HandlerThread 的使用

HandlerThread 的使用范围比较有限，开个子线程不断接收消息处理耗时任务，所以他的使用方法也是比较固定：

```java
HandlerThread ht = new HandlerThread("handler");
ht.start();
Handler handler = new Hander(ht.getLooper());
handler.sendMessage(msg);
```

获取到他的 Looper，外部自定义 Handler 来使用即可。

## 最后

Handler，MessageQueue，Looper 三者共同构成了 Android 消息机制，各司其职。其中 Handler 主要负责发送消息和处理消息，MessageQueue 主要负责消息的排序以及在没有需要处理的消息阻塞代码，Looper 负责从 MessageQueue 中取出消息给 Handler 处理，同时达到切换线程的目的。















































