# Android 之 window 机制 token 验证

当我们想要在屏幕上展示一个 Dialog 的时候，我们可能会在 Activity 的 onCreate 方法里面这么写：

```java
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    val dialog = AlertDialog.Builder(this)
    dialog.run{
        title = "我是标题"
        setMessage("我是内容")
    }
    dialog.show()
}
```

它的构造需要一个 context 对象，但是这个 context 不能是 ApplicationContext 等其他 context，只能是 ActivityContext。这样的代码是没问题的，如果我们使用 Application 传入会怎么样呢？

```java
override fun onCreate(savedInstanceState: Bundle?) {
    ...
    // 注意这里换成了ApplicationContext
    val dialog = AlertDialog.Builder(applicationContext)
    ...
}
```

运行一下：

![](/picture/data-24.image)

报错了，原因是 `you need to use a Theme.AppCompat theme (or descendant(后代)) with this activity`,那我们给他添加一个 Theme：

```java
override fun onCreate(savedInstanceState: Bundle?) {
    ...
    // 注意这里添加了主题
    val dialog = AlertDialog.Builder(applicationContext,R.style.AppTheme)
    ...
}
```

好了再次运行：

![](/picture/data-25.image)

嗯嗯？又崩溃了，原因是：`unable to add view -- token null is not  valid; is you activity running? `token 为 null？这个 token 是什么？为什么同样是 context，使用 activity 没问题，用 ApplicationContext就出问题了？他们之间的区别是什么？这篇文章就围绕这个 token 来展开讨论一下。

## 什么是 token？

首先我们看到报错是在 `ViewRootImpl.java:907`，这个地方肯定有进行 token 判断，然后抛出异常，这样我们就能找到 token了，那我们直接去这个地方看看：

```java
ViewRootImpl.class(api29)
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ...
    int res;
    ...
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                        mTempInsets);
    ...
    if (res < WindowManagerGlobal.ADD_OKAY) {
        ...
        switch (res) {
            case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
            case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                /*
                *	1
                */
                throw new WindowManager.BadTokenException(
                    "Unable to add window -- token " + attrs.token
                    + " is not valid; is your activity running?");    
                ...
        }
        ...
    }
    ...
}
```

我们看到代码就是注释1的地方抛出了异常，是根据一个变量`res`来判断的，这个`res`来自方法 `addToDisplay`，那么 token 的判断肯定在这个方法里面了，`res`只是一个判断的结果，那么我们需要进入这个 `addToDisplay`里看一下。mWindowSession 的类型是 IWindowSession，他是一个接口，那他的实现类是什么？这里就涉及到 window 机制的相关内容：

> WindowManagerService 是系统服务进程，应用进程跟window 联系需要通过阔进程通信：AIDL，这里的 IWindowSession 只是一个 Binder 接口，他的具体实现类在系统服务进程的 Session 类。所以这里的逻辑就跳转到了 Session 类的 addToDisplay 方法中。

那我们继续到 Session 的方法中看一下：

```java
Session.class(api29)
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
   	final WindowManagerService mService; 
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
            Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
            InsetsState outInsetsState) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
                outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel,
                outInsetsState);
    }
}
```

可以看到，Session 确实是继承自接口 IWindowSession，因为 WMS 和 Session都是运行在系统进程，所以不需要阔进程通信，直接调用 WMS 方法：

```java
public int addWindow(Session session, IWindow client, int seq,
        LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
        Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
        InsetsState outInsetsState) {
   	...
    WindowState parentWindow = null;
    ...
	// 获取parentWindow
    parentWindow = windowForClientLocked(null, attrs.token, false);
    ...
    final boolean hasParent = parentWindow != null;
    // 获取token
    WindowToken token = displayContent.getWindowToken(
        hasParent ? parentWindow.mAttrs.token : attrs.token);
    ...
  	// 验证token
    if (token == null) {
    if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
          Slog.w(TAG_WM, "Attempted to add application window with unknown token "
                           + attrs.token + ".  Aborting.");
            return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
        }
       ...//各种验证
    }
    ...
}
```

