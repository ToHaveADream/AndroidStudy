# Android 之 Activity 启动流程详解（基于 api28)

#### 前言

Android 作为四大组件之一，它的启动绝对没有这么简单。这里涉及到了系统服务进程，启动过程细节非常多，这里只展示主体流程。

因为涉及到不同进程之间的通信：系统服务进程和本地进程，在最新版本的 Android 使用的是 AIDL 来阔进程通信。所以需要对 AIDL 有一定的了解，会帮助你理解整个启动流程。

#### 普通 Activity 的创建

普通Activity 的创建也就是我们平常在代码中采用 `startActivity(Intent intent)`方法来创建 Activity 的方式。总体流程如下：

![](/picture/data-09.image)

启动过程设计到两个进程：本地进程和系统服务进程。本地进程也就是我们的应用所在的进程，系统服务进程为所有应用共用的进程。整体思路是：

- Activity 向 Instrumentation 请求创建
- Instrumentation 通过 AMS 在本地进程的 IBinder 接口，访问 AMS，这里采用的跨进程技术是 AIDL。
- 然后 AMS 进程一系列的工作，如判断该 Activity是否存在，启动模式是什么，有没有进行注册等等。
- 通过 ClientLifeCycleManager，利用本地进程在系统服务进程的 IBinder 接口直接访问本地 ActivityThread

> ApplicationThread 是 ActivityThread 的内部类，IApplicationThread 是在远程服务端的Binder 接口

- ApplicationThread 接收到服务端的事务后，把事务直接转交给 ActivityThread 处理
- ActivityThread 通过 Instrumentation 利用类加载器进程实例创建，同时利用 Instrumentation 回调 activity 生命周期函数

这里涉及到了两个进程，本地进程主要负责创建 activity 以及回调生命周期，服务进程主要判断该 Activity 是否合法，是否需要创建 Activity 栈等等，进程之间就涉及到了进程通信：AIDL。

接下来介绍几个比较关键的类：

- Instrumentation 是 Activity 与外界联系的类（不是 activity 本身的统称外界，相对 Activity 而言），activity 通过 Instrumentation 来请求创建，ActivityThread 通过 Instrumentation 来创建 Activity 和调用 activity 的生命周期
- ActivityThread，每个应用程序唯一一个实例，负责对 Activity 创建的管理，而 ApplicationThread 只是应用程序和服务端进程通信的类而已，只负责通信，把 AMS 的任务交给 ActivityThread。
- AMS，全称为 ActivityManagerService，负责统筹服务端对 Activity 的创建流程

#### 根Activity 的创建

根 Activity也就是我们点击桌面图标的时候，应用程序第一个 Activity 启动的流程。这里侧重讲解多个进程之间的关系，先看整体流程图。

![](/picture/data-10.image)

主要涉及四个进程：

- Launcher进程，也就是桌面进程
- 系统服务进程，也就是 AMS所在进程
- Zygote 进程，负责创建进程
- 应用程序进程，也就是即将要启动的进程

主要流程：

1. Launcher 进程请求 AMS 创建 Activity
2. AMS 请求 Zygote 创建进程
3. Zygote 通过 fork 自己来创建进程，并通知 AMS 创建完成
4. AMS 通知应用进程创建根 Activity

**和普通 Activity 的创建很像，主要多了创建进程这一步。**

#### 源码讲解

Activity 请求 AMS 的过程

##### 流程图

![](/picture/data-11.image)

##### 源码

1. 系统通过调用 Launcher 的 startActivitySafely 方法来启动应用程序。Launcher 是一个类，负责启动根 Activity。

> ​       这一步是根 Activity 启动才有的流程，普通启动是没有的，放在这里作为一点补充

```java
packages/apps/Launcher3/src/com/android/launcher3/Launcher.java/;
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
    	//这里调用了父类的方法，继续查看父类的方法实现
        boolean success = super.startActivitySafely(v, intent, item);
        ...
        return success;
    }
```

