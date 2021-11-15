# Activity 的启动流程详解

Android 系统对四大组件都做了很大程度的封装，这样我们可以快速使用组件。Activity 的启动在系统封装后，变的极为简单，显式启动 activity 代码如下：

```java
Intent intent = new Intent(this, TestActivity.class);
this.startActivity(intent);
```

这样就可以启动 activity 了，那么问题来了，

- 这个代码是如何启动一个 activity 的？
- 里面做了什么事情？
- activity 生命周期是如何执行的？
- activity 对象是什么时候创建的？
- 视图是如何处理以及何时可见的？

### 流程分析

#### Activity 启动的发起

下面对 activity 的工作流程进行梳理，达到对 activity 整体流程的掌握。从 startActivity 方法开始，会走到 **startActivityForResult** 方法：

```javascript
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {           
           options = transferSpringboardActivityOptions(options);
           Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                   this, mMainThread.getApplicationThread(), mToken, this,
                  intent, requestCode, options);
           if (ar != null) {
              mMainThread.sendActivityResult(                   
                   mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                   ar.getResultData());
            }
            if (requestCode >= 0) {
                mStartedActivity = true;
           }
           cancelInputsAndStartExitTransition(options);        
        } else {
            ...
        }
    }
```

看到里面调用了 mInstructation.execStartActivity 方法，其中一个参数是 mMainThread.getApplicationThread ,它的类型是 **ApplicationThread**, **ApplicationThread**是 **ActivityThread** 的内部类，继承 IApplication.**Stub**,也是个 Binder 对象，在 activity工作流程中有重要作用，而 Instrumentation 具有跟踪 application 和 activity 生命周期的功能，用于 android 应用测试框架中代码检测，接下来看 mInstrumentation.execStartActivity 方法：

```javascript
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        ...

        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityTaskManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

这里看到 Activity 的启动又交给了 ActivityTaskManager.getService, 这是啥？跟进去看看：

```javascript
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        ...
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityTaskManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

看到 IBinder 这个标志，这里是获取一个跨进程的服务，获取的是什么服务呢? 是 **ActivityTaskManagerService(ATMS)**, 它继承于 IActivityTaskManager.stub, 是个 binder 对象，并且是通过单例提供服务的。ATMS 是用于管理 Activity及其容器（任务、堆栈、显示等）的系统服务，运行在系统服务进程（system_server) 之中。

> 值得说明的是，ATMS 是在 Android 10 中新增的，分担了之前 ActivityManagerService(AMS) 的一部分共呢个（activity task 相关）。在 Android10 之前，这个地方获取的服务是 AMS。

接着看，ActivityTaskManager.getService().startActivity 有个返回值 result，且调用了 checkStartActivityResult(result, intent):

```javascript
    public static void checkStartActivityResult(int res, Object intent) {
        if (!ActivityManager.isStartResultFatalError(res)) {
            return;
        }

        switch (res) {
            case ActivityManager.START_INTENT_NOT_RESOLVED:
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                            "Unable to find explicit activity class "
                            + ((Intent)intent).getComponent().toShortString()
                            + "; have you declared this activity in your AndroidManifest.xml?");
                throw new ActivityNotFoundException(
                        "No Activity found to handle " + intent);
            case ActivityManager.START_PERMISSION_DENIED:
                throw new SecurityException("Not allowed to start activity "
                        + intent);
            ...
            
            case ActivityManager.START_CANCELED:
                throw new AndroidRuntimeException("Activity could not be started for "
                        + intent);
            default:
                throw new AndroidRuntimeException("Unknown error code "
                        + res + " when starting " + intent);
        }
    }
```

这是用来检查 activity 启动的结果，如果发生致命错误，就会抛出异常。

### 2.2 Activity 的管理---ATMS

好了，到这里，Activity 的启动就跨进程 （IPC）的转移到系统进程提供的服务 ATMS 中了，接着看 ATMS 的 startActivity:

```java
//ActivityTaskManagerService
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
    
    @Override
    public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }

    int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivityAsUser");

        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
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

跟到 startActivityUser 中，通过 getActivityStartController().obtainStart 方法获取 ActivityStarter 实例，然后调用一系列方法，最后的 execute() f方法是开始启动 activity：

```javascript
    int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                        mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            }
        } finally {
            onExecutionComplete();
        }
    }
```

分了两种情况，不过，不论 startActivityMayWait 还是 startActivity 最终都是走到下面这个 startActiivty 方法：

```JavaScript
    private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity, boolean restrictedBgActivity) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity, restrictedBgActivity);
        } finally {
            final ActivityStack currentStack = r.getActivityStack();
            startedActivityStack = currentStack != null ? currentStack : mTargetStack;

           ...
        }

        postStartActivityProcessing(r, result, startedActivityStack);
        return result;
    }
