```

```

### Activity 调用 finish 方法，会回调哪些生命周期方法？

```tex
Android中activity可以调用finish方法，结束自己，但是调用finish方法，activity到底会走那些生命周期方法下面直接上结论：
在onCreate中：onCreate->onDestroy
在onStart中：onCreate->onStart->onStop->onDestroy
在onResume中：onCreate->onStart->onResume->onPause->onStop->onDestroy
```

之所以是这样的，源码中给出了解释，Activity 会判断状态，只有没有被 finish 才会执行下一个生命周期。

```java
mInstrumentation.callActivityOnCreate(activity, r.state) // 函数中会判断：
if (!r.activity.mFinished) {
activity.performStart();
r.stopped = false;
}
/**执行完 onCreate()后，判断这时 activity 有没有finish ，没有就会接着执行 onStart()，否则会调用 destory()
执行完 onStart()后会执行 handleResumeActivity 函数，其中performResumeActivity 函数中：*/
if (r != null && !r.activity.mFinished) {
r.activity.performResume();
}
/**会调用 onResume 如果此时finish，就不会执行finish()，会调用ActivityManagerNative.getDefault()
.finishActivity(token, Activity.RESULT_CANCELED, null);执行销毁 */
```

# 面试官：为什么 Activity.finish()之后 10 S 才 onDestroy？

## 没有及时回调的 onStop/onDestroy

今天从 Activity.finish 开始写一遍流程。

在读源码之前，先来复现以下 10s onDestroy 的场景，写一个简单的 FirstActivity 跳转到 SecondActivity 的场景，并记录下各个生命周期和调用 finish() 的时间间隔。

```java
class FirstActivity : BaseLifecycleActivity() {

    private val binding by lazy { ActivityFirstBinding.inflate(layoutInflater) }
    var startTime = 0L

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)

        binding.goToSecond.setOnClickListener {
            start<SecondActivity>()
            finish()
            startTime = System.currentTimeMillis()
        }
    }

    override fun onPause() {
        super.onPause()
        Log.e("finish","onPause() 距离 finish() ：${System.currentTimeMillis() - startTime} ms")
    }

    override fun onStop() {
        super.onStop()
        Log.e("finish","onStop() 距离 finish() ：${System.currentTimeMillis() - startTime} ms")
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.e("finish","onDestroy() 距离 finish() ：${System.currentTimeMillis() - startTime} ms")
    }
}
```

`SecondActivity` 是一个普通的没有进行任何操作的空白 Activity 。点击按钮跳转到 SecondActivity，打印日志如下：

```java
FirstActivity: onPause，onPause() 距离 finish() ：5 ms
SecondActivity: onCreate
SecondActivity: onStart
SecondActivity: onResume
FirstActivity: onStop，onStop() 距离 finish() ：660 ms
FirstActivity: onDestroy，onDestroy() 距离 finish() ：663 ms
```

可以看到正常情况下，FirstActivity  回调 onPause 之后，SecondActivity 开始正常的生命周期流程，直到 onResume 被回调，对用户可见时，FirstActivity 才会回调onPause 和 onDestroy，时间间隔也在正常的范围内。

我们再模拟一个在 SecondActivity 启动时进行大量动画启动的场景，源源不断的向主线程消息队列塞消息。修改以下 SecondActivity 的代码。

```java
class SecondActivity : BaseLifecycleActivity() {

    private val binding by lazy { ActivitySecondBinding.inflate(layoutInflater) }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)

        postMessage()
    }

    private fun postMessage() {
        binding.secondBt.post {
            Thread.sleep(10)
            postMessage()
        }
    }
}
```

再来看一下日志：

```java
FirstActivity: onPause, onPause() 距离 finish() ：6 ms
SecondActivity: onCreate
SecondActivity: onStart
SecondActivity: onResume
FirstActivity: onStop, onStop() 距离 finish() ：10033 ms
FirstActivity: onDestroy, onDestroy() 距离 finish() ：10037 ms
```

