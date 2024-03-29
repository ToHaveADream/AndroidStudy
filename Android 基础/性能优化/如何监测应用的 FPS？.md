## 面试官：如何检测应用的 FPS？

### 什么是 FPS？

　即使你不知道 FPS，但是你一定知道，`[在 Android 中，每一帧的绘制时间不要超过 16.67 ms]`。那么，这个 16.67 ms 是怎么来的呢？就是由 FPS 决定的。

`[FPS ,Frame Per Second]`每秒显示的帧数，也叫帧率。Android 设备的 FPS 一般是 60，也就是每秒要刷新 60 帧，所以留给每一帧的绘制时间最多只有 `[1000 /60  = 16.167 ms]`。一旦某一帧的绘制时间超过了限制，就会发生`[掉帧]`。用户在连续两帧会看到同样的画面。

监测 FPS 在一定程度上可以反映应用的卡顿情况，原理也很简单，但前提是你对屏幕刷新机制和绘制流程很熟悉。所以我不会直接进入主题，让我们先从 `View.invalidate()`说起。

### 从 View.invalidate 说起

要探究屏幕刷新机制和 View 绘制流程，`View.invalidate`无疑是一个好选择，它会发起一次绘制流程。

```java
> View.java

public void invalidate() {
    invalidate(true);
}

public void invalidate(boolean invalidateCache) {
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}

void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
    boolean fullInvalidate) {
    ......
    final AttachInfo ai = mAttachInfo;
    final ViewParent p = mParent;
    if (p != null && ai != null && l < r && t < b) {
        final Rect damage = ai.mTmpInvalRect;
        damage.set(l, t, r, b);
	// 调用 ViewGroup.invalidateChild()
        p.invalidateChild(this, damage);
    }
    ......
}
```

这里调用到 `ViewGroup.invalidateChild()`。

```java
> ViewGroup.java

public final void invalidateChild(View child, final Rect dirty) {
    final AttachInfo attachInfo = mAttachInfo;
    ......
    ViewParent parent = this;
    if (attachInfo != null) {
        ......
        do {
            View view = null;
            if (parent instanceof View) {
                view = (View) parent;
            }
            ......
            parent = parent.invalidateChildInParent(location, dirty);
            ......
        } while (parent != null);
    }
}
```

这里有一个递归，不停的调用父 View 的 `invalidateChildInParent`方法，直到最顶层父 View 为止。这很好理解，仅靠 View 自身是无法绘制自己的，必须依靠最顶层的父 View 才可以测量，布局，绘制整个 View 树。但是最顶层的父 View 是谁呢？是 `setContentView` 传入的布局文件吗？不是，它解析之后被塞进了 `DecorView` 中。是 `DecorView` 吗？也不是，它也有父亲。

DecorView 的 parent 是谁呢？这就来到了 ActivityThread.handleResume 方法。

```java
> ActivityThread.java

public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward, String reason) {
    ......
    // 1. 回调 onResume()
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ......
    View decor = r.window.getDecorView();
    decor.setVisibility(View.INVISIBLE);
    ViewManager wm = a.getWindowManager();
    // 2. 添加 decorView 到 WindowManager
    wm.addView(decor, l);
    ......
}
```

第二部中实际调用的是 `WindowManagerImpl.addView` 方法。`WindowManagerImpl`中又调用了 WindowManagerGlobal.addView 方法。

```java
> WindowManagerGlobal.java

// 参数 view 就是 DecorView
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ......
    ViewRootImpl root;
    // 1. 初始化 ViewRootImpl
    root = new ViewRootImpl(view.getContext(), display);

    mViews.add(view);
    mRoots.add(root);
    // 2. 重点在这
    root.setView(view, wparams, panelParentView);
    ......
}
```

跟进 `ViewRootImpl.setView` 方法。