WMS 的 addWindow方法代码这么多怎么找到关键代码？还记得 viewRootImpl 在判断 res 是什么值得情况下抛出异常吗？没错是` WindowManagerGlobal.ADD_BAD_APP_TOKEN WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN`,我们只需要找到其中一个就可以找到 token 的判断位置，从代码中可以看到，当 token == null 的时候，会进行各种判断，第一个返回的就是 `WindowManagerGlobal.ADD_BAD_APP_TOKEN `，这样我们就顺利找到 token 的类型：**WindowToken **。看一下这个类：

```java
class WindowToken extends WindowContainer<WindowState> {
    ...
    // The actual token.
    final IBinder token;
}
```

官方告诉我们这里的 token 变量才是真正的 token，而这个 token 是一个 IBinder 对象。

到这里 token 是什么已经清楚了：

> - token 是一个 IBinder 对象
> - 只有利用 token 才能成功添加 Dialog

那么在接下来就有更多的问题需要思考了：

- Dialog 在 show 过程中是如何拿到 token 并给到 WMS 验证的？
- 这个 token 在 activity 和 Application 两者之间有什么不同？
- WMS 怎么知道 token 是合法的，换句话说，WMS 是如何验证 token 的？

## dialog 如何获取到 context 的 token 的？

首先，我们解决第一个问题，Dialog 在show 的过程中是如何拿到 token 并给到 WMS 验证的？

我们知道导致两种 context 弹出 dialog 的不同结果，原因在于 token的问题。那么在弹出 Dialog 的过程中，他是如何拿到 context 的 token 并给到 WMS 验证的？源码内容很多，我们需要先看一下token 是封装在哪个参数被传输到了 WMS，确定了参数我们的搜索范围就减少了，我们回到 WMS 的代码：

```java
parentWindow = windowForClientLocked(null, attrs.token, false);
WindowToken token = displayContent.getWindowToken(
        hasParent ? parentWindow.mAttrs.token : attrs.token);
```

我们可以看到 token 和一个 `attrs.token`关系非常密切，而这个 attrs 从调用栈一路往回走到了 viewRootImpl 中：

```java
ViewRootImpl.class(api29)
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
   ...
}
```

可以看到这是一个 WindowManager.LayoutParams 类型的对象。那我们接下来需要从 show()开始，追踪这个 token 是如何被获取到的：

```java
Dialog.class(api30)
public void show() {
    ...
    WindowManager.LayoutParams l = mWindow.getAttributes();
    ...
    mWindowManager.addView(mDecor, l);
    ...
}
```

这里的 mWindow 和 mWindowManager 是什么？我们到 Dialog 的构造函数一看究竟：

```java
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    // 如果context没有主题，需要把context封装成ContextThemeWrapper
    if (createContextThemeWrapper) {
        if (themeResId == Resources.ID_NULL) {
            final TypedValue outValue = new TypedValue();
            context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
            themeResId = outValue.resourceId;
        }
        mContext = new ContextThemeWrapper(context, themeResId);
    } else {
        mContext = context;
    }
    // 初始化windowManager
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    // 初始化PhoneWindow
    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    ...
    // 把windowManager和PhoneWindow联系起来
    w.setWindowManager(mWindowManager, null, null);
    ...
}
```

初始化的逻辑我们看重点就好：首先判断这是不是个有主题的 context，如果不是需要设置主题并封装成一个 ContextThemeWrapper对象，这也是我们文章开始使用 applicationContext 对象会抛出异常的原因。然后获取 windowManager，注意，这里是重点。这里的 context 可能是 Activity 或者 Application，他们的 `getSystemService`返回的 WindowManager 是一样的吗，看代码：

```java
Activity.class(api29)
public Object getSystemService(@ServiceName @NonNull String name) {
    if (getBaseContext() == null) {
        throw new IllegalStateException(
                "System services not available to Activities before onCreate()");
    }
    if (WINDOW_SERVICE.equals(name)) {
        // 返回的是自身的WindowManager
        return mWindowManager;
    } else if (SEARCH_SERVICE.equals(name)) {
        ensureSearchManager();
        return mSearchManager;
    }
    return super.getSystemService(name);
}

ContextImpl.class(api29)
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```