```

里面调用了 **startActivityUnchecked**方法，之后调用 RootActivityContainer 的 resumeFoucusedStacksTopActivities 方法。接着跳到了 ActivityStack 的 resumeTopActivityUnCheckedLocked 方法：

```javascript
//ActivityStack
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }
        boolean result = false;
        try {
            mInResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);

            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
    }
```

跟进 resumeTopActivityInnerLocked 方法：

```JavaScript
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        ...
        boolean pausing = getDisplay().pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
             // 暂停上一个Activity
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
        ...
        //这里next.attachedToProcess()，只有启动了的Activity才会返回true
        if (next.attachedToProcess()) {
            ...
            
            try {
                final ClientTransaction transaction =
                        ClientTransaction.obtain(next.app.getThread(), next.appToken);
                ...
                //启动了的Activity就发送ResumeActivityItem事务给客户端了，后面会讲到
                transaction.setLifecycleStateRequest(
                        ResumeActivityItem.obtain(next.app.getReportedProcState(),
                                getDisplay().mDisplayContent.isNextTransitionForward()));
                mService.getLifecycleManager().scheduleTransaction(transaction);
               ....
            } catch (Exception e) {
                ....
                mStackSupervisor.startSpecificActivityLocked(next, true, false);
                return true;
            }
            ....
        } else {
            ....
            if (SHOW_APP_STARTING_PREVIEW) {
            	    //这里就是 冷启动时 出现白屏 的原因了：取根activity的主题背景 展示StartingWindow
                    next.showStartingWindow(null , false ,false);
                }
            // 继续当前Activity，普通activity的正常启动 关注这里即可
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
        return true;
    }
```

先对上一个 Activity 执行 pause 操作，再执行当前创建操作，代码最终进入到了 ActivityStackSupervisor.startActivityLocked 方法，这里有一个点，启动 activity 前调用了 next.showStartingWindow 方法来展示一个 window，这是 冷启动时出现白屏的原因了。继续看 **startSpecificActivityLocked** 方法。

```javascript
    void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            try {
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
            knownToBeDead = true;
        }

        ...
        
        try {
            if (Trace.isTagEnabled(TRACE_TAG_ACTIVITY_MANAGER)) {
                Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "dispatchingStartProcess:"
                        + r.processName);
            }
            // 上面的wpc != null && wpc.hasThread()不满足的话，说明没有进程，就会取创建进程
            final Message msg = PooledLambda.obtainMessage(
                    ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
                    r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
            mService.mH.sendMessage(msg);
        } finally {
            Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
        }
    }
```

有一个判断条件 wpc != null && wpc.hasThread()，意思是 是否启动了应用进程，内部是通过 IApplicationThread 是否为空来判断。这里我们只看已启动应用进程的情况，调用了 **realStartActivityLocked**：

```javascript
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
			...
                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);
                final DisplayContent dc = r.getDisplay().mDisplayContent;
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.icicle, r.persistentState, results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                                r.assistToken));
                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);
                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
                ...
        return true;
    }
```

中间有段代码如上，通过 ClientTransaction.obtain(...) 获取了 clientTransaction, 其中参数 proc.getThread() 是 IApplicationThread, 就是前面提到的 ApplicationThread 在系统进程的代理。

ClientTransaction 是包括一系列的待客户端处理的事务的容器，客户端接受后取出事务并执行。

接着看，使用 clientTransaction.addCallback 添加了 LaunchActivityItem 实例：

```javascript
	//都是用来发送到客户端的
	private List<ClientTransactionItem> mActivityCallbacks;
	
    public void addCallback(ClientTransactionItem activityCallback) {
        if (mActivityCallbacks == null) {
            mActivityCallbacks = new ArrayList<>();
        }
        mActivityCallbacks.add(activityCallback);
    }
```

看下 LaunchActivityItem 实例的获取：

```java
    /** Obtain an instance initialized with provided params. */
    public static LaunchActivityItem obtain(Intent intent, int ident, ActivityInfo info,
            Configuration curConfig, Configuration overrideConfig, CompatibilityInfo compatInfo,
            String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state,
            PersistableBundle persistentState, List<ResultInfo> pendingResults,
            List<ReferrerIntent> pendingNewIntents, boolean isForward, ProfilerInfo profilerInfo,
            IBinder assistToken) {
        LaunchActivityItem instance = ObjectPool.obtain(LaunchActivityItem.class);
        if (instance == null) {
            instance = new LaunchActivityItem();
        }
        setValues(instance, intent, ident, info, curConfig, overrideConfig, compatInfo, referrer,
                voiceInteractor, procState, state, persistentState, pendingResults,
                pendingNewIntents, isForward, profilerInfo, assistToken);

        return instance;
    }