FirstActivity 的 onPause 没有受到影响。因为在 Activity 的跳转中，目标 Activity 只有在前一个 Activity 的 onPause 之后才会开始正常的声明周期。而 onStop 和 onDestroy 整整过了 10S  才回调。

对比以上两个场景，我们可以猜测，当 SecondActivity 的主线程过于繁忙，没有机会停下来喘口气的时候，会造成 FirstActivity 无法及时回调 `onStop` 和 `onDestroy` 。基于以上猜测，我们就可以从 AOSP 中来寻找答案了。

接下来从 ASOP 中寻找答案。

## 从 Activity.finish() 说起

以下源代码基于 Android 11.0 版本。

```java
> Activity.java
public void finish() {
    finish(DONT_FINISH_TASK_WITH_ACTIVITY);
}
```

重载了带参数的 finish方法。参数是DONT_FINISH_TASK_WITH_ACTIVITY，含义也很明白，不会销毁 Activity 所在的任务栈。

```java
> Activity.java

private void finish(int finishTask) {
    // mParent 一般为 null，在 ActivityGroup 中会使用到
    if (mParent == null) {
        ......
        try {
			// Binder 调用 AMS.finishActivity()
            if (ActivityManager.getService()
                    .finishActivity(mToken, resultCode, resultData, finishTask)) {
                mFinished = true;
            }
        } catch (RemoteException e) {
        }
    } else {
        mParent.finishFromChild(this);
    }
    ......
}
```

这里的 mParent 大多数情况下都是 null，不需要考虑 else 分支的情况，一些大龄 Android 程序员可能会了解 ActiivityGroup，在这种情况下 mParent 可能不为 null。其中 binder 调用了 AMS.finishActivity 方法。

```java
> ActivityManagerService.java

public final boolean finishActivity(IBinder token, int resultCode, Intent resultData,
        int finishTask) {
    ......
    synchronized(this) {
        // token 持有 ActivityRecord 的弱引用
        ActivityRecord r = ActivityRecord.isInStackLocked(token);
        if (r == null) {
            return true;
        }
        ......
        try {
            boolean res;
            final boolean finishWithRootActivity =
                    finishTask == Activity.FINISH_TASK_WITH_ROOT_ACTIVITY;
            // finishTask 参数是 DONT_FINISH_TASK_WITH_ACTIVITY，进入 else 分支
            if (finishTask == Activity.FINISH_TASK_WITH_ACTIVITY
                    || (finishWithRootActivity && r == rootR)) {
                res = mStackSupervisor.removeTaskByIdLocked(tr.taskId, false,
                        finishWithRootActivity, "finish-activity");
            } else {
            	// 调用 ActivityStack.requestFinishActivityLocked()
                res = tr.getStack().requestFinishActivityLocked(token, resultCode,
                        resultData, "app-request", true);
            }
            return res;
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
}
```

注意方法参数中的 token 对象，token 对象是 Activity Record 的静态内部类，它持有外部 ActivityRecord 的弱引用，继承自 IApplicationToken.Stub，是一个 Binder 对象。ActivityRecord 就是对当前 Activity 的具体描述，包含了 Activity 的所有信息。

传入的 finishTask 方法的参数是 DONT_FINISH_TASK_WITH_ACTIVITY，所以接着会调用 ActivityStack.requestFinishActivityLocked() 方法。

```java
> ActivityStack.java

final boolean requestFinishActivityLocked(IBinder token, int resultCode,
        Intent resultData, String reason, boolean oomAdj) {
    ActivityRecord r = isInStackLocked(token);
    if (r == null) {
        return false;
    }

    finishActivityLocked(r, resultCode, resultData, reason, oomAdj);
    return true;
}

    final boolean finishActivityLocked(ActivityRecord r, int resultCode, Intent resultData,
        String reason, boolean oomAdj) {
        // PAUSE_IMMEDIATELY 为 true，在 ActivityStackSupervisor 中定义
    return finishActivityLocked(r, resultCode, resultData, reason, oomAdj, !PAUSE_IMMEDIATELY);
}
```