```java
> ViewRootImpl.java

// 参数 view 就是 DecorView
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;

            // 1. 发起首次绘制
            requestLayout();

            // 2. Binder 调用 Session.addToDisplay()，将 window 添加到屏幕
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);

            // 3. 重点在这，注意 view 是 DecorView，this 是 ViewRootImpl 本身
            view.assignParent(this);
        }
    }
}
```

跟进`View.assignParent()`方法。

```java
> View.java

// 参数 parent 是 ViewRootImpl
void assignParent(ViewParent parent) {
    if (mParent == null) {
        mParent = parent;
    } else if (parent == null) {
        mParent = null;
    } else {
        throw new RuntimeException("view " + this + " being added, but"
                + " it already has a parent");
    }
}
```

还记得我们跟了这么久在干嘛吗？为了探究 View 的刷新流程，我们跟着 `View.invalidate`方法一路追到 `ViewGroup.invalidateChild()`，其中递归调用 Parent 的 invalidateChildInParent() 方法，所以我们在 `给 DecorView 找爸爸`。现在很清晰了，`DecorView 的爸爸就是 ViewRootImpl`,所以最终调用的就是 `ViewRootImpl.invalidateChildInParent()`方法。

```java
> ViewRootImpl.java

public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    // 1. 线程检查
    checkThread();

    if (dirty == null) {
        // 2. 调用 scheduleTraversals()
        invalidate();
        return null;
    } else if (dirty.isEmpty() && !mIsAnimating) {
        return null;
    }
    ......
    // 3. 调用 scheduleTraversals()
    invalidateRectOnScreen(dirty);

    return null;
}
```

无论是注释 2 处的 `invalidate()` 还是注释 3 处的 `invalidateRectOnScreen()` ，最终都会调用到 `scheduleTraversals()` 方法。

`scheduleTraversals()` 在 View 绘制流程中是个极其重要的方法，我不得不单独开一节来聊聊它。

## 承上启下的 “编舞者”

上一节中，我们从 `View.invalidate()` 方法开始追踪，一直跟到 `ViewRootImpl.scheduleTraversals()` 方法。

```java
> ViewRootImpl.java

void scheduleTraversals() {
    // 1. 防止重复调用
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
		// 2. 发送同步屏障，保证优先处理异步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
		// 3. 最终会执行 mTraversalRunnable 这个任务
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ......
    }
}
```

1. `mTraversalScheduled`是个布尔值，防止重复调用，在一次 vsync 信号期间多次调用是没有意义的。
2. 利用 Handler 的同步屏障机制，优先处理异步消息
3. Choreographer 登场

到这里，鼎鼎大名的 `[编舞者 -- Choreographer [[ˌkɔːriˈɑːɡrəfər]]`就该出场了（为了避免面试中出现不会读单词的尴尬，掌握以下发音还是很有必要的。

通过 `mChoreographer` 发送了一个任务 `mTraversalRunnable`,最终会在某个时刻被执行。在看源码之前，先抛出几个问题：

1.  `mChoreographer` 什么时候被初始化的
2.  `mTraversalRunnable`是什么？
3.  `mChoreographer` 是如何发送任务以及任务是如何被调度执行的？

围绕这三个问题，我们再回到源码中。

先看第一个问题，这就需要回到上一节介绍过的 `WindowManagerGlobal.addView()`方法。

```java
> WindowManagerGlobal.java

// 参数 view 就是 DecorView
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ......
    ViewRootImpl root;
    // 1. 初始化 ViewRootImpl
    root = new ViewRootImpl(view.getContext(), display);

    mViews.add(view);
    mRoots.add(root);
    
    root.setView(view, wparams, panelParentView);
    ......
}
```

注释1里面新建了 ViewRootImpl 对象，跟进 ViewRootImpl 的构造函数。

```java
> ViewRootImpl.java

public ViewRootImpl(Context context, Display display) {
    mContext = context;
    // 1. IWindowSession 代理对象，与 WMS 进行 Binder 通信
    mWindowSession = WindowManagerGlobal.getWindowSession();
    ......
    mThread = Thread.currentThread();
    ......
    // IWindow Binder 对象
    mWindow = new W(this);
    ......
    // 2. 初始化 mAttachInfo
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
                context);
    ......
    // 3. 初始化 Choreographer，通过 Threadlocal 存储
    mChoreographer = Choreographer.getInstance();
    ......
}
```