```

new 了一个 LaunchActivityItem 然后设置各种值。从名字就能看出，它就是用来启动 activity 的。它是怎们发挥作用的呢？

回到 realStartActivityLocked 方法，接着调用了 mService.getLifecycleManager().scheduleTransaction(clientTransaction), mService 是 ActivityTaskManagerService, getLifecycleManager() 方法获取的是 ClientLifeCycleManager 实例，它的 scheduleTransaction 方法如下：

```javascript
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            transaction.recycle();
        }
    }
```

就是调用ClientTransaction的schedule方法，那就看看：

```java
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```
很简单，就是调用 IApplicationThread 的 scheduleTransaction 方法。由于 IApplicationThread 是 ApplicationThread 在系统进程的代理，所以真正执行的地方就是客户端的 ApplicationThread 中了，也就是说，Activity 启动的操作又跨进程的还给了客户端。

好了，这里梳理下：

> 启动 activity 的操作从客户端 跨进程 转移到 ATMS，ATMS 通过 ActivityStarted、ActivityStack、ActivityStackSupervisor 对 Activity 任务、activity 栈、 activity 记录管理后，又通过 跨进程 把启动过程 又转移到了客户端

#### 2.3 线程切换及消息处理---mH

接着上面的分析，我们找到 ApplicationThread 的 scheduleTransaction 方法：

```java
        @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

那就再看 ActivityThread 的 scheduleTransaction 方法，实际在其父类 ClientTransactionHandler 中：

```java
    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
```

使用 sendMessage 发送消息，参数是 ActivityThread.H.EXECUTE_TRANSACTION 和 transaction, 接着看：

```javascript
    void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }
    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) {
            Slog.v(TAG,
                    "SCHEDULE " + what + " " + mH.codeToString(what) + ": " + arg1 + " / " + obj);
        }
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
```

最后调用了 mH.sendMessage(msg) ，mH 是个啥？我们看看：

```java
//ActivityThread
final H mH = new H();
    class H extends Handler {
        public static final int BIND_APPLICATION        = 110;
        @UnsupportedAppUsage
        public static final int EXIT_APPLICATION        = 111;
        @UnsupportedAppUsage
        public static final int RECEIVER                = 113;
        @UnsupportedAppUsage
        public static final int CREATE_SERVICE          = 114;
        @UnsupportedAppUsage
        public static final int SERVICE_ARGS            = 115;
        ...
        public static final int EXECUTE_TRANSACTION = 159;
        public static final int RELAUNCH_ACTIVITY = 160;
        ...
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                case EXIT_APPLICATION:
                    if (mInitialApplication != null) {
                        mInitialApplication.onTerminate();
                    }
                    Looper.myLooper().quit();
                    break;
                case RECEIVER:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveComp");
                    handleReceiver((ReceiverData)msg.obj);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                case CREATE_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
                    handleCreateService((CreateServiceData)msg.obj);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                case BIND_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
                    handleBindService((BindServiceData)msg.obj);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                ...
                
                case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // Client transactions inside system process are recycled on the client side
                        // instead of ClientLifecycleManager to avoid being cleared before this
                        // message is handled.
                        transaction.recycle();
                    }
                    break;
                case RELAUNCH_ACTIVITY:
                    handleRelaunchActivityLocally((IBinder) msg.obj);
                    break;
                ...
            }
            Object obj = msg.obj;
            if (obj instanceof SomeArgs) {
                ((SomeArgs) obj).recycle();
            }
            if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));
        }
    }
```

mH 是在创建 activityThread 实例时赋值的，是自定义 Handler 子类 H 的实例，也就是在 ActivityThread 的 main 方法中，并且初始化时，主线程已经有了 mainLooper, 所以，使用这个mH 来 sendMessage 就把消息发送到了主线程。

那么是从哪个线程发送的呢？那就要看看 ApplicationThread 的 scheduleTransaction 方法是执行在哪个线程了。我们知道，服务器的 binder 方法运行在 Binder 的线程池中，也就是说系统进行跨进程调用 ApplicationThread 的 scheduleTransaction 就是执行在 binder 的线程池中了。

那这里，消息就在主线程处理了，那么是怎么处理 Activity 的启动的呢？接着看，我们找到 ActivityThread.H.EXECUTE_TRANSACTION 这个消息的处理，就在 handleMessage 方法的倒数第三个 case：取出 ClientTransaction 实例，调用 TransactionExecutor 的 execute 方法，那就看看：