最后调用的是一个重载的 finishActivityLocked 方法。

```java
> ActivityStack.java

// 参数 pauseImmediately 是 false
final boolean finishActivityLocked(ActivityRecord r, int resultCode, Intent resultData,
        String reason, boolean oomAdj, boolean pauseImmediately) {
    if (r.finishing) { // 重复 finish 的情况
        return false;
    }

    mWindowManager.deferSurfaceLayout();
    try {
		// 标记 r.finishing = true，
		// 前面会做重复 finish 的检测就是依赖这个值
        r.makeFinishingLocked();
        final TaskRecord task = r.getTask();
        ......
		// 暂停事件分发
        r.pauseKeyDispatchingLocked();

        adjustFocusedActivityStack(r, "finishActivity");

		// 处理 activity result
        finishActivityResultsLocked(r, resultCode, resultData);

        // mResumedActivity 就是当前 Activity，会进入此分支
        if (mResumedActivity == r) {
            ......
            // Tell window manager to prepare for this one to be removed.
            r.setVisibility(false);

            if (mPausingActivity == null) {
				// 开始 pause mResumedActivity
                startPausingLocked(false, false, null, pauseImmediately);
            }
            ......
        } else if (!r.isState(PAUSING)) {
            // 不会进入此分支
            ......
        } 
        return false;
    } finally {
        mWindowManager.continueSurfaceLayout();
    }
}
```

调用 finish 之后肯定要先 Pause 当前 Activity，接着看 `startPausingLocked` 方法。

```java
> ActivityStack.java

    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
        ......
        ActivityRecord prev = mResumedActivity;

        if (prev == null) {
            // 没有 onResume 的 Activity，不能执行 pause
            if (resuming == null) {
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return false;
        }
        ......

        mPausingActivity = prev;
		// 设置当前 Activity 状态为 PAUSING
        prev.setState(PAUSING, "startPausingLocked");
        ......

        if (prev.app != null && prev.app.thread != null) {
            try {
                ......
                // 1. 通过 ClientLifecycleManager 分发生命周期事件
                // 最终会向 H 发送 EXECUTE_TRANSACTION 事件
                mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
                mPausingActivity = null;
            }
        } else {
            mPausingActivity = null;
        }
        ......
        // mPausingActivity 在前面已经赋值，就是当前 Activity
        if (mPausingActivity != null) { 
            ......
            if (pauseImmediately) { // 这里是 false，进入 else 分支
                completePauseLocked(false, resuming);
                return false;
            } else {
				// 2. 发送一个延时 500ms 的消息，等待 pause 流程一点时间
				// 最终会回调 activityPausedLocked() 方法
                schedulePauseTimeout(prev);
                return true;
            }
        } else {
            // 不会进入此分支
        }
    }
```

这里有两步重点操作。第一步是注释1 处通过 生命周期管理器 分发生命周期流程。第二部是发送一个延时 500 ms 的消息，等待以下 onPause 流程。但是如果第一步在 500 ms 内已经完成了这个流程，则会取消这个消息。所以这两部的最终逻辑其实是一致的。这里就直接看第一步：

```java
mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
```

ClientLifecycleManager 会向主线程的 Handler H 发送 **EXECUTE_TRANSACTION** 事件，调用 XXXActivityItem 的 execute 和 postExcute 方法。excute 方法中会 Binder 调用 ActivityThread 中对应的 handleXXXXActivity 方法。在这里就是 handlePauseActivity 方法，其中会通过 Instrumentation.callActivityOnPause(r.activity) 方法回调 Activity.onPause()。

```java
> Instrumentation.java

public void callActivityOnPause(Activity activity) {
    activity.performPause();
}
```

到这里，onPause 方法就被执行了，但是流程没有结束，接着就该显示下一个 Activity了。前面刚刚说过会调用 PauseActivityItem 的 excute 和 postExcute 方法。excute 方法 回调了当前 Activity.onPause，而 postExcute 方法就是去寻找要显示的 Activity。

