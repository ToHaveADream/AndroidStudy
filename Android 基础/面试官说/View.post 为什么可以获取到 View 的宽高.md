## 面试官：View.post（）为什么能够获取到 View 的宽高？

### 小测试：哪里可以获取到 View 的宽高？

先来一段测试代码：

```java
class MainActivity : BaseLifecycleActivity() {

    private val binding by lazy { ActivityMainBinding.inflate(layoutInflater) }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)

        // 在 onCreate() 中获取宽高
        Log.e("measure","measure in onCreate: width=${window.decorView.width}, height=${window.decorView.height}")

        // 在 View.post() 回调中获取宽高
        binding.activity.post {
            Log.e("measure","measure in View.post: width=${window.decorView.width}, height=${window.decorView.height}")
        }
    }

    override fun onResume() {
        super.onResume()
        // 在 onResume() 回调中获取宽高
        Log.e("measure","measure in onResume: width=${window.decorView.width}, height=${window.decorView.height}")
    }
}
```

大多数人都能直截了当的给出答案：

```
E/measure: measure in onCreate: width=0, height=0
E/measure: measure in onResume: width=0, height=0
E/measure: measure in View.post: width=1080, height=2340
```

在 `onCreate()` 和 `onResume()` 中是无法获取到宽高的，而 `View.post()` 回调中可以。从日志打印顺序可以看出来，`View.post()` 回调中的打印语句是最后执行的。

抛开代码来思考一下这个问题，**什么时候可以获取到 View 的宽高？** 毫无疑问，最起码肯定得在 **View 被测量** 这个时间点之后。从上面的结果来看，`onCreate()` 和 `onResume()` 发生在这个时间点之前，`View.post()` 的回调发生在这个时间点之后。我们只要搞清楚这个时间点，问题就迎刃而解了。

## View 在什么时间点被测量？

大家知道 view 的绘制流程发生在 onResume 流程（并不是 Activity.onResume() 回调）中，但我还是决定从头开始说起。

当冷启动（应用进程不存在）一个 App 时，首先要和 zygote 建立 Socket 连接，将创建进程需要的参数发送给 zygote，zygote 服务端接收到参数之后调用 *ZygoteConnection.processOneCommand()*处理参数，并 fork 出应用进程，最后通过 `findStaticMain` 找到 `ActivityThread` 类的 `main` 方法并执行，应用进程就启动了。

`ActivityThread` 虽然不是一个线程类，但它是运行在主线程的，你就把它认为是主线程也没有关系。在`main()` 方法中，创建了 `ActivityThread`对象，调其 `attach` 方法，并开启了主线程消息循环，基于事件的消息队列机制就开始工作了。

- 在 `ActivityThread.attach()`  方法中，Binder 调用了 `AMS.attachApplication`方法，进行客户端的准备工作，创建 context，创建 Application 等等。
- `mStackSupervisor.attachAppliocationLocked(app)`，最终调用到了 `realStartActivityLocked` 启动 Activity

Activity 的启动流程就不赘述了，之前写过一篇裹脚布式的文章。这里直接跳到 `ActivityThread.performLaunchActivity()`方法。

```java
> ActivityThread.java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    // 获取 ComponentName
    ComponentName component = r.intent.getComponent();
    ...
    // 创建 ContextImpl 对象
    ContextImpl appContext = createBaseContextForActivity(r);
    ...
    // 反射创建 Activity 对象
    activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
    ......
    // 创建 Application 对象
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    ......      
	// attach 方法中会创建 PhoneWindow 对象
    activity.attach(appContext, this, getInstrumentation(), r.token,
            r.ident, app, r.intent, r.activityInfo, title, r.parent,
            r.embeddedID, r.lastNonConfigurationInstances, config,
            r.referrer, r.voiceInteractor, window, r.configCallback);
    // 执行 onCreate()
    if (r.isPersistable()) {
        mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
    } else {
        mInstrumentation.callActivityOnCreate(activity, r.state);
    }
    ...
    return activity;
}
```

`mInstrumentation.callActivityOnCreate()`方法最终会回调 `Activity.onCreate()`方法；到这里，`setContentView()`方法就被执行了。`setContentView` 逻辑很复杂，但是干的事情很直白，创建 `DecorView`,然后根据我们传入的布局文件 id 解析 xml，将得到的 view 塞进 DecorView 中，注意，到现在，我们得到的是一个 **空壳子 View 树**，它没有被添加屏幕上，其实也不能添加到屏幕上。所以，在 `onCreate`回调中获取视图宽高显然是不可取的。

看完 `onCreate`,我们跳过 `onStart`，里面没干啥太重要的事情，直接来到 `onResume`。