**Activity 返回的其实是自身的 WindowManager，而 Application 是调用 ContextImpl 的方法，返回的是应用服务 windowManager。**这两个有什么不同，暂时不知道，往下看，寻找答案。我们回到前面的方法，看到 `mWinodwManager.addView(mDecor,l)`;我们知道一个 PhoneWindow 对应一个 WindowManager，这里使用的 WindowManager 并不是 Dialog自己创建的 WindowManager，而是参数context 的 windowManager，也意味着并没有使用自己创建的 PhoneWindow。Dialog创建PhoneWindow 的目的是为了使用 DecorView 模板，我们可以看到 addView 的参数里并不是 window 而是 mDecor。

我们继续看代码，同时要注意这个 `l`参数。最终 token 就是封装在里面。`addView`方法最终会调用到了`WindowManagerGlobal`的`addView`方法。

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ...
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    }
	...
    ViewRootImpl root;
    ...
    root = new ViewRootImpl(view.getContext(), display);
	...
    try {
        root.setView(view, wparams, panelParentView);
    } 
    ...
}
```

这里我们只看 WindowManager.LayoutParams 参数，parentWindow 肯定不是 null，进入到`adjustLayoutParamsForSubWindow` 方法进行调整参数。最后调用 ViewRootImpl 的 setView 方法，到这里 WindowManager.LayoutParams 这个参数依旧没有被设置 token，那么最大的可能性就是在 `adjustLayoutParamsForSubWindow` 方法中了，马上进去看看：

```java
Window.class(api29)
void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
    CharSequence curTitle = wp.getTitle();
    if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
            wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
        // 子窗口token获取逻辑
        if (wp.token == null) {
            View decor = peekDecorView();
            if (decor != null) {
                wp.token = decor.getWindowToken();
            }
        }
        ...
    } else if (wp.type >= WindowManager.LayoutParams.FIRST_SYSTEM_WINDOW &&
                wp.type <= WindowManager.LayoutParams.LAST_SYSTEM_WINDOW) {
        // 系统窗口token获取逻辑
        ...
    } else {
        // 应用窗口token获取逻辑
        if (wp.token == null) {
            wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
        }
        ...
    }
    ...
}
```

终于看到了 token 的赋值了，这里分为三种情况：应用层窗口、子窗口和系统窗口，分别进行 token 赋值。

应用窗口直接获取的是与 WindowManager 对应的 PhoneWindow 的 mAppToken，而子窗口是拿到 DecorView 的 token，系统窗口属于比较特殊的窗口，使用 Application 也可以弹出，但是需要权限。而这里的关键就是：**这个 dialog 是什么类型的窗口？以及windowManager 对应的 PhoneWindow 有没有 token？**

而这个判断跟我们前面赋值的不同 WindowManagerImpl 有直接的关系。到这里，就必须到 Activity 和 Application 创建 WindowManager 的过程一看究竟了。

## Activity 与 Application 的 WindowManager

首先我们看到 Activity 的 window 创建流程。这里需要对Activity 的启动流程有一定的了解。追踪 Activity 的启动流程，最终会追到 ActivityThread 的 `performLaunchActivity`:

```java
ActivityThread.class(api29)
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
	// 最终会调用这个方法来创建window
    // 注意r.token参数 appContext 是 contextImpl
    activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window, r.configCallback,
        r.assistToken);
    ...
}
```

这个方法调用了 Activity 的 attach 方法来初始化 window，同时我们看到参数里面有 r.token 这个参数，这个token 最终回到哪里去，我们赶紧继续看下去：

```java
Activity.class(api29)
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    ...
	// 创建window
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ...
	// 创建windowManager
    // 注意token参数
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    mWindowManager = mWindow.getWindowManager();
    ...
}
```

attach 方法里面创建了 PhoneWindow 以及对应的 WindowManager,再把创建的 windowManager 给到 Activity 的mWindowManager 属性。我们看到创建 WindowManager 的参数里面有 token，我们继续看下去：

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated;
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

这里利用应用服务的 windowManager 给 Activity 创建了 WindowManager。同时把 token 保存在了 PhoneWindow 内，到这里我们知道 Activity 的 PhoneWindow 是拥有 token 的，那么 Application 呢？

Application 调用的是 ContextImpl 的 getSystemService 方法，而这个方法返回的是应用服务的 windowManager，Application本身并没有创建自己的 PhoneWindow 和 windowManager，所以也没有给 PhoneWindow 赋值 token 的过程。

因此**，Activity 拥有自己的 PhoneWindow 以及 WindowManager，同时他的 PhoneWindow 拥有 token；而 Application并没有自己的 Phone Window，他返回的WindowManager 是应用服务的 windowManager，并没有赋值 token 的过程**。

那么到这里结论已经出来了，我们回到赋值 token 的那个方法中：

```java
Window.class(api29)
void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
    if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
            wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
        // 子窗口token获取逻辑
        if (wp.token == null) {
            View decor = peekDecorView();
            if (decor != null) {
                wp.token = decor.getWindowToken();
            }
        }
        ...
    } else {
        // 应用窗口token获取逻辑
        if (wp.token == null) {
            wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
        }
        ...
    }
    ...
}
```

当我们使用 Activity 来添加 Dialog  的时候，此时 Activity 的 DecorView 已经是添加到 屏幕上了，也就是我们的 Activity是有界面了，这个情况下，他是属于子窗口的类型被添加到 PhoneWindow 中，而他的 token 就是 DecorView 的token，此时DecorView 已经被添加到屏幕上了，他本身是拥有 token 的。

> 当一个view 被添加到屏幕上后，他所对应 的 viewRootImpl 有一个 token 对象，这个 token 来自 WindowManagerGlobal，他是一个 IWindowSession 对象。从源码中可以看到，当我们的 PhoneWindow 的 DecorView 展示到屏幕后，后续添加的子window的token，就都是这个 IWindowSession 对象了

而如果是第一次添加，也就是应用界面，那么他的token 就是Activity 初始化传入的 token。但是如果使用的是 

Application，因为他的内部没有token，那么这里获取到的token就是null，后面到 WMS 就会抛出遗产给了，而这也就是为什么使用 Activity可以弹出 Dialog 而 Application 不可以的原因，因为受到了 token 的限制。

## WMS 是如何验证 token 的？

到这里我们已经知道。我们从 WMS 的token 判断找到了 token 的类型以及 token 的载体：WindowManager.LayoutParams,然后我们再从 dialog 的创建流程追到了赋值 token 的时候会因为 windowManager 的不同而不同。因此我们再去查看了两者不同的 windowManager，最终得到结论 **Activity 的 PhoneWindow 拥有 token，而 Application 使用的是应用级服务的 WindowManager，并没有 token**。

那么此时还是还有疑问：

- token 到底是在什么时候被创建的？
- WMS 怎么知道这个 token 是合法的？

虽然到目前我们已经弄清原因，但是知识却少了一块，秉着探索知识的好奇心我们继续研究下去：

我们从前面 Activity 的创建 window 过程知道 token 来自 `r.token`,这个 `r`  是 ActivityRecord,是AMS 启动 Activity 的时候传进来的 Activity 信息。那么要追踪这个 token 的创建就必须顺着这个 `r` 的传递线路一路回溯，同样这设计到 Activity 的完整启动流程。首先看一下这个 ActivityRecord 是在哪里被创建的：

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

这样我们需要继续往前回溯，看看这个 token 是在哪里被获取的：

```java
/frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java
    