```java
    public void execute(ClientTransaction transaction) {
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");
        final IBinder token = transaction.getActivityToken();
        ...
        executeCallbacks(transaction);
        executeLifecycleState(transaction);
        ...
    }
```

继续跟进 executeCallbacks 方法：

```java
    public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        if (callbacks == null || callbacks.isEmpty()) {
            // No callbacks to execute, return early.
            return;
        }
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Resolving callbacks in transaction");

        final IBinder token = transaction.getActivityToken();
        ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

        // In case when post-execution state of the last callback matches the final state requested
        // for the activity in this transaction, we won't do the last transition here and do it when
        // moving to final state instead (because it may contain additional parameters from server).
        final ActivityLifecycleItem finalStateRequest = transaction.getLifecycleStateRequest();
        final int finalState = finalStateRequest != null ? finalStateRequest.getTargetState()
                : UNDEFINED;
        // Index of the last callback that requests some post-execution state.
        final int lastCallbackRequestingState = lastCallbackRequestingState(transaction);

        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            final ClientTransactionItem item = callbacks.get(i);
            ...
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            ...
        }
    }
```

遍历callbacks，调用ClientTransactionItem的execute方法，而我们这里要关注的是ClientTransactionItem的子类LaunchActivityItem，看下它的execute方法：

```java
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```

里面调用了 client.handleLaunchActivity 方法，client 是 ClientTransactionHandler 的实例，是在 TransactionExecutor 构造方法传入的，TransactionExecutor 创建是在 ActivityThread 中：

```java
//ActivityThread
private final TransactionExecutor mTransactionExecutor = new TransactionExecutor(this);
```

所以，client.handleLaunchActivity 方法就是 ActivityThread 的 handleLaunchActivity 方法。

好了，到这里 **ApplicationThread** 把 启动 Activity 的操作，通过 mH 切到了主线程，走到了 ActivityThread 的 handleLaunchActivity 方法。

#### 2.4 Activity 启动核心实现---初始化及生命周期

那就接着看：

```java
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        ...
        final Activity a = performLaunchActivity(r, customIntent);
        ...
        return a;
    }
```

继续跟 **performLaunchActivity** 方法，这里就是 activity 启动的核心实现了：

```java
    /**  activity 启动的核心实现. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    	//1、从ActivityClientRecord获取待启动的Activity的组件信息
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
		//创建ContextImpl对象
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
        	//2、创建activity实例
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            ..
        }
        try {
        	//3、创建Application对象（如果没有的话）
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            ...
            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
              
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                
                //4、attach方法为activity关联上下文环境
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);
                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }
                activity.mCalled = false;
                //5、调用生命周期onCreate
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
            }
            r.setState(ON_CREATE);
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }

        } 
        ...
        return activity;
    }
```

PerformLaunchActivity 主要完成以下事情：

- 从 ActivityClientRecord 获取待启动的 Activity 组件信息
- 通过 mInstrumentation.newActivity 方法使用类加载器创建 activity 实例
- 使用 LoadedApk 的 makeApplciation 方法创建 application 对象，内部也是通过 mInstrumentation 使用类加载器，创建后就调用了 instrumentation.callApplicationOnCreate 方法，也就是 Application 的 onCreate 方法。
- 创建 contextImpl 对象并通过 attach 方法对重要数据进行初始化，关联了 context 的具体实现 contextImpl ,attach 方法内部还完成了 window 的创建，这样 window 接受到外部事件后就能传递给 Activity 了。
- 调用 activity 的 onCreate 方法，是通过 mInstrumentation.callActivityOnCreate 方法完成。

到这里 Activity 的 onCREATE 方法执行完成，那么 onStart、onResume 呢？

上面看到有 LaunchActivityItem, 是用来启动 Activity 的，也就是走到 Activity 的 onCreate，那么是不是有“XXXXActivityItem" 呢？有的：

- LaunchActivityItem 远程App端的onCreate生命周期事务
- ResumeActivityItem 远程App端的onResume生命周期事务
- PauseActivityItem 远程App端的onPause生命周期事务
- StopActivityItem 远程App端的onStop生命周期事务
- DestroyActivityItem 远程App端onDestroy生命周期事务

另外梳理过程中涉及的几个类：

- ClientTransaction 客户端事务控制者
- ClientLifeCycleManager 客户端的生命周期事务执行者
- TransactionExecutor 远程通信事务执行者

那么我们看下 ResumeActivityItem 吧

我们看下 ActivityStackSupervisor 的 realStartActivityLocked 方法：

```javascript
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
			...
                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);

                final DisplayContent dc = r.getDisplay().mDisplayContent;
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.icicle, r.persistentState, results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                                r.assistToken));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                //这里ResumeActivityItem
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
                ...
        return true;
    }
```

