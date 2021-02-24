# Android 全面解析之context机制三：在认知 context

## 前言

广播和内容提供器并不是context 家族中的一员，所以它们本身并不是 context，因为它们的context 肯定是直接或间接从 Application、Activity、Service 获取。然后对 context 的设计进行了讨论，从更高的角度看 Context，能够帮助我们看到 context 的本质，也能帮助我们更好的理解并使用  context。

## BroadCast 的 context 获取流程

broadcast 和上面的组件不同，它不是继承于 context，所以它的 context 是需要通过 Application、Activity 或者 service 来给予。我们一般使用广播的 context 是在接收器中，如：

```java
class MyClass :BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        TODO("use context")
    }
}
```

那么 onReceive 的 context 对象是从哪里来的呢？同样我们先看广播接收器的注册流程：

![](picture/data-05.image)

因为在创建 Receiver 的时候没有传入 context，所以我们要追踪它的注册流程，看看在哪里获取了 context。我们先看到 ContextImpl 的 `registerReceiver`方法：

```java
ContextImpl.class(api29)
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
        String broadcastPermission, Handler scheduler) {
    // 注意参数
    return registerReceiverInternal(receiver, getUserId(),
            filter, broadcastPermission, scheduler, getOuterContext(), 0);
}
```

registerReceiver 方法最终会来到这个重载方法，我们可以注意到，这里有个 getOuterContext，这个是什么？还记得 Activity 的 context 创建过程吗？这个方法获取的就是 Activity 本身。我们继续看下去：

```java
ContextImpl.class(api29)
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
        IntentFilter filter, String broadcastPermission,
        Handler scheduler, Context context, int flags) {
    IIntentReceiver rd = null;
    if (receiver != null) {
        if (mPackageInfo != null && context != null) {
            ...
            rd = mPackageInfo.getReceiverDispatcher(
                receiver, context, scheduler,
                mMainThread.getInstrumentation(), true);
        }
        ...
    }
    ...
}
```

这里利用 context 创建了 ReceiverDispatcher，我们继续深入看：

```java
LoadedApk.class(api29)
public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
        Context context, Handler handler,
        Instrumentation instrumentation, boolean registered) {
    synchronized (mReceivers) {
        LoadedApk.ReceiverDispatcher rd = null;
        ...
        if (rd == null) {
            rd = new ReceiverDispatcher(r, context, handler,
                    instrumentation, registered);
            ...
        }
        ...
    }
}

ReceiverDispatcher.class(api29)
ReceiverDispatcher(..., Context context,...) {
    ...
    mContext = context;
    ...
}
```

这里确实把 receiver 和 context 创建了 ReceiverDispatcher。但是，怎么没有给 Receiver？其实这涉及到广播的内部设计结构。Receiver 是没有跨进程通信能力的，而广播需要 AMS 的调控，所以必须有一个可以跟AMS 沟通的对象，这个对象就是 InnerReceiver，而 ReceiverDispatcher 就是负责维护它们两个之间的联系，如下图：

![](/picture/data-06.image)

而onReceive 方法也是由 ReceiveDispatcher 回调的，最后我们再看回调 onReceivce 的那部分代码：

```java
ReceiverDispatcher.java/Args.class;
public final Runnable getRunnable() {
    return () -> {
        ...;
        try {
            ...;
            // 可以看到这里回调了receiver的方法，这样整个接收广播的流程就走完了。
            receiver.onReceive(mContext, intent);
        }
    }
}
```

Args.class 是 receiver 的内部类，mContext 就在在创建 ReceiverDispatcher 时传入的对象，到这里我们就知道这个对象是 Activity 了。

但是，不一定每个都是 Activity，在源码中我们知道是通过 `getOuterContext`来获取 context，如果是通过别的context 注册广播，那么对应的对象也就不同了，只是我们一般都在 Activity 中创建广播，所以这个Context一般是 Activity对象。

## ContentProvider 的 context 获取流程

ContentProvider 我们用的太少了，内容提供器主要用于应用间内容共享的。虽然ContentProvider 是由系统提供的，但是它本身并不属于 Context 家族体系内，所以它的 context 也是从其他获取的。先看下 ContentProvider 的创建流程：

![](picture/data-07.image)

这不是 Application 创建的流程图吗？是的，ContentProvider 是伴随着应用启动被创建的，来看一张更加详细的流程图：

![](picture/data-08.image)

我们把目光聚集到 ContentProvider 的创建上，也就是 `installContentProvider`方法。这个方法是在 handleBindApplication 中被调用的，我们看到调用这个方法的地方：