public void execute(ClientTransaction transaction) {
    ...
    executeCallbacks(transaction);
    ...
}
public void executeCallbacks(ClientTransaction transaction) {
    ...
        final IBinder token = transaction.getActivityToken();
        item.execute(mTransactionHandler, token, mPendingActions);
    ...
}
```

可以看到我们的 token 在 ClientTransaction 对象获取到。ClientTransaction 是 AMS 传来的一个事务，负责 Activity 的启动，里面包含两个 item，一个复制执行 activity 的 create 工作，一个负责 activity 的resume 工作。那么我们这里就需要到 ClientTransaction 的创建过程一探究竟了。下面我们的逻辑就要进入系统进程了：

```java
ActivityStackSupervisor.class(api28)
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
    boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
            r.appToken);
    ...
}
```

这个方法创建了 ClientTransaction，但是 token 并不是在这里被创建的，我们继续往上回溯：

```java
ActivityStarter.java(api28)
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        SafeActivityOptions options,
        boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
        TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup) {
    ...
  
    //记录得到的activity信息
    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
            resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
            mSupervisor, checkedOptions, sourceRecord);
   ...
}
```

我们一路回溯，终于看到了 ActivityRecord 的创建，我们进去构造方法中看看有没有 token 相关的构造：

```java
ActivityRecord.class(api28)
ActivityRecord(... Intent _intent,...) {
    appToken = new Token(this, _intent);
    ...
}