```java
packages/apps/Launcher3/src/com/android/launcher3/BaseDraggingActivity.java/;
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
        ...
        // Prepare intent
        //设置标志singleTask，意味着在新的栈打开
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        if (v != null) {
            intent.setSourceBounds(getViewBounds(v));
        }
        try {
            boolean isShortcut = Utilities.ATLEAST_MARSHMALLOW
                    && (item instanceof ShortcutInfo)
                    && (item.itemType == Favorites.ITEM_TYPE_SHORTCUT
                    || item.itemType == Favorites.ITEM_TYPE_DEEP_SHORTCUT)
                    && !((ShortcutInfo) item).isPromise();
            //下面注释1和注释2都是直接采用startActivity进行启动。注释1会做一些设置
            //BaseDraggingActivity是继承自BaseActivity，而BaseActivity是继承自Activity
            //所以直接就跳转到了Activity的startActivity逻辑。
            if (isShortcut) {
                // Shortcuts need some special checks due to legacy reasons.
                startShortcutIntentSafely(intent, optsBundle, item);//1
            } else if (user == null || user.equals(Process.myUserHandle())) {
                // Could be launching some bookkeeping activity
                startActivity(intent, optsBundle);//2
            } else {
                LauncherAppsCompat.getInstance(this).startActivityForProfile(
                        intent.getComponent(), user, intent.getSourceBounds(), optsBundle);
            }
               
           ...
        } 
    	...
        return false;
    }
```

2、Activity 通过 Instrumentation 来启动 Activity

```java
/frameworks/base/core/java/android/app/Activity.java/;
public void startActivity(Intent intent, @Nullable Bundle options) {
    	//最终都会跳转到startActivityForResult这个方法
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
   
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
    	//mParent是指activityGroup，现在已经采用Fragment代替，这里会一直是null
    	//下一步会通过mInstrumentation.execStartActivity进行启动
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);//1
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            ...
        }
    ...
}
```

3、Instrumentation 请求 AMS 进行启动，该类的作用是监控应用程序和系统的交互，到此为止，任务就交给 AMS了，AMS　进行一系列处理后，会通过本地的接口 IActivityManager 来进行回调启动 Activity。

```java
/frameworks/base/core/java/android/app/Instrumentation.java/;
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
    ...
    //这个地方比较复杂，先说结论。下面再进行解释    
    //ActivityManager.getService()获取到的对象是ActivityManagerService，简称AMS
    //通过AMS来启动activity。AMS是全局唯一的，所有的活动启动都要经过他的验证，运行在独立的进程中
    //所以这里是采用AIDL的方式进行跨进程通信，获取到的对象其实是一个IBinder接口
           
    //注释2是进行检查启动结果，如果异常则抛出，如没有注册。
    try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);//1
            checkStartActivityResult(result, intent);//2
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
       
}
```

> 这一步是通过 AIDL 技术进行跨进程通信，拿到 AMS 的代理对象，把启动任务交给了 AMS。
>
> ```java
> /frameworks/base/core/java/android/app/ActivityManager.java/;
> //单例类
> public static IActivityManager getService() {
> return IActivityManagerSingleton.get();
> }
> 
> private static final Singleton<IActivityManager> IActivityManagerSingleton =
> new Singleton<IActivityManager>() {
> @Override
> protected IActivityManager create() {
> //得到AMS的IBinder接口
> final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
> //转化成IActivityManager对象。远程服务实现了这个接口，所以可以直接调用这个
> //AMS代理对象的接口方法来请求AMS。这里采用的技术是AIDL
> final IActivityManager am = IActivityManager.Stub.asInterface(b);
> return am;
> }
> };
> ```

##### AMS处理请求的流程

###### 流程图

![](/picture/data-12.image)

###### 源码

1. 接下来看 AMS 的实现逻辑，AMS这部分的源码是通过 ActivityStartController 来创建一个 ActivityStarter，然后把逻辑都交给 ActivityStarter 去执行。ActivityStarter 是 Android 7.0 加入的类。

```java
 /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java/;
//跳转到startActivityAsUser
//注意最后多了一个参数UserHandle.getCallingUserId()，表示调用者权限
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
   
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivity");
   
        userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");
   
        // TODO: Switch to user app stacks here.
    	//这里通过ActivityStartController获取到ActivityStarter，通过ActivityStarter来
    	//执行启动任务。这里就把任务逻辑给到了AcitivityStarter
        return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();
   
    }
```

> ActivityStartController 获取 ActivityStarter
>
> ```java
> /frameworks/base/services/core/java/com/android/server/am/ActivityStartController.java/;
> //获取到ActivityStarter对象。这个对象仅使用一次，当他的execute被执行后，该对象作废
> ActivityStarter obtainStarter(Intent intent, String reason) {
> return mFactory.obtain().setIntent(intent).setReason(reason);
> }
> ```