在 `ViewRootImpl`的构造函数中，注释 3 处初始化了 `mChoreographer`，调用的是`Choreographer.getInstance()`方法。

```java
> Choreographer.java

public static Choreographer getInstance() {
    return sThreadInstance.get();
}
```

`sThreadInstance`是一个 `ThreadLocal<Choreographer>`对象。

```java
> Choreographer.java

private static final ThreadLocal<Choreographer> sThreadInstance =
        new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
        Looper looper = Looper.myLooper();
        if (looper == null) {
            throw new IllegalStateException("The current thread must have a looper!");
        }
        // 新建 Choreographer 对象
        Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
        if (looper == Looper.getMainLooper()) {
            mMainInstance = choreographer;
        }
        return choreographer;
    }
};
```

所以 mChoreographer 保存在 ThreadLocal 中的线程私有对象。它的构造函数中需要传入当前线程的（这里是主线程的）Looper 对象。

这里再插一个题外话，`[主线程 Looper 是在什么时候创建的？]`回顾一下应用进程的创建流程：

1. 调用 Process.start 创建应用进程
2. `ZygoteProcess` 负责和 `Zygote` 进程建立 `socket` 连接，并将创建进程需要的参数发送给 `Zygote` 的 socket 服务端
3. Zygote 服务端接收到参数之后调用 `ZygoteConnection.processOneCommand()` 处理参数，并 fork 进程
4. 最后通过 `findStaticMain` 找到 `ActivityThread `类的 `main `方法并执行，子进程就启动了。

ActivityThread 并不是一个线程，但它是运行在主线程上的，主线程 Looper 就是在它的 main 方法中执行的。

```java
> ActivityThread.java

public static void main(String[] args) {
    ......
    // 创建主线程 Looper
    Looper.prepareMainLooper(); 
    ......
    // 创建 ActivityThread ，并 attach(false)
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
    ......
    // 开启主线程消息循环
    Looper.loop();
}
```

Looper 也是存储在 ThreadLocal 中的。

再回到 Choreographer，我们来看一下它的构造函数。

```java
> Choreographer.java

private Choreographer(Looper looper, int vsyncSource) {
    mLooper = looper;
    // 处理事件
    mHandler = new FrameHandler(looper);
    // USE_VSYNC 在 Android 4.1 之后默认为 true，
    // FrameDisplayEventReceiver 是个 vsync 事件接收器 
    mDisplayEventReceiver = USE_VSYNC
            ? new FrameDisplayEventReceiver(looper, vsyncSource)
            : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;

    // 一帧的时间，60pfs 的话就是 16.7ms
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
    // 回调队列
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
```

这里出现了几个新面孔，`Framehandler`、`FrameDisplayEventReceiver`、`CallbackQueue`，这里暂且混一个脸熟:smile:。

介绍完 Choreographer 是如何初始化的，再回到 Choreographer 发送任务那块。

```java
mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
```

我们看看 `mTraversalRunnable`是什么东西？

```java
> ViewRootImpl.java

final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
    
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

这里是一个 Runnable 对象，run 方法中会执行 `doTraversal()`方法。

```java
> ViewRootImpl.java

void doTraversal() {
    if (mTraversalScheduled) {
        // 1. mTraversalScheduled 置为 false
        mTraversalScheduled = false;
	// 2. 移除同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

	// 3. 开始布局，测量，绘制流程
        performTraversals();
        ......
    }
```

再对比一下最开始发起绘制的 `scheduleTraversals`方法。

```java
> ViewRootImpl.java

void scheduleTraversals() {
    // 1. mTraversalScheduled 置为 true，防止重复调用
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
	// 2. 发送同步屏障，保证优先处理异步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
	// 3. 最终会执行 mTraversalRunnable 这个任务
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ......
    }
}
```

`mTraversalRunnable` `最终会调用 `performTraversals`方法，来完成整个 View 的测量，布局和绘制流程。