static class Token extends IApplicationToken.Stub {
   ...
    Token(ActivityRecord activity, Intent intent) {
        weakActivity = new WeakReference<>(activity);
        name = intent.getComponent().flattenToShortString();
    }
    ...
}
```

可以看到确实这里进行了 token 的创建。而这个 token 看接口是一个 binder 对象 ，他持有 ActivityRecord 的弱引用，这样可以访问到 activity 的所有信息，到这里 token 的创建我们也找到了，那么 WMS 是怎么知道一个 token 是合法的呢？每个 token 创建之后，会后续发送到 WMS，WMS 对token进行缓存，而后续对于应用发送来 的token 只需要拿出来匹配一下就知道是否合法了。那么 WMS 是怎么拿到 token 的呢？

activity 的启动流程后续会走到一个方法：`startActivityLocked`，他有一个重要的方法调用，如下：

```
ActivityStack.class(api28)
void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity,
        boolean newTask, boolean keepCurTransition, ActivityOptions options) {
    ...
    r.createWindowContainer();
    ...
}
```

这个方法就把 token 发送到了 WMS 那里去，我们继续看一下:

```java
ActivityRecord.class(api28)
void createWindowContainer() {
    ...
    // 注意参数有token，这个token就是之前初始化的token
    mWindowContainerController = new AppWindowContainerController(taskController, appToken,
            this, Integer.MAX_VALUE /* add on top */, info.screenOrientation, fullscreen,
            (info.flags & FLAG_SHOW_FOR_ALL_USERS) != 0, info.configChanges,
            task.voiceSession != null, mLaunchTaskBehind, isAlwaysFocusable(),
            appInfo.targetSdkVersion, mRotationAnimationHint,
            ActivityManagerService.getInputDispatchingTimeoutLocked(this) * 1000000L);
	...
}
```

注意参数有 token，这个 token 就是之前初始化的 token，我们进入到他的构造方法看一下：

```java
AppWindowContainerController.class(api28)
public AppWindowContainerController(TaskWindowContainerController taskController,
        IApplicationToken token, AppWindowContainerListener listener, int index,
        int requestedOrientation, boolean fullscreen, boolean showForAllUsers, int configChanges,
        boolean voiceInteraction, boolean launchTaskBehind, boolean alwaysFocusable,
        int targetSdkVersion, int rotationAnimationHint, long inputDispatchingTimeoutNanos,
        WindowManagerService service) {
    ...
    synchronized(mWindowMap) {
        AppWindowToken atoken = mRoot.getAppWindowToken(mToken.asBinder());
       ...
        atoken = createAppWindow(mService, token, voiceInteraction, task.getDisplayContent(),
                inputDispatchingTimeoutNanos, fullscreen, showForAllUsers, targetSdkVersion,
                requestedOrientation, rotationAnimationHint, configChanges, launchTaskBehind,
                alwaysFocusable, this);
        ...
    }
}
```

还记得在看 WMS的时候他验证的是什么对象吗？WindowToken，而AppWindowToken 就是 WindowToken 的子类，继续看下去：

```
AppWindowContainerController.class(api28)
AppWindowToken createAppWindow(WindowManagerService service, IApplicationToken token,
        boolean voiceInteraction, DisplayContent dc, long inputDispatchingTimeoutNanos,
        boolean fullscreen, boolean showForAllUsers, int targetSdk, int orientation,
        int rotationAnimationHint, int configChanges, boolean launchTaskBehind,
        boolean alwaysFocusable, AppWindowContainerController controller) {
    return new AppWindowToken(service, token, voiceInteraction, dc,
            inputDispatchingTimeoutNanos, fullscreen, showForAllUsers, targetSdk, orientation,
            rotationAnimationHint, configChanges, launchTaskBehind, alwaysFocusable,
            controller);
}
AppWindowToken(WindowManagerService service, IApplicationToken token, ...) {
    this(service, token, voiceInteraction, dc, fullscreen);
    ...
}