2、这部分主要是 ActivityStarter 的源码内容，设计到的源码非常多。AMS 把整个启动逻辑都丢给 ActivityStarter 去处理了。这里主要做启动前处理，创建进程等。

```java
/frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java/;
//这里需要做启动预处理，执行startActivityMayWait方法
int execute() {
        try {
           	...
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            }
            ...
        } 
    	...
    }
   
//启动预处理
private int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
            int userId, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup) {
        ...
		//跳转startActivity
         final ActivityRecord[] outRecord = new ActivityRecord[1];
     	int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                allowPendingRemoteAnimationRegistryLookup);
}
   
//记录启动进程和activity的信息
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        SafeActivityOptions options,
        boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
        TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup) {
    ...
    //得到Launcher进程    
    ProcessRecord callerApp = null;
    if (caller != null) {
        callerApp = mService.getRecordForAppLocked(caller);
        ...
    }
    ...
    //记录得到的activity信息
    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
            resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
            mSupervisor, checkedOptions, sourceRecord);
    if (outActivity != null) {
        outActivity[0] = r;
    }
    ...
    mController.doPendingActivityLaunches(false);
	//继续跳转
    return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
            true /* doResume */, checkedOptions, inTask, outActivity);
}
   
//跳转startActivityUnchecked
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {
    int result = START_CANCELED;
    try {
        mService.mWindowManager.deferSurfaceLayout();
        //跳转
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity);
    } 
    ...
    return result;
}
   
//主要做与栈相关的逻辑处理，并跳转到ActivityStackSupervisor进行处理
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
    ...
    int result = START_SUCCESS;
    //这里和我们最初在Launcher设置的标志FLAG_ACTIVITY_NEW_TASK相关，会创建一个新栈
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
            && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        result = setTaskFromReuseOrCreateNewTask(taskToAffiliate, topStack);
    }
    ...
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
            mStartActivity.getTask().topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
            || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            ...
        } else {
            if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityUnchecked");
            }
            //跳转到ActivityStackSupervisor进行处理
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                                                            mOptions);
        }
    }
}
```

3、ActivityStackSupervisor 主要负责做 Activity 栈的相关工作，会结合 ActivityStack 来进行工作。主要判断 activity 的状态，是否处于站定或者处于停止状态等

```java
/frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java/;
   
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
	...
    //判断要启动的activity是不是出于停止状态或者Resume状态
    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
    if (r == null || !r.isState(RESUMED)) {
        mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
    } else if (r.isState(RESUMED)) {
        // Kick off any lingering app transitions form the MoveTaskToFront operation.
        mFocusedStack.executeAppTransition(targetOptions);
    }
    return false;
}
```

4、ActivityStack 主要处理 activity 在栈中的状态

```java
/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java/;
//跳转resumeTopActivityInnerLocked
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mStackSupervisor.inResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }
    boolean result = false;
    try {
        // Protect against recursion.
        mStackSupervisor.inResumeTopActivity = true;
        //跳转resumeTopActivityInnerLocked
        result = resumeTopActivityInnerLocked(prev, options);
	...
    } finally {
        mStackSupervisor.inResumeTopActivity = false;
    }
    return result;
}
   
//跳转到StackSupervisor.startSpecificActivityLocked，注释1
@GuardedBy("mService")
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    if (next.app != null && next.app.thread != null) {
        ...
    } else {
        ...
        mStackSupervisor.startSpecificActivityLocked(next, true, true);//1
    }     
    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
    return true;
}
```

5、这里又回到了 ActivityStackSupervisor，判断进程是否已经创建，未创建抛出异常，然后创建事务交汇本地执行，这里的事务很关键，Activity 执行的工作就是这个事务，事务的内容是里面的 item，所以要注意下面的 两个 item。