```java
> PauseActivityItem.java

public void postExecute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    try {
        ActivityManager.getService().activityPaused(token);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
}
```

binder 调用了 AMS.activityPaused 方法。

```java
> ActivityManagerService.java

public final void activityPaused(IBinder token) {
    synchronized(this) {
        ActivityStack stack = ActivityRecord.getStackLocked(token);
        if (stack != null) {
            stack.activityPausedLocked(token, false);
        }
    }
}
```

调用了 ActivityStack.activityPausedLocked 方法。

```java
> ActivityStack.java

final void activityPausedLocked(IBinder token, boolean timeout) {
    final ActivityRecord r = isInStackLocked(token);
    if (r != null) {
        // 看这里
        mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
        if (mPausingActivity == r) {
            mService.mWindowManager.deferSurfaceLayout();
            try {
                // 看这里
                completePauseLocked(true /* resumeNext */, null /* resumingActivity */);
            } finally {
                mService.mWindowManager.continueSurfaceLayout();
            }
            return;
        } else {
            // 不会进入 else 分支
        }
    }
}
```

上面有这么一行代码 `mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r)` ，移除的就是之前延迟 500ms 的消息。接着看 `completePauseLocked()` 方法。

```java
> ActivityStack.java

private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
    ActivityRecord prev = mPausingActivity;

    if (prev != null) {
		// 设置状态为 PAUSED
        prev.setState(PAUSED, "completePausedLocked");
        if (prev.finishing) { // 1. finishing 为 true，进入此分支
            prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false,
                    "completedPausedLocked");
        } else if (prev.app != null) {
            // 不会进入此分支
        } else {
            prev = null;
        }
        ......
    }

    if (resumeNext) {
		// 当前获取焦点的 ActivityStack
        final ActivityStack topStack = mStackSupervisor.getFocusedStack();
        if (!topStack.shouldSleepOrShutDownActivities()) {
			// 2. 恢复要显示的 activity
            mStackSupervisor.resumeFocusedStackTopActivityLocked(topStack, prev, null);
        } else {
            checkReadyForSleep();
            ActivityRecord top = topStack.topRunningActivityLocked();
            if (top == null || (prev != null && top != prev)) {
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
        }
    }
    ......
}
```

这里分了两步走。注释1 处判断了 `finishing` 状态，还记得 finishing 在何处被赋值为 `true` 的吗？在 `Activity.finish() -> AMS.finishActivity() -> ActivityStack.requestFinishActivityLocked() -> ActivityStack.finishActivityLocked()` 方法中。所以接着调用的是 `finishCurrentActivityLocked()` 方法。注释2 处就是来显示应该显示的 Activity ，就不再追进去细看了。

再跟到 `finishCurrentActivityLocked()` 方法中，看这名字，肯定是要 **stop/destroy** 没跑了。

```java
> ActivityStack.java

/*
 * 把前面带过来的参数标出来
 * prev, FINISH_AFTER_VISIBLE, false,"completedPausedLocked"
 */
final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj,
        String reason) {
     
    // 获取将要显示的栈顶 Activity
    final ActivityRecord next = mStackSupervisor.topRunningActivityLocked(
            true /* considerKeyguardState */);

	// 1. mode 是 FINISH_AFTER_VISIBLE，进入此分支
    if (mode == FINISH_AFTER_VISIBLE && (r.visible || r.nowVisible)
            && next != null && !next.nowVisible) {
        if (!mStackSupervisor.mStoppingActivities.contains(r)) {
			// 加入到 mStackSupervisor.mStoppingActivities
            addToStopping(r, false /* scheduleIdle */, false /* idleDelayed */);
        }
		// 设置状态为 STOPPING
        r.setState(STOPPING, "finishCurrentActivityLocked");
        return r;
    }

    ......

    // 下面会执行 destroy，但是代码并不能执行到这里
    if (mode == FINISH_IMMEDIATELY
            || (prevState == PAUSED
                && (mode == FINISH_AFTER_PAUSE || inPinnedWindowingMode()))
            || finishingActivityInNonFocusedStack
            || prevState == STOPPING
            || prevState == STOPPED
            || prevState == ActivityState.INITIALIZING) {
        boolean activityRemoved = destroyActivityLocked(r, true, "finish-imm:" + reason);
        ......
        return activityRemoved ? null : r;
    }
    ......
}
```