WindowToken.class
WindowToken(WindowManagerService service, IBinder _token, int type, boolean persistOnEmpty,
        DisplayContent dc, boolean ownerCanManageAppTokens, boolean roundedCornerOverlay) {
    token = _token;
    ...
    onDisplayChanged(dc);
}
```

createAppWindow 方法调用了 AppWindow 的构造器，然后再调用了父类 WindowToken  的构造器，我们可以看到这里对 token 进行了缓存，并调用了一个方法，我们看看这个方法做了什么：

```java
WindowToken.class
void onDisplayChanged(DisplayContent dc) {
    dc.reParentWindowToken(this);
	...
}

DisplayContent.class(api28)
void reParentWindowToken(WindowToken token) {
    addWindowToken(token.token, token);
}
private void addWindowToken(IBinder binder, WindowToken token) {
    ...
    mTokenMap.put(binder, token);
    ...
}
```

mTokenMap 是一个 HashMap<IBinder，WindowToken> 对象，这里就可以保存一开始初始化的 token 以及后来创建的 windowToken 两者的关系，这里的逻辑其实已经实在 WMS中了，这个也是保存在 WMS中。AMS 和 WMS 都是运行在系统服务进程中的，所以他们之间是可以调用方法，不存在跨进程。WMS 就可以根据 IBinder 对象拿到 windowToken 进行信息对比了。

## 整体流程把握

前面根据思考问题的思维走完了整个 token 流程，感觉还有一点乱，下面画一张图梳理一下：

![](/picture/data-26.image)

- token 在创建 ActivityRecord 的时候一起被创建，他是一个 IBinder 对象，实现了接口 IApplicationToken
- token  创建之后会发送到 WMS，WMS 中封装程 WindowToken，并存在一个 HashMap
- token 会随着 ActivityRecord 被发送到本地进程，ActivityRecord 根据 AMS的指令执行Activity 的启动逻辑
- Activity 启动的过程中会创建 PhoneWindow 和 对应的 WindowManager，同时把 token 存在 PhoneWindow 中
- 通过 Activity 的WindowManager 添加 view/ 弹出 dialog 会把 PhoneWindow 中的 token 放在窗口 LayoutParams 中
- 通过 view RootImpl 向 WMS 验证，WMS 在 LayoutParams 拿到 IBinder 之后就可以在 Map 中获取 WindowToken
- 根据获取的结果就可以判断 该 token 的合法情况

## 从源码设计看 token

在 context 一文中讲到，不同的 context 有不同的职责，系统对不同的 context 限制了不同的权利，让其在对应场景下的组件只能做对应的事情。

token看着是属于 window 机制 的领域内容，其实是 context 的机制范围。我们知道 context 有三个实现类：Activity、Application、Service，context 是区分一个类是普通 Java 类还是 Android 组件的关键。context 拥有访问系统资源的权限，是各种组件访问系统的接口对象，但是三种 context 只有 Activity 允许有界面。**为了防止开发者乱用 context 造成混乱，那么必须对 context 的权限进行限制，这也是 token 存在的意义**。拥有 token 的context 可以创建界面、进行UI操作，而没有 token 的context 如 service、Application，是不允许添加 view 到屏幕上的。

为什么说这不属于 window 机制的范畴，window 机制中我们知道 WMS 控制每一个 window，是通过 ViewRootImpl 中的 IWindowSession 来进行通信的，token 在这个过程中只充当了一个验证的作用，且当 PhoneWindow 显示了 DecorView 之后，后续添加的 view 使用的 token 都是 ViewRootImpl 中的 IWindowSession 对象，这表示当一个 PhoneWindow 可以显示界面之后，那么 与后续其添加的 view 无需再次判断。因而，**token真正限制的，是  context 是否可以显示界面，而不是 window**。

最后得到结论就是不要使用**Application或者 Service 做 UI操作**。

## 总结

android体系中各种机制之间是互相联系，彼此连接构成一个完整的系统框架。token涉及到window机制和context机制，同时对activity的启动流程也要有一定的了解。阅读源码各种机制的源码，可以从多个维度来帮助我们对一个知识点的理解。同时阅读源码的过程中，不要局限在当前的模块内，思考不同机制之间的联系，系统为什么要这么设计，解决了什么问题，可以帮助我们从架构的角度去理解整个android源码设计。阅读源码切忌无目标乱看一波，要有明确的目标、验证什么问题，针对性寻找那一部分的源码，与问题无关的源码暂时忽略，不然会在源码的海洋里游着游着就溺亡了。