```java
/frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java/;
   
void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    //得到即将启动的activity所在的进程
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);
    getLaunchTimeTracker().setLaunchTime(r);
   
    //判断该进程是否已经启动,跳转realStartActivityLocked，真正启动活动
    if (app != null && app.thread != null) {
        try {
            if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                    || !"android".equals(r.info.packageName)) {
                app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                        mService.mProcessStats);
            }
            realStartActivityLocked(r, app, andResume, checkConfig);//1
            return;
        } 
        ...
    }
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}
   
   
//主要创建事务交给本地执行
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
    boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    //创建启动activity的事务ClientTransaction对象
    // Create activity launch transaction.
    final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
            r.appToken);
    // 添加LaunchActivityItem，该item的内容是创建activity
    clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
            System.identityHashCode(r), r.info,
            // TODO: Have this take the merged configuration instead of separate global
            // and override configs.
            mergedConfiguration.getGlobalConfiguration(),
            mergedConfiguration.getOverrideConfiguration(), r.compat,
            r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
            r.persistentState, results, newIntents, mService.isNextTransitionForward(),
            profilerInfo));
   
    // Set desired final state.
    //添加执行Resume事务ResumeActivityItem,后续会在本地被执行
    final ActivityLifecycleItem lifecycleItem;
    if (andResume) {
        lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
    } else {
        lifecycleItem = PauseActivityItem.obtain();
    }
    clientTransaction.setLifecycleStateRequest(lifecycleItem);
   
    // 通过ClientLifecycleManager来启动事务
    // 这里的mService就是AMS
    // 记住上面两个item：LaunchActivityItem和ResumeActivityItem，这是事务的执行单位
    // Schedule transaction.
    mService.getLifecycleManager().scheduleTransaction(clientTransaction);
}
```

> 通过 AMS 获取 ClientLifeCycleManager
>
> ```java
> /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java/;
> //通过AMS获取ClientLifecycleManager
> ClientLifecycleManager getLifecycleManager() {
> return mLifecycleManager;
> }
> ```

6、ClientLifeCycleManager 是事务管理类，负责执行事务

```java
/frameworks/base/services/core/java/com/android/server/am/ClientLifecycleManager.java
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    final IApplicationThread client = transaction.getClient();
    //执行事务
    transaction.schedule();
    if (!(client instanceof Binder)) {
        transaction.recycle();
    }
}
```

7、把事务交给本地 ActivityThread 执行，这里通过本地 ApplicationThread 在服务端的接口 IApplicationThread 来进行跨进程通信，后面逻辑就回到了应用程序进程了。

```java
/frameworks/base/core/java/android/app/servertransaction/ClientTransaction.java/;
   
//这里的IApplicationThread是要启动进程的IBinder接口
//ApplicationThread是ActivityThread的内部类，IApplicationThread是IBinder代理接口
//这里将逻辑转到本地来执行
private IApplicationThread mClient;
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```

#### ActivityThread 创建 Activity 的过程

##### 流程图

![](/picture/data-13.image)

##### 源码

1. IApplicationThread 接口的本地实现类 ActivityThread 的内部类ApplicationThread

```java
/frameworks/base/core/java/android/app/ActivityThread.java/ApplicationThread.class/;
//跳转到ActivityThread的方法实现
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}
```

2、ActivityThread 执行事务。ActivityThread 是继承 ClientTransactionHandler，scheduleTransaction 的具体实现是在 ClientTransactionHandler 实现的，这里的主要内容是把事务发送给 ActivityThread 的内部类 H 执行。H 是一个 Handle，这个Handle 来切换到主线程执行逻辑。

```java
/frameworks/base/core/java/android/app/ClientTransactionHandler.java
void scheduleTransaction(ClientTransaction transaction) {
    //事务预处理
    transaction.preExecute(this);
    //这里很明显可以利用Handle机制切换线程，下面看看这个方法的实现
    //该方法的具体实现是在ActivityThread，是ClientTransactionHandler的抽象方法
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
   
/frameworks/base/core/java/android/app/ActivityThread.java/;
final H mH = new H();
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    if (DEBUG_MESSAGES) Slog.v(
        TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
        + ": " + arg1 + " / " + obj);
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    //利用Handle进行切换。mH是H这个类的实例
    mH.sendMessage(msg);
}
```

3、H 对事务进行处理，调用事务池来处理事务

```java
/frameworks/base/core/java/android/app/ActivityThread.java/H.class
//调用事务池对事务进行处理
public void handleMessage(Message msg) {
    if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
    switch (msg.what) {
        ...
        case EXECUTE_TRANSACTION:
            final ClientTransaction transaction = (ClientTransaction) msg.obj;
            //调用事务池对事务进行处理
            mTransactionExecutor.execute(transaction);
            if (isSystem()) {
                transaction.recycle();
            }
            // TODO(lifecycler): Recycle locally scheduled transactions.
            break;
            ...
    }
    ...
}
```