前面只说了通过clientTransaction.addCallback添加LaunchActivityItem实例，在注意下面接着调用了clientTransaction.setLifecycleStateRequest(lifecycleItem)方法，lifecycleItem是ResumeActivityItem或PauseActivityItem实例，这里我们关注ResumeActivityItem，先看下setLifecycleStateRequest方法：

```javascript
    /**
     * Final lifecycle state in which the client activity should be after the transaction is
     * executed.
     */
	private ActivityLifecycleItem mLifecycleStateRequest;
	
    public void setLifecycleStateRequest(ActivityLifecycleItem stateRequest) {
        mLifecycleStateRequest = stateRequest;
    }
```

mLifecycleStateRequest 表示执行 transaction 后的最终的生命周期状态。

继续看处理 ActivityThread.H.EXECUTE_TRANSACTION 这个消息的处理，即 TransactionExecutor 的 execute 方法：

```java
    public void execute(ClientTransaction transaction) {
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");

        final IBinder token = transaction.getActivityToken();
        ...
        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        ...
    }
```

前面我们关注的是 executeCallbacks 方法，现在看看 executeLifecycleState 方法：

```javascript
    /** Transition to the final state if requested by the transaction. */
    private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        if (lifecycleItem == null) {
            // No lifecycle request, return early.
            return;
        }

        final IBinder token = transaction.getActivityToken();
        final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
        ...

        // Execute the final transition with proper parameters.
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
```

这里取出了 ActivityLifecycleItem 并且调用了它的 execute 方法，实际就是 ResumeActivityItem 的方法：

```javascript
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```

经过上面的分析实际就是走到 ActivityThread 的 handleResumeActivity 方法：

```javascript
    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ...
        // performResumeActivity内部会走onStart、onResume
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        if (r == null) {
            // We didn't actually resume the activity, so skipping any follow-up actions.
            return;
        }
        ...
        
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            ...
            
        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            if (r.newConfig != null) {
                performConfigurationChangedForActivity(r, r.newConfig);
                if (DEBUG_CONFIGURATION) {
                    Slog.v(TAG, "Resuming activity " + r.activityInfo.name + " with newConfig "
                            + r.activity.mCurrentConfig);
                }
                r.newConfig = null;
            }
            if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
            WindowManager.LayoutParams l = r.window.getAttributes();
            if ((l.softInputMode
                    & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                    != forwardBit) {
                l.softInputMode = (l.softInputMode
                        & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                        | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm = a.getWindowManager();
                    View decor = r.window.getDecorView();
                    wm.updateViewLayout(decor, l);
                }
            }

            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
            	//添加window、设置可见
                r.activity.makeVisible();
            }
        }

        r.nextIdle = mNewActivities;
        mNewActivities = r;
        if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
        Looper.myQueue().addIdleHandler(new Idler());
    }
```

handleResumeActivity 主要做了以下事情：

- 调用生命周期：通过 performResumeActivity 方法，内部调用生命周期 onStart、onResume 方法
- 设置视图可：通过 activity.makeVIsible 方法，添加 window、设置可见

先看第一点，生命周期的调用。即 performResumeActivity 方法：

```
    public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
            String reason) {
        final ActivityClientRecord r = mActivities.get(token);
        ...
        try {
            ...
            r.activity.performResume(r.startsNotResumed, reason);
            ...
        } 
        ...
        return r;
    }
```

调用了 activity.perormResume 方法：

```javascript
    final void performResume(boolean followedByPause, String reason) {
        dispatchActivityPreResumed();
        //内部会走onStart
        performRestart(true /* start */, reason);
        ...
        // 走onResume
        mInstrumentation.callActivityOnResume(this);
        ...
		//这里是走fragment的onResume
        mFragments.dispatchResume();
        mFragments.execPendingActions();
        ...
    }
```

先调用了 performRestart, performRestart 又会调用 performStart, 其内部调用了 mInstrucmentation.callActivityOnStart(this), 也就是 Activity 的 onStart 方法了。然后是 mInstrumentation.callActivityOnResume，也就是 Activity 的 onResume 方法了，到这里启动后的生命周期走完了。

再看第二点，设置视图可见，即 Activity.makeVisible 方法：

```javascript
//Activity
    void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```

这里把 activity 的顶级布局 mDecor 通过 windowManager.addView 方法，把视图添加到 window, 并设置 mDecor 可见，到这里视图是真正的可见了；值得注意的是，视图的真正可见是在 onResume 方法之后的。