分析到这里，就差一步了，`[mTraversalRunnable 是如何被调度执行的？]`。我们再回到 `Choreographer.postCallback()`方法。

```java
> Choreographer.java

public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}

public void postCallbackDelayed(int callbackType,
        Runnable action, Object token, long delayMillis) {
    ......
    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}

// 传入的参数依次是 Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null，0
private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    ......
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        // 1. 将 mTraversalRunnable 塞入队列
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) { // 立即执行
            // 2. 由于 delayMillis 是 0，所以会执行到这里
            scheduleFrameLocked(now);
        } else { // 延迟执行
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

首先根据 callbackType(这里是 CALLBACK_TRAVERSAL) 将稍后要执行的 mTraversalRunnable 放入相应队列中，其中的具体逻辑就不看了。

然后由于 delayMillis 是 0，所以 dueTime 和   now 是相等的，所以直接执行 `scheduleFrameLocked(now)` 方法。如果 delayMillis 不为 0 的话，会通过 FrameHandler 发送一个延时消息，最后执行的仍然是 `scheduleFrameLocked(now)` 方法。

```
> Choreographer.java

private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) { // Android 4.1 之后 USE_VSYNCUSE_VSYNC 默认为 true
               
            // 如果是当前线程，直接申请 vsync，否则通过 handler 通信
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                // 发送异步消息
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else { // 未开启 vsync，4.1 之后默认开启
            final long nextFrameTime = Math.max(
                    mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}

```

开头到现在，已经好几次提到 `VSYNC`,官方也有一个视频介绍 [Android Performance Patterns: Understanding VSYNC](https://www.youtube.com/watch?v=1iaHxmfZGGc&list=UU_x5XG1OV2P6uZZ5FSM9Ttw&index=1963)，大家可以看一看。简而言之，VSYNC 是为了解决屏幕刷新率和 GPU 帧率不一致导致的“屏幕撕裂”问题。VSYNC 在 PC 端是很久依赖就存在的技术，但在 4.1 之前，Google 才将其引入到 Android 显示系统中，已解决饱受诟病的 UI 显示不流畅问题。

再简单一点，可以把 VSYNC 看成一个由硬件发出的定时信号，通过 Choreographer 监听这个信号。每当信号来临时，统一开始绘制工作。这就是 `scheduleVsyncLocked()`方法的工作内容。

```java
> Choreographer.java

private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}
```

`mDisplayEventReceiver`是`FrameDisplayEventReceiver` 对象，但它并没有 `scheduleVsync() `方法，而是直接调用父类的方法。`FrameDisplayEventReceiver` 的父类是 `DisplayEventReceiver`。

```java
> DisplayEventReceiver.java

public abstract class DisplayEventReceiver {

    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
	        // 注册监听 vsync 信号，会回调 dispatchVsync() 方法
            nativeScheduleVsync(mReceiverPtr);
        }
    }

    // 有 vsync 信号时，由 native 调用此方法
    private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
        // timestampNanos 是 vsync 回调的时间
        onVsync(timestampNanos, builtInDisplayId, frame);
    }

    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
    }
}
```

在 `scheduleVsync() ` 方法中会通过 `nativeScheduleVsync`方法注册下一次 vsync 信号的监听，从方法名也能看出来，下面会进入 native 调用，

注册监听之后，当下次 vsync 信号来临时，会通过 jni 回调 java 层的 `dispatchVsync`方法，其中又调用了 `onVsync`方法。父类  `DisplayEventReceiver`的 `onVsync`方法是个空实现，我们再回到子类`FrameDisplayEventReceiver`，它是 Choreographer 的内部类。

```java
> Choreographer.java