> 注：Activity 的生命周期是由 `ClientLifecycleManager` 类来调度的，具体原理可以看这篇文章 [从源码看 Activity 生命周期](https://luyao.tech/archives/activitylifecycle1#clienttransactionaddcallback) 。

```java
> ActivityThread.java

public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    ...
    // 1. 回调 onResume
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ···
    View decor = r.window.getDecorView();
    decor.setVisibility(View.INVISIBLE);
    ViewManager wm = a.getWindowManager();
    WindowManager.LayoutParams l = r.window.getAttributes();
    // 2. 添加 decorView 到 WindowManager
    wm.addView(decor, l);
    ...
}
```

两件事，回调 `onResume`和添加 DecorView 到 WindowManager。所以，在 `onResume`回调中获取 view 的宽高其实和 `onCreate`中没啥区别，都获取不到。

`wm.addView(decor,l)`最终调用到`WindowManagerGlobal.addView()`。

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ...
    // 1. 重点，初始化 ViewRootImpl
    root = new ViewRootImpl(view.getContext(), display);
    // 2. 重点，发起绘制并显示到屏幕上
    root.setView(view, wparams, panelParentView);
```

先来看注释1处的 `ViewRootImpl`的构造函数。

```java
public ViewRootImpl(Context context, Display display) {
    ...
	// 1. IWindowSession 代理对象，与 WMS 进行 Binder 通信
    mWindowSession = WindowManagerGlobal.getWindowSession();
    ...
    // 2.
    mWidth = -1;
    mHeight = -1;
    ...
    // 3. 初始化 AttachInfo
    // 记住 mAttachInfo 是在这里被初始化的
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this, context);
    ...
	// 4. 初始化 Choreographer，通过 Threadlocal 存储
    mChoreographer = Choreographer.getInstance();
}
```

- 初始化 mWindowSession，它可以与 WMS 进行 Binder 通信
- 这里看到宽高还未赋值
- 初始化 Attach Info
- 初始化 Choreographer

再看注释 2 处的 `ViewRootImpl.setView() `方法。

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

            // 3. 将 decorView 的 parent 赋值为 ViewRootImpl
            view.assignParent(this);
        }
    }
}
```

`requestLayout`方法发起了首次绘制

```java
> ViewRootImpl.java

public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
		// 检查线程
        checkThread();
        mLayoutRequested = true;
        // 重点
        scheduleTraversals();
    }
}
```

`ViewRootImpl.scheduleTraversals`方法大致如下：

- `ViewRootImpl.scheduleTraversals()`方法中会建立同步屏障，优先处理异步消息。通过 `Choreographer.postCallback()`方法提交了任务 `mTraversalRunnable`,这个任务就是负责 View 的测量，布局，绘制。
- `Choreographer.postCallback()`方法通过 `DisplayEventReceiver.nativeScheduleVsync()`方法向系统底层注册下一次`vsync`信号的监听。当下一次 `vsync`来临时，系统会回调其 `dispatchVsync`方法，最终回调 `FrameDisplayEventReceiver.onVsync()` 方法。
-  `FrameDisplayEventReceiver.onVsync()` 方法中取到之前提交的`mTraversalRunnable` 并执行，这样就完成了一次绘制流程。

`mTraversalRunnable`中执行的是 `doTraversal`方法。

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

```java
> ViewRootImpl.java

private void performTraversals() {
    ...
    // 1. 绑定 Window，重点记忆一下
    host.dispatchAttachedToWindow(mAttachInfo, 0);
    mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);

    getRunQueue().executeActions(mAttachInfo.mHandler);

    // 2. 请求 WMS 计算窗口大小
    relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);

    // 3. 测量
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

    // 4. 布局
    performLayout(lp, mWidth, mHeight);

    // 5. 绘制
    performDraw();
}
```

View 被测量的时机已经找到了，现在来验证一下 `view.post`是不是在这个时机执行回调的。

## 探秘 View.post()

```java
> View.java

public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        // 1. attachInfo 不为空，通过 mHandler 发送
        return attachInfo.mHandler.post(action);
    }
    // 2. attachInfo 为空，放入队列中
    getRunQueue().post(action);
    return true;
}
```

这里的关键是 `attachInfo`是否为空。在上一节中介绍过，再来回顾一下：

- `attachInfo`是在 `ViewRootImpl`的构造函数中初始化的
- `ViewRootImpl`是在`WindowManagerGlobal.addView`创建的
- `WindowManagerGlobal.addView`是在 ActivityThread 的 `handleResumeActivity`中调用的，但是是在 `Activity.onResume`回调之后

所以，如果 `attachInfo`不为空的话，至少已经处在进行视图绘制的这次消息处理当中。把 post 方法要执行的 Runnable 利用 Handler 发送出去，当包含这个 Runnable 的 Message 被执行时，是一定可以获取到 View 的宽高的。