注释 1 处 `mode` 的值是 `FINISH_AFTER_VISIBLE` ，并且现在新的 Activity 还没有 `onResume`，所以 `r.visible || r.nowVisible` 和 `next != null && !next.nowVisible` 都是成立的，并不会进入后面的 destroy 流程。虽然看到这还没得到想要的答案，但是起码是符合预期的。如果在这就直接 destroy 了，**延迟 10s 才 onDestroy** 的问题就无疾而终了。

对于这些暂时还不销毁的 Activity 都执行了 `addToStopping(r, false, false)` 方法。我们继续追进去。

```java
> ActivityStack.java

void addToStopping(ActivityRecord r, boolean scheduleIdle, boolean idleDelayed) {
    if (!mStackSupervisor.mStoppingActivities.contains(r)) {
        mStackSupervisor.mStoppingActivities.add(r);
        ......
    }
    ......
    // 省略的代码中，对 mStoppingActivities 的存储容量做了限制。超出限制可能会提前出发销毁流程
}
```

这些在等待销毁的 Activity 被保存在了 `ActivityStackSupervisor` 的 `mStoppingActivities` 集合中，它是一个 `ArrayList<ActivityRecord>` 。

整个 finish 流程就到此为止了。前一个 Activity 被保存在了 `ActivityStackSupervisor.mStoppingActivities` 集合中，新的 Activity 被显示出来了。

问题似乎进入了困境，什么时候回调 `onStop/onDestroy` 呢？其实这个才是根本问题。上面撸了一遍 finish() 并看不到本质，但是可以帮助我们形成一个完整的流程，这个一直是看 AOSP 最大的意义，**帮助我们把零碎的上层知识形成一个完整的闭环。**

## 是谁指挥着 onStop/onDestroy 的调用？

回到正题来，在 Activity 跳转过程中，为了保证流畅的用户体验，只要前一个 Activity 与用户不可交互，即 onPause 被回调之后，下一个 Activity 就要开始自己的声明周期流程了，所以 onStop/onDestroy 的调用时间是不确定的，甚至像文章开头的例子中，整整过了 10s 才回调。那么，到底是由谁来驱动 onstop/onDestroy 的执行呢？我们来看看下一个 Activity 的 onResume 过程。

直接看 ActivityThread.handleResumeActivity 方法，相信大家对声明周期的调用流程也很熟悉了。

```java
> ActivityThread.java

public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    ......
    // 回调 onResume
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ......
    final Activity a = r.activity;
    ......
    if (r.window == null && !a.mFinished && willBeVisible) {
        ......
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
				// 添加 decorView 到 WindowManager
                wm.addView(decor, l);
            } else {
                a.onWindowAttributesChanged(l);
            }
        }
    } else if (!willBeVisible) {
        ......
    }
    ......

    // 主线程空闲时会执行 Idler
    Looper.myQueue().addIdleHandler(new Idler());
}
```

handleResumeActivity 方法是整个 UI 显示流程的重中之重，它首先会回调 Activity.onResume ，然后将 DecorView 添加到 Window 上，其中包括了创建 ViewRootImpl，创建 Choreographer，与 WMS 进行 Binder 通信，注册了 vsync 信号，著名的 measure/draw/layout。这一块的源码很值得阅读。

在完成最终的界面绘制和显示之后，有这么一句代码 Looper.myQueue().addIdleHandler(new Idler()) 。IdleHandler 提供了一种机制，当主线程消息队列空闲时，会执行 IdleHandler 的回调方法。怎么算空闲，有两个条件。