4、事务池对事务进行处理，事务池会把事务池中的两个 Item 拿出来分别执行，这两个事务是上面讲的两个 Item，对应不同的初始化工作。

```java
/frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java
       
public void execute(ClientTransaction transaction) {
    final IBinder token = transaction.getActivityToken();
    log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);
   
    //执行事务
    //这两个事务就是当时在ActivityStackSupervisor中添加的两个事件（第8步）
    //注释1执行activity的创建，注释2执行activity的窗口等等并调用onStart和onResume方法
    //后面主要深入注释1的流程
    executeCallbacks(transaction);//1
    executeLifecycleState(transaction);//2
    mPendingActions.clear();
    log("End resolving transaction");
}
   
public void executeCallbacks(ClientTransaction transaction) {
    ...
        //执行事务
        //这里的item就是当初添加的Item，还记得是哪个吗？
       	// 对了就是LaunchActivityItem
        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
    ...
}
   
private void executeLifecycleState(ClientTransaction transaction) {
    ...
    // 和上面的一样，执行事务中的item，item类型是ResumeActivityItem
    lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
    lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
}
```

5、LaunchActivityItem 调用 ActivityThread 执行创建逻辑

```java
/frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java/;
   
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client);
    // ClientTransactionHandler是ActivityThread实现的接口，具体逻辑回到ActivityThread
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

6、ActivityThread 执行 Activity 的创建。只要利用 Instrumentation 来创建 activity和回调activity 的生命周期，并创建 activity 的上下文和 APP 上下文：

```java
/frameworks/base/core/java/android/app/ActivityThread.java/;
   
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ...
        // 跳转到performLaunchActivity
        final Activity a = performLaunchActivity(r, customIntent);
    ...
}
   
//使用Instrumentation去创建activity回调生命周期
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    	//获取ActivityInfo，用户存储代码、AndroidManifes信息。
    	ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            //获取apk描述类
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
       
    	// 获取activity的包名类型信息
    	ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }
    ...
        // 创建context上下文
        ContextImpl appContext = createBaseContextForActivity(r);
   		// 创建activity
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            // 通过Instrumentation来创建活动
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        }
    ...
        try {
            // 根据包名创建Application，如果已经创建则不会重复创建
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            ...
            // 为Activity添加window
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            appContext.setOuterContext(activity);
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);
            }
	...
        // 通过Instrumentation回调Activity的onCreate方法
        ctivity.mCalled = false;
        if (r.isPersistable()) {
            mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
            mInstrumentation.callActivityOnCreate(activity, r.state);
        }
}    
```

> 这里深入看一下 onCreate 什么时候会被调用
>
> ```java
> /frameworks/base/core/java/android/app/Instrumentation.java/;
> public void callActivityOnCreate(Activity activity, Bundle icicle,
>   PersistableBundle persistentState) {
> prePerformCreate(activity);
> // 调用了activity的performCreate方法
> activity.performCreate(icicle, persistentState);
> postPerformCreate(activity);
> }
> 
> /frameworks/base/core/java/android/app/Activity.java/;
> final void performCreate(Bundle icicle, PersistableBundle persistentState) {
> mCanEnterPictureInPicture = true;
> restoreHasCurrentPermissionRequest(icicle);
> // 这里就回调了onCreate方法了
> if (persistentState != null) {
>   onCreate(icicle, persistentState);
> } else {
>   onCreate(icicle);
> }
> ...
> }
> ```

7、Instrumentation 通过类加载器来创建 Activity 实例

```java
/frameworks/base/core/java/android/app/Instrumentation.java/;
   
public Activity newActivity(ClassLoader cl, String className,
        Intent intent)
        throws InstantiationException, IllegalAccessException,
        ClassNotFoundException {
    String pkg = intent != null && intent.getComponent() != null
            ? intent.getComponent().getPackageName() : null;
    // 利用AppComponentFactory进行实例化
    return getFactory(pkg).instantiateActivity(cl, className, intent);
}
```

8、最后一步，通过 AppComponentFactory 工厂创建实例

