在 `onCreate`和 `onResume`这两个回调中，`attachInfo`肯定是空的，这时候需要依赖`getRunQueue().post(action)`.原理也很简单，把 `post`方法要执行的 Runnable 存储在一个队列中，在合适的实际（View 已被测量）拿出来执行。先来看看 `getRunQueue`拿到的是一个什么队列。

```java
> View.java

private HandlerActionQueue getRunQueue() {
    if (mRunQueue == null) {
        mRunQueue = new HandlerActionQueue();
    }
    return mRunQueue;
}
```

```java
public class HandlerActionQueue {
    private HandlerAction[] mActions;
    private int mCount;

    public void post(Runnable action) {
        postDelayed(action, 0);
    }

    // 发送任务
    public void postDelayed(Runnable action, long delayMillis) {
        final HandlerAction handlerAction = new HandlerAction(action, delayMillis);

        synchronized (this) {
            if (mActions == null) {
                mActions = new HandlerAction[4];
            }
            mActions = GrowingArrayUtils.append(mActions, mCount, handlerAction);
            mCount++;
        }
    }

    // 执行任务
    public void executeActions(Handler handler) {
        synchronized (this) {
            final HandlerAction[] actions = mActions;
            for (int i = 0, count = mCount; i < count; i++) {
                final HandlerAction handlerAction = actions[i];
                handler.postDelayed(handlerAction.action, handlerAction.delay);
            }

            mActions = null;
            mCount = 0;
        }
    }

    ...

    private static class HandlerAction {
        final Runnable action;
        final long delay;

        public HandlerAction(Runnable action, long delay) {
            this.action = action;
            this.delay = delay;
        }

        public boolean matches(Runnable otherAction) {
            return otherAction == null && action == null
                    || action != null && action.equals(otherAction);
        }
    }
}
```

队列`HandlerActionQueue`是一个初始容量为 4 的 `HandlerAction`数组。HandlerAction 有两个成员变量，要执行的 Runnable 和延迟执行的时间。

队列的执行逻辑在 `excuteActions(Handler)`方法中，通过传入的 Handler 进行任务分发。现在我们只要找到 `excuteActions`的调用时机就可以了。在 View.java 中就可以找到，在 `dispatchAttachedToWindow`方法中进行了分发。

```java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    ...
    if (mRunQueue != null) {
            // 分发任务
            mRunQueue.executeActions(info.mHandler);
            mRunQueue = null;
        }
    // 回调 onAttachedToWindow()
    onAttachedToWindow();
}
```

注意注释1 处，**明明是先调用的 `dispatchAttachedToWindow`,再进行的测量流程，为什么 `dispatchAttachedToWindow`中可以获取到 View 的宽高呢？**

首先，`performTraversals()`是再主线程消息队列的一次消息处理过程中执行的，而`dispatchAttachedToWindow()`间接调用的`mRunQueue.executeActions()`发送的任务也是通过 Handler 发送到主线程消息队列的，那么它的执行一定再这次的 `performTraversals()`方法执行之后。所以，在这里获取 View 的宽高没有问题。

## 总结

**根据 ViewRootImpl 是否已经创建，View.post() 会执行不同的逻辑。如果 ViewRootImpl 已经创建，即 mAttachInfo 已经初始化，直接通过 Handler 发送消息来执行任务。如果 ViewRootImpl 未创建，即 View 尚未开始绘制，会将任务保存为 HandlerAction，暂存在队列 HandlerActionQueue 中，等到 View 开始绘制，执行 performTraversal() 方法时，在 dispatchAttachedToWindow() 方法中通过 Handler 分发 HandlerActionQueue 中暂存的任务。**

**另外要注意，View 绘制是发生在一次 Meesage 处理过程中的，View.post() 执行的任务也是发生在一次 Message 处理过程中的，它们一定是有先后顺序的。**

## 还可以怎么获取视图宽高？

除了通过`view.post`获取视图宽高之后，还有两种比较推荐的方式：

第一种，`onWindowFocusChanged`。

```java
override fun onWindowFocusChanged(hasFocus: Boolean) {  
    super.onWindowFocusChanged(hasFocus)
    if (hasFocus){
        ...
    }
}
```

第二种，`OnGlobalLayoutListener`。

```java
binding.dialog.viewTreeObserver.addOnGlobalLayoutListener(object : ViewTreeObserver.OnGlobalLayoutListener{
    override fun onGlobalLayout() {
        binding.dialog.viewTreeObserver.removeOnGlobalLayoutListener(this)
        ...
    }
})
```

这两种方法都可能被调用多次。当 Activity 获取和失去焦点的时候，`onWindowFocusChanged`都会调用。当 View 树发生状态变化时，`OnGlobalLayoutListener`也会调用多次，可以根据需要移除监听。





