在正常的消息处理机制之后，额外对 IdleHandler 进行了处理。当本次取到的 Message 为空或者需要延时处理的时候，就会去执行 `mIdleHandlers` 数组中的 IdleHandler 对象。其中还有一些关于 pendingIdleHandlerCount 的额外逻辑来防止循环处理。

所以，不出意外的话，当新的 Activity 完成页面绘制并显示之后，主线程就可以停下歇一歇，来执行 `IdleHandler` 了。再回来 `handleResumeActivity()` 中来，`Looper.myQueue().addIdleHandler(new Idler())` ，这里的 `Idler` 是 `IdleHandler` 的一个具体实现类。

```java
> ActivityThread.java

private class Idler implements MessageQueue.IdleHandler {
    @Override
    public final boolean queueIdle() {
        ActivityClientRecord a = mNewActivities;
        ......
        }
        if (a != null) {
            mNewActivities = null;
            IActivityManager am = ActivityManager.getService();
            ActivityClientRecord prev;
            do {
                if (a.activity != null && !a.activity.mFinished) {
                    try {
                        // 调用 AMS.activityIdle()
                        am.activityIdle(a.token, a.createdConfig, stopProfiling);
                        a.createdConfig = null;
                    } catch (RemoteException ex) {
                        throw ex.rethrowFromSystemServer();
                    }
                }
                prev = a;
                a = a.nextIdle;
                prev.nextIdle = null;
            } while (a != null);
        }
        ......
        return false;
    }
}
```

Binder 调用了 `AMS.activityIdle()` 。

```java
> ActivityManagerService.java

public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
        
    final long origId = Binder.clearCallingIdentity();
    synchronized (this) {
        ActivityStack stack = ActivityRecord.getStackLocked(token);
        if (stack != null) {
            ActivityRecord r =
                    mStackSupervisor.activityIdleInternalLocked(token, false /* fromTimeout */,
                            false /* processPausingActivities */, config);
            ......
        }
    }
}
```

调用了 `ActivityStackSupervisor.activityIdleInternalLocked()` 方法。

```java
> ActivityStackSupervisor.java

final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
        boolean processPausingActivities, Configuration config) {

    ArrayList<ActivityRecord> finishes = null;
    ArrayList<UserState> startingUsers = null;
    int NS = 0;
    int NF = 0;
    boolean booting = false;
    boolean activityRemoved = false;

    ActivityRecord r = ActivityRecord.forTokenLocked(token);
       
    ......
    // 获取要 stop 的 Activity
    final ArrayList<ActivityRecord> stops = processStoppingActivitiesLocked(r,
            true /* remove */, processPausingActivities);
    NS = stops != null ? stops.size() : 0;
    if ((NF = mFinishingActivities.size()) > 0) {
        finishes = new ArrayList<>(mFinishingActivities);
        mFinishingActivities.clear();
    }

    // 该 stop 的 stop
    for (int i = 0; i < NS; i++) {
        r = stops.get(i);
        final ActivityStack stack = r.getStack();
        if (stack != null) {
            if (r.finishing) {
                stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false,
                        "activityIdleInternalLocked");
            } else {
                stack.stopActivityLocked(r);
            }
        }
    }

    // 该 destroy 的 destroy
    for (int i = 0; i < NF; i++) {
        r = finishes.get(i);
        final ActivityStack stack = r.getStack();
        if (stack != null) {
            activityRemoved |= stack.destroyActivityLocked(r, true, "finish-idle");
        }
    }
    ......

    return r;
}
```

`stops` 和 `finishes` 分别是要 stop 和 destroy 的两个 ActivityRecord 数组。`stops` 数组是通过 `ActivityStackSuperVisor.processStoppingActivitiesLocked()` 方法获取的，追进去看一下

```
> ActivityStackSuperVisor.java

final ArrayList<ActivityRecord> processStoppingActivitiesLocked(ActivityRecord idleActivity,
        boolean remove, boolean processPausingActivities) {
    ArrayList<ActivityRecord> stops = null;

    final boolean nowVisible = allResumedActivitiesVisible();
    // 遍历 mStoppingActivities
    for (int activityNdx = mStoppingActivities.size() - 1; activityNdx >= 0; --activityNdx) {
        ActivityRecord s = mStoppingActivities.get(activityNdx);
        ......
    }
    return stops;
}
```