另外一点， Activity 视图渲染到 window 后，会设置 window 焦点变化，先走到 DecorView 的 onWindowFoucsChanged 方法，最后是到 Activity 的 onWindowFoucsChanged 方法，表示首帧绘制完成，此时 Activity 可交互。

好了，到这里就是真正的创建完成并且可见了。

梳理成关系图如下：

![](/picture/fragment07.png)

设涉及到的类如下：

| 类名                                   | 作用                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| ActivityThread                         | 应用的入口类，系统通过调用main函数，开启消息循环队列。ActivityThread所在线程被称为应用的主线程（UI线程） |
| ApplicationThread                      | 是ActivityThread的内部类，继承IApplicationThread.Stub，是一个IBinder，是ActiivtyThread和AMS通信的桥梁，AMS则通过代理调用此App进程的本地方法，运行在Binder线程池 |
| H                                      | 继承Handler，在ActivityThread中初始化，即主线程Handler，用于主线程所有消息的处理。本片中主要用于把消息从Binder线程池切换到主线程 |
| Intrumentation                         | 具有跟踪application及activity生命周期的功能，用于监控app和系统的交互 |
| ActivityManagerService                 | Android中最核心的服务之一，负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块相类似，因此它在Android中非常重要，它本身也是一个Binder的实现类。 |
| ActivityTaskManagerService             | 管理activity及其容器（task, stacks, displays）的系统服务（Android10中新增，分担了AMS的部分职责） |
| ActivityStarter                        | 用于解释如何启动活动。该类收集所有逻辑，用于确定Intent和flag应如何转换为活动以及相关的任务和堆栈 |
| ActivityStack                          | 用来管理系统所有的Activity，内部维护了Activity的所有状态和Activity相关的列表等数据 |
| ActivityStackSupervisor                | 负责所有Activity栈的管理。AMS的stack管理主要有三个类，ActivityStackSupervisor，ActivityStack和TaskRecord |
| ClientLifecycleManager                 | 客户端生命周期执行请求管理                                   |
| ClientTransaction                      | 是包含一系列的 待客户端处理的事务 的容器，客户端接收后取出事务并执行 |
| LaunchActivityItem、ResumeActivityItem | 继承ClientTransactionItem，客户端要执行的事务信息，启动activity |

以上就是一个 **普通 Activity** 启动的完整流程。

为啥我说“普通 activity” 呢？因为你会发现，整个流程是从 startActivity 方法开始的，是我们在一个 Activity 里面启动另外一个 activity 的情况。那么一个 app 的根 Activity 是如何启动的呢？

另外还注意到，在 ActivityStackSupervisor 的 startSpecificAcitivityLocked 方法中，上面分析有提到，有个判段条件 if(wpc != null && wpc.hasThread()), 意思是 是否启动的应用进程，而我们只分析了已启动应用进程的情况，那么未启动应用进程的情况就是 根 Activity 的启动了。下面来分析 应用程序的启动过程。

# 三、根 Activity 的启动----应用进程启动

我们知道，想要启动一个应用程序（app)，需要点击手机桌面的应用图标。Android 系统的桌面叫做 Launcher，有以下作用：

- 作为 Android 系统的启动器，用于启动应用程序
- 作为 Android 系统的桌面，用于显示和管理显示应用程序的快捷图标和其他桌面组件

Launcher 本身也是一个应用程序，它在启动的过程中会请求 PackagerManagerService 返回系统中已经安装的 app 的信息，并将其快捷图标显示在桌面程序上，用户可以点击图标启动 app。

### 3.1 应用进程的创建

当点击 app 图标后，Launcher 会在桌面 activity 内调用（此 activity 就叫做 Launcher）startActivitySafely 方法，startActivitySafely 方法会调用 startActivity 方法，接下来的部分就和上面分析的 Activity 启动的发起过程一致了，即通过 IPC 走到了 ATMS，直到 ActivityStackSupervisor 的 startSpecificActiviyLocked 方法中对应用进程是否存在的判断。一起看下：

```javascript
    void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            try {
            	//有应用进程就启动activity
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
            knownToBeDead = true;
        }

        ...
        
        try {
            if (Trace.isTagEnabled(TRACE_TAG_ACTIVITY_MANAGER)) {
                Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "dispatchingStartProcess:"
                        + r.processName);
            }
            // Post message to start process to avoid possible deadlock of calling into AMS with the
            // ATMS lock held.
            // 上面的wpc != null && wpc.hasThread()不满足的话，说明没有进程，就会去创建进程
            final Message msg = PooledLambda.obtainMessage(
                    ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
                    r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
            mService.mH.sendMessage(msg);
        } finally {
            Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
        }
    }
```

逻辑很清晰：有应用进程就启动 activity（普通 activity），没有就去创建进程（然后再启动根 Activity）。