private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {
    private long mTimestampNanos;
    private int mFrame;

    // vsync 信号监听回调
    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        ......
        long now = System.nanoTime();
        // // timestampNanos 是 vsync 回调的时间，不能比 now 大
        if (timestampNanos > now) {
            Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                    + " ms in the future!  Check that graphics HAL is generating vsync "
                    + "timestamps using the correct timebase.");
            timestampNanos = now;
        }
        ......
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        // 这里传入的是 this，会回调本身的 run() 方法
        Message msg = Message.obtain(mHandler, this);
        // 这是一个异步消息，保证优先执行
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        doFrame(mTimestampNanos, mFrame);
    }
}
```

在 `onSync`回调中，向主线程发送了一个异步消息。注意 `sendMessageAtTime`方法参数中的时间是 `timestampNanos / TimeUtils.NANOS_PER_MS`。`timestampNanos`是 vsync信号的时间戳，单位是纳秒，所以这里做一个除法转换为毫秒。代码执行到这里的时候 vsync 信号已经发生，所以 timestampNanos 是比当前时间小的。这样这个消息塞进 MessageQueue 的时候就可以直接塞到前面了。另外 callback 是 this，所以当消息被执行时，调用的是自己的 run 方法，run 方法中调用的是 `doFrame`方法。

```java
> Choreographer.java

void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        if (!mFrameScheduled) {
            return; // no work to do
        }
        ......

        long intendedFrameTimeNanos = frameTimeNanos;
        startNanos = System.nanoTime();
		// 计算超时时间
		// frameTimeNanos 是 vsync 信号回调的时间，startNanos 是当前时间戳
		// 相减得到主线程的耗时时间
        final long jitterNanos = startNanos - frameTimeNanos;
		// mFrameIntervalNanos 是一帧的时间
		if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
			// 掉帧超过 30 帧，打印 log
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            frameTimeNanos = startNanos - lastFrameOffset;
        }
        ......
    }

    try {
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

		// doCallBacks() 开始执行回调
        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
    }
    ......
}
```

在 `choreographer.postCallback` 方法中将 `mTraversalRunnable`塞进了 `mCallbackQueue[]`数组中，下面的 `doCallBacks()`方法中就要把它取出来执行了。

```java
> Choreographer.java

    void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            final long now = System.nanoTime();
			// 根据 callbackType 找到对应的 CallbackRecord 对象
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;
            ......
        }
        try {
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                // 执行 callBack
                c.run(frameTimeNanos);
            }
        } finally {
           ......
        }
    }
```

根据 `callbackType`找到对应的 `mCallbackQueues`，然后执行，具体流程就不深入分析了。`callbackType`共有四个类型。

```java
> Choreographer.CallbackRecord

public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }
```

到此为止，`mTraversalRunnable`得以被执行，`view.invalidate()`的整个流程就走通了。总结一下：

- 从 `View.invalidate()` 开始，最后会递归调用 `parent.invalidateChildInParent()` 方法。这里最顶层的 parent 是 `ViewRootImpl` 。ViewRootImpl 是 DecorView 的 parent，这个赋值调用链是这样的 `ActivityThread.handleResumeActivity -> WindowManagerImpl.addView() -> WindowManagerGlobal.addView() -> ViewRootImpl.setView() -> View.assignParent()` 。

- ViewRootImpl.invalidateChildInParent()` 最终调用到 `scheduleTraversals()` 方法，其中建立同步屏障之后，通过 `Choreographer.postCallback()` 方法提交了任务 `mTraversalRunnable`，这个任务就是负责 View 的测量，布局，绘制。

- `Choreographer.postCallback()` 方法通过 `DisplayEventReceiver.nativeScheduleVsync()` 方法向系统底层注册了下一次 vsync 信号的监听。当下一次 vsync 来临时，系统会回调其 `dispatchVsync()` 方法，最终回调 `FrameDisplayEventReceiver.onVsync()` 方法。

- `FrameDisplayEventReceiver.onVsync()` 方法中取出之前提交的 `mTraversalRunnable` 并执行。这样就完成了整个绘制流程。