中间的详细处理逻辑就不看了，我们只需要关注这里遍历的是 **ActivityStackSuperVisor 中的 mStoppingActivities 集合** 。在前面分析 `finish()` 流程到最后的 `addToStopping()` 方法时提到过，

> **这些在等待销毁的 Activity 被保存在了 ActivityStackSupervisor 的 mStoppingActivities 集合中，它是一个 ArrayList<ActivityRecord> 。**

看到这里，终于打通了流程。再回头想一下文章开头的例子，由于人为的在 SecondActivity 不间断的向主线程塞消息，导致 Idler 迟迟无法被执行，`onStop/onDestroy` 也就不会被回调。

## 谁让 onStop/onDestroy 延迟了 10s ？

对，**不会被回调。** 可实际情况是这样吗？并不是，明明是过了 10s 被回调。这就说明了即使主线程迟迟没有机会执行 Idler，系统仍然提供了兜底机制，防止已经不需要的 Activity 长时间无法被回收，从而造成内存泄漏等问题。从实际现象就可以猜测到，这个兜底机制就是 onResume 之后 10s 主动去进行释放操作。

再回到之前显示待跳转 Activity 的 `ActivityStackSuperVisor.resumeFocusedStackTopActivityLocked()` 方法。我这里就不带着大家追进去了，直接给出调用链。

> ASS.resumeFocusedStackTopActivityLocked() -> ActivityStack.resumeTopActivityUncheckedLocked() -> ActivityStack.resumeTopActivityInnerLocked() -> ActivityRecord.completeResumeLocked() -> ASS.scheduleIdleTimeoutLocked()

```java
> ActivityStackSuperVisor.java

void scheduleIdleTimeoutLocked(ActivityRecord next) {
    Message msg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG, next);
    mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT);
}
复制代码
```

`IDLE_TIMEOUT` 的值是 10，这里延迟 10s 发送了一个消息。这个消息是在 `ActivityStackSupervisorHandler` 中处理的。

```java
private final class ActivityStackSupervisorHandler extends Handler {
......
case IDLE_TIMEOUT_MSG: {
    activityIdleInternal((ActivityRecord) msg.obj, true /* processPausingActivities */);
    } break;
......
}

void activityIdleInternal(ActivityRecord r, boolean processPausingActivities) {
    synchronized (mService) {
        activityIdleInternalLocked(r != null ? r.appToken : null, true /* fromTimeout */,
                processPausingActivities, null);
    }
}
复制代码
```

忘记 `activityIdleInternalLocked` 方法的话可以 ctrl+F 向上搜索一下。如果 10s 内主线程执行了 Idler 的话，就会移除这个消息。

到这里，所有的问题就全部理清了。

Activity 的 onStop/onDestroy 是依赖 IdleHandler 来回调的，正常情况下当主线程空闲时会调用。但是由于某些特殊场景下的问题，导致主线程迟迟无法空闲，onStop/onDestroy 也会迟迟得不到调用。但这并不意味着 Activity 永远得不到回收，系统提供了一个兜底机制，当 onResume 回调 10s 之后，如果仍然没有得到调用，会主动触发。

虽然有兜底机制，但无论如何这肯定不是我们想看到的。如果我们项目中的 onStop/onDestroy 延迟了 10s 调用，该如何排查问题呢？可以利用 `Looper.getMainLooper().setMessageLogging()` 方法，打印出主线程消息队列中的消息。每处理一条消息，都会打印如下内容：

```
logging.println(">>>>> Dispatching to " + msg.target + " " + msg.callback + ": " + msg.what);
logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
复制代码
```

另外，由于 `onStop/onDestroy` 调用时机的不确定性，在做资源释放等操作的时候，一定要考虑好，以避免产生资源没有及时释放的情况。

