```java
private void handleBindApplication(AppBindData data) {
    try {
        // 创建Application
        app = data.info.makeApplication(data.restrictedBackupMode, null);
  ...
        if (!data.restrictedBackupMode) {
            if (!ArrayUtils.isEmpty(data.providers)) {
                // 安装ContentProvider
                installContentProviders(app, data.providers);
        }
    }    
}
```

可以看到这里传入了 application 对象，我们继续看下去：

```java
private void installContentProviders(
        Context context, List<ProviderInfo> providers) {
    final ArrayList<ContentProviderHolder> results = new ArrayList<>();
    for (ProviderInfo cpi : providers) {
        ...
        ContentProviderHolder cph = installProvider(context, null, cpi,
                false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
        ...
    }
...
}
```

这里调用了 installProvider，继续往下看：

```java
private ContentProviderHolder installProvider(Context context,
        ContentProviderHolder holder, ProviderInfo info,
        boolean noisy, boolean noReleaseNeeded, boolean stable) {
    ContentProvider localProvider = null;
    IContentProvider provider;
    if (holder == null || holder.provider == null) {
        ...
  // 这里c最终是由context构造的
        Context c = null;
        ApplicationInfo ai = info.applicationInfo;
        if (context.getPackageName().equals(ai.packageName)) {
            c = context;
        }
        ...
        try {
            // 创建ContentProvider
            final java.lang.ClassLoader cl = c.getClassLoader();
            LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
            ...
            localProvider = packageInfo.getAppFactory()
                    .instantiateProvider(cl, info.name);
            provider = localProvider.getIContentProvider();
            ...
   // 把context设置给ContentProvider
            localProvider.attachInfo(c, info);
        } 
        ...
    } 
    ...
}
```

这里最重要的一行代码是 localProvider.attach(c,info),在这里把 context 设置给了 ContentProvider，我们在深入一点看看：

```java
ContentProvider.class(api29)
public void attachInfo(Context context, ProviderInfo info) {
    attachInfo(context, info, false);
}
private void attachInfo(Context context, ProviderInfo info, boolean testing) {
    ...
    if (mContext == null) {
        mContext = context;
        ...
    }
    ...
}
```

这里确实把 context 赋值给了 ContentProvider 的内部变量 mContext，这样 ContentProvider 就可以使用 Context 了。而这个 context 正是一开始传进来的 Application。

## 从源码设计角度看 Context

研究 FrameWork 层知识，不能只停留在他是什么，有什么作用即可。FrameWork 层是一个整体，构成了 Android 这个庞大的体系，还需要看Context，在其中扮演者什么样的角色，解决了什么样的问题。Window 机制中讲到 window 的存在是为了解决屏幕上 view 的显示逻辑和触摸反馈问题。那么 Context呢？有什么作用呢？

Android 系统是一个完整的生态，它搭建了一个环境，让各种程序可以上面运行。而任何一个程序，想运行在这个环境上，必须要得到系统的允许，也就是 **软件安装**。安卓与电脑不同的是，它不是任意一个程序就可以直接访问到系统资源。我们在 windows 上面可以写一个 Java 程序，然后直接开启一个文件流就可以读取和修改文件了，而Android 却没有那么简单，它 **任意一个程序的运行都必须得到系统的调控**。也就是，即使程序获得许可，程序本身要运行，还是需要系统来控制程序运行，程序无法自发的执行在 Android 环境中。我们通过源码知道 程序的main 方法，仅仅只是开启了线程的 Looper循环，而后续的一切，都必须等待 AMS 控制。

那应用程序自己硬要执行可不可以？可以，但是没用，想要获取系统资源，如启动四大组件、读取布局文件、读写数据库、调用系统摄像头等等，都必须要通过 Context，而Context必须要通过AMS 来获取。这就区分了一个程序是一个普通的 Java 程序，还是Android 程序。

Context 承受的两大职责：身份权限、访问系统的接口。一个 Java 类，如果没有 Context 那么就是一个普通的类，而当他获得 context 那么就可以称之为一个组件了，因为它获得了访问系统的权限，它不再是一个普通的身份，是属于 android 的公民了，而 公民并不是无法无天，系统也可以通过 context 来封装以及限制程序的权限。想要弹出一个通知，你必须通过这个 api，用户关闭你的通知权限，你就不能通过其他路径来弹出通知了。同时程序也无需知道底层到底是如何实现，只管调用 api 即可。四大组件为何称为四大组件，因为它们生来就有了 context，特别是 activity 和 service，包括 Application。而我们写的一切程序，都必须间接或者直接从中获取context。

总而言之，context 就是负责区分 Android 内外程序的一个机制，限制程序访问系统的资源。

## 最后

关于 context 的就这些了。虽然内容平常很少使用，但是非常有助于我们对Android 整个系统框架的理解，而当我们对系统有更加深入的了解之后，写出来的程序也会更加的健壮。