## 如何检测应用的 FPS？

检测应用的 FPS 很简单。每次 vsync 信号回调中，都会把四种类型的 `mCallbackQueue`队列中的回调任务。而`Choreographer`又对外提供了提交回调任务的方法，这个方法就是  `Choreographer.getInstance().postFrameCallback()`；看一下：

```java
> Choreographer.java

public void postFrameCallback(FrameCallback callback) {
    postFrameCallbackDelayed(callback, 0);
}

public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    ......
    // 这里的类型是 CALLBACK_ANIMATION
    postCallbackDelayedInternal(CALLBACK_ANIMATION,
            callback, FRAME_CALLBACK_TOKEN, delayMillis);
}
```

和 View.invalidate()流程中调用的 `Choreographer.postCallback()`基本一致，仅仅只是 callback 类型不一致，这里是 `CALLBACK_ANIMATION`。

这里给出代码：

```java
object FpsMonitor {

    private const val FPS_INTERVAL_TIME = 1000L
    private var count = 0
    private var isFpsOpen = false
    private val fpsRunnable by lazy { FpsRunnable() }
    private val mainHandler by lazy { Handler(Looper.getMainLooper()) }
    private val listeners = arrayListOf<(Int) -> Unit>()

    fun startMonitor(listener: (Int) -> Unit) {
        // 防止重复开启
        if (!isFpsOpen) {
            isFpsOpen = true
            listeners.add(listener)
            mainHandler.postDelayed(fpsRunnable, FPS_INTERVAL_TIME)
            Choreographer.getInstance().postFrameCallback(fpsRunnable)
        }
    }

    fun stopMonitor() {
        count = 0
        mainHandler.removeCallbacks(fpsRunnable)
        Choreographer.getInstance().removeFrameCallback(fpsRunnable)
        isFpsOpen = false
    }

    class FpsRunnable : Choreographer.FrameCallback, Runnable {
        override fun doFrame(frameTimeNanos: Long) {
            count++
            Choreographer.getInstance().postFrameCallback(this)
        }

        override fun run() {
            listeners.forEach { it.invoke(count) }
            count = 0
            mainHandler.postDelayed(this, FPS_INTERVAL_TIME)
        }
    }
}
```

大致逻辑是这样的 ：

- 声明变量 `count` 用于统计回调次数
- 通过 `Choreographer.getInstance().postFrameCallback(fpsRunnable)` 注册监听下一次 vsync信号，提交任务，任务回调只做两件事，一是 `count++`，二是继续注册监听下一次 vsync 信号 。
- 通过 Handler 做个定时任务，每隔一秒统计 `count` 值并清空。



## 最后

目前也有很多开源项目实现了 FPS 监测或者流畅度检测的功能。

滴滴开源的 [DoraemonKit](https://github.com/didi/DoraemonKit/) 的做法和上面介绍的是一致的（没错，我就是仿照它写的），可以看一下 `PerformanceDataManager.startMonitorFrameInfo()` 方法的实现。

腾讯开源的 [Matrix](https://github.com/Tencent/matrix#matrix_cn) 虽然也是在 Choreographer 上动手脚，但做的更加彻底，它可以监听到 **CALLBACK_INPUT**、**CALLBACK_ANIMATION**、**CALLBACK_TRAVERSAL**  三种事件各自精确的耗时，重点关注 `LooperMonitor` 和 `UIThreadMonitor` 这两个类。具体原理这里就不再分析了，可以给大家推荐一篇文章， [Matrix-TraceCanary的设计和原理分析手册](https://linjiang.tech/2019/11/12/matrix-trace-canary) 。

这一期的文章到这就结束了。以 **如何监测应用的 FPS** 为引子，重点讲述了屏幕刷新机制和 Choreographer 的工作流程，中间也掺杂了一些其他内容。后续的文章也将延续此风格，尽量涵括更多的知识点，敬请期待。

最后留下一个简单的问题，**为什么会发生掉帧？** 



