应用进程存在的判断条件是：wpc != null && wpc.hasThread()，看下 WindowProcessController 的 hasThread 方法：

```javascript
    // The actual proc...  may be null only if 'persistent' is true (in which case we are in the
    // process of launching the app)
    private IApplicationThread mThread;
    
    boolean hasThread() {
        return mThread != null;
    }
```

前面已有说明，IApplicationThread 是 ApplicationThread 客户端（app）在服务端（系统进程）的代理，这里判断 **IApplicationThread 不为空，就代表进程已存在**，为啥这么判断呢？这里先立个 flag，进程创建之后，一定会有给 IApplicationThread 赋值的操作，这个就符合这个逻辑了。我们继续看看进程是如何创建的，以及创建之后是否有给 IApplicationThread 赋值的操作。

使用 ActivityTaskManagerService 的 mH（继承 handler) 发送了一个消息，消息中第一个参数是 ActivityManagerInternal:startProcess, :eggplant:ActivityManagerInternal 的实现是 AMS 的内部类 LocalService, LocalService 的 startProcess 方法调用了 AMS 的 startProcessLocked 方法，那么我们就看看 AMS 的 startProcessLocked 方法，这里应该就是创建进程了：

```javascript
    final ProcessRecord startProcessLocked(String processName,
            ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            HostingRecord hostingRecord, boolean allowWhileBooting,
            boolean isolated, boolean keepIfLarge) {
        return mProcessList.startProcessLocked(processName, info, knownToBeDead, intentFlags,
                hostingRecord, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
                null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
                null /* crashHandler */);
    }
```

这里调用了 ProcessList.startProcessLocked 方法，内部又多次调用了 startProcessLocked  不同的重载方法，最后走到 startProcess 方法：

```javascript
    private Process.ProcessStartResult startProcess(HostingRecord hostingRecord, String entryPoint,
            ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
            String seInfo, String requiredAbi, String instructionSet, String invokeWith,
            long startTime) {
        try {
        ...
               startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, invokeWith, app.info.packageName,
                        new String[] {PROC_START_SEQ_IDENT + app.startSeq});
            }
            checkSlow(startTime, "startProcess: returned from zygote!");
            return startResult;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        }
    }
```

调用了 Process.start 方法，跟进看下：

```javascript
    public static ProcessStartResult start(@NonNull final String processClass,
                                           @Nullable final String niceName,
                                           int uid, int gid, @Nullable int[] gids,
                                           int runtimeFlags,
                                           int mountExternal,
                                           int targetSdkVersion,
                                           @Nullable String seInfo,
                                           @NonNull String abi,
                                           @Nullable String instructionSet,
                                           @Nullable String appDataDir,
                                           @Nullable String invokeWith,
                                           @Nullable String packageName,
                                           @Nullable String[] zygoteArgs) {
        return ZYGOTE_PROCESS.start(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, packageName,
                    /*useUsapPool=*/ true, zygoteArgs);
    }
```

ZYGOTE_PROCESS 是用于保持与 Zygote 进程的通信状态，发送 socket 请求与 Zygote 进程通信。Zygote 进程是 **进程孵化器**，用于创建进程。简单介绍下：

- Zygote 通过 fork 创建了一个进程
- 在新建的进程中创建 Binder 线程池（此进程支持了 Binder IPC）
- 最终通过反射获取到了 ActivityThread 类并执行了 main 方法

#### 3.2 根 Activity 的启动

ActivityThread 的 main 方法，主要就是开启了主线程的消息循环。

```javascript
    final H mH = new H();
    
    public static void main(String[] args) {
        ...
        //1、准备主线程的Looper
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
        //这里实例化ActivityThread，也就实例化了上面的mH，就是handler。
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

		//获取handler
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        ...
        //主线程looper开启
        Looper.loop();
		//因为主线程的Looper是不能退出的，退出就无法接受事件了。一旦意外退出，会抛出异常
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

在这里，我们关注 这两行代码：

```java
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);
```

创建ActivityThread实例，同时会创建ApplicationThread实例，ApplicationThread实例是ActivityThread实例的属性。然后调用了attach方法：

```java
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
            	//把ApplicationThread实例关联到AMS中
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            ...
        } 
        ...
    }
```

前面提到过这里mgr就是AMS在客户端的代理，所以mgr的attachApplication方法，就是IPC的走到AMS的attachApplication方法了：

```javascript
    public final void attachApplication(IApplicationThread thread, long startSeq) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

attachApplicationLocked方法很长，这里保留重要的几点：

```javascript
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {

			...
				//1、IPC操作，创建绑定Application
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions);
			...
            // 2、赋值IApplicationThread
            app.makeActive(thread, mProcessStats);
			...
			
        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
            	//3、通过ATMS启动 根activity
                didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
		...
}
```

AMS 的 attachApplicationLocked 方法主要三件事：

- 通过 IApplciationThread 的 bindApplciation 方法，IPC 操作，创建绑定 application
- 通过 makeActive 赋值 IApplicationThread，即验证了上面的猜测
- 通过ATMS 启动 根 Activity

先看 makeActive 方法：

```java
    public void makeActive(IApplicationThread _thread, ProcessStatsService tracker) {
        ...
        thread = _thread;
        mWindowProcessController.setThread(thread);
    }
```

看到使用mWindowProcessController.setThread(thread)确实完成了IApplicationThread的赋值。这样就可以依据IApplicationThread是否为空判断进程是否存在了。

再看创建绑定 Application 的过程：IApplicationThread 的 bindApplication 方法实现是客户端的 ApplicationThread 的 bindApplication 方法，它又使用 H 转移到了 ActivityThread 的 handleBindApplication 方法（从 binder 线程池转移到了主线程），看下 handleBindApplciation  方法：

```javascript
private void handleBindApplication(AppBindData data) {
	...
	            final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                    appContext.getClassLoader(), false, true, false);
            final ContextImpl instrContext = ContextImpl.createAppContext(this, pi,
                    appContext.getOpPackageName());
            try {
            	//创建Instrumentation
                final ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            } 
            ...
            final ComponentName component = new ComponentName(ii.packageName, ii.name);
            mInstrumentation.init(this, instrContext, appContext, component,
                    data.instrumentationWatcher, data.instrumentationUiAutomationConnection);
	...
			//创建Application
            app = data.info.makeApplication(data.restrictedBackupMode, null);
	...
            mInitialApplication = app;
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
    ...
            try {
            	//内部调用Application的onCreate方法
                mInstrumentation.callApplicationOnCreate(app);
            }
	...
}
```

主要就是创建 Application，并且调用生命周期 onCreate 方法。也就是说。正常情况下 Application 的初始化是在 handleBindApplication 方法中的，并且是在创建进程之后。performLaunchActivity 中只是做了一个检测，异常情况 Application 不存在的情况会创建。

这里注意一点，创建 Application 后，内部会调用 attach 方法，attach 内部会调用 attachBaseContext 方法，attachBaseContext 方法是我们能接触到的一个方法，接着才是 onCreate 方法。

再来看 根 activity 的启动，回到上面 AMS 的 attachApplicationLocked 方法，调用了 mAtmInternal.attachApplication 方法，mAtmInternal 是 ActivityTaskManagerInternal 实例，具体实现是在 ActivityTaskManagerService 的内部类 LocalService, 去看看：

```javascript
//ActivityTaskManagerService#LocalService
        public boolean attachApplication(WindowProcessController wpc) throws RemoteException {
            synchronized (mGlobalLockWithoutBoost) {
                return mRootActivityContainer.attachApplication(wpc);
            }
        }
```

mRootActivityContainer 是 RootActivityContainer 实例，看下它的 attachApplication 方法：

```javascript
    boolean attachApplication(WindowProcessController app) throws RemoteException {
        final String processName = app.mName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ActivityDisplay display = mActivityDisplays.get(displayNdx);
            final ActivityStack stack = display.getFocusedStack();
            if (stack != null) {
                stack.getAllRunningVisibleActivitiesLocked(mTmpActivityList);
                final ActivityRecord top = stack.topRunningActivityLocked();
                final int size = mTmpActivityList.size();
                for (int i = 0; i < size; i++) {
                    final ActivityRecord activity = mTmpActivityList.get(i);
                    if (activity.app == null && app.mUid == activity.info.applicationInfo.uid
                            && processName.equals(activity.processName)) {
                        try {
                            if (mStackSupervisor.realStartActivityLocked(activity, app,
                                    top == activity /* andResume */, true /* checkConfig */)) {
                                didSomething = true;
                            }
                        } 
                        ...
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisible(null, 0, false /* preserve_windows */);
        }
        return didSomething;
    }
```

遍历 activity 栈，此时理论上应该只有一个根 activity，然后调用 mStackSupervisor.realStartActivityLocked 方法，看到这里我们直到了，这里开始走上面的流程了。

我们发现，**根activity的启动前 需要创建应用进程，然后走到ActivityThread的main方法，开启主线程循环，初始化并绑定Application、赋值IApplicationThread，最后真正的启动过程和普通Activity是一致的。**

![](/picture/fragment08.png)

# 总结

以上的流程就是这样，接下来分析：Window 编舞者 网络框架 compose view 等知识。