# Android 全面解析之 context机制 二：context 创建流程

在（1）中，我们已经了解了 context 了，但是始终缺少一点什么：activity 是什么时候创建的，它的 contextImpl 是如何被赋值的？Application呢？为什么说 contentProvider 的context 是 Application，Broadcast 的 context 是 Activity？contextImpl 又是如何被创建的？如何要解决这些疑惑，那就必须要阅读源码了，阅读源码可以形成自己对整个机制的思考和理解。同时可以让自己对 context 的知识真正落实到代码上，增强自己对知识的自信心。

## Application

Application 是应用级别的 context，是在应用创建的时候的被创建的，是第一个被创建的 context，也是最后一个被销毁的 context。因为追踪 Application 的创建需要从程序的启动流程看起。应用启动的源码流程如下（简化版本）：

![](picture/data-02.image)

应用程序从 ActivityThread 的main 方法开始执行，从 Handler 消息机制中我们知道 main 方法主要是开启线程的 loopler 以及 handler，然后由AMS 向主线程发送 message 控制程序的启动过程，因为我们这里可以把目标锁定在图中的最后一个方法： handleBindApplication,Application最有可能就是在这里被创建：

```java
ActivityThread.class (api29)

private void handleBindApplication(AppBindData data) {
    ...
 // 创建LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    ...
    Application app;
    ...
    try {
  // 创建Application
        app = data.info.makeApplication(data.restrictedBackupMode, null);
        ...
    }
    try {
        ...
  // 回调Application的onCreate方法
        mInstrumentation.callApplicationOnCreate(app);
    }
    ...
}
```

handleBindApplication 的参数AppBindData 是 AMS 给应用程序的启动信息，其中就包含了“权限凭证”----ApplicationInfo 等。LoadedApk 就是通过这些对象来获取创建对系统资源的访问权限，然后通过 LoadApk 来创建 contextImpl 以及 Application。

这里我们只关注和 context 创建有关的逻辑，前面启动程序的源码以及 AMS 如何处理，这里不讲了。那么接下来我们继续关注 Application 是如何创建的：

```java
LoadeApk.class(api29)
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    // 如果application已经存在则直接返回
    if (mApplication != null) {
        return mApplication;
    }
 ...
    Application app = null;
    String appClass = mApplicationInfo.className;
    ...
    try {
        java.lang.ClassLoader cl = getClassLoader();
       ...
  // 创建ContextImpl
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
  // 利用类加载器加载我们在AndroidMenifest指定的Application类
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        // 把Application的引用给comtextImpl，这样contextImpl也可以很方便地访问Application
        appContext.setOuterContext(app);
    } 
    ...
    mActivityThread.mAllApplications.add(app);
    // 把app设置为mApplication，当我们调用context.getApplicationContext就是获取这个对象
    mApplication = app;

    if (instrumentation != null) {
        try {
   // 回调Application的onCreate方法
            instrumentation.callApplicationOnCreate(app);
        } 
        ...
    }
  ...
    return app;
}
```

代码的逻辑也不复杂，首先判断 LoadedApk 对象中的 mApplication 是否存在，否则创建 ContextImpl，在利用类加载器和 ContextImpl 创建 Application，最后把 Application 对象赋值给 LoadedApk 的 mApplication，再回调 Application 的 onCreate 方法。我们先看一下 contextImpl 是如何被创建的：

```java
ContextImpl.class(api29)
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo,
        String opPackageName) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
            null, opPackageName);
    context.setResources(packageInfo.getResources());
    return context;
}
```

这里直接 new 了一个 ContextImpl，同时给 contextImpl 赋值 访问系统资源相关的“权限”对象------ActivityThread,LoadApk 等。让我们再回到 Application的创建过程。我们可以猜测，再 newApplication 包含的逻辑肯定有：利用反射创建 Application，再把 contextImpl 赋值给 Application。原因是每个人自定义的 Application 类不同，需要利用反射来创建对象，其次 Application 中的 mBase 属性是对 ContextImpl 的引用。看源码：

```java
Instrumentation.class(api29)
public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
    Application app = getFactory(context.getPackageName())
            .instantiateApplication(cl, className);
    app.attach(context);
    return app;
}

Application.class(api29)
final void attach(Context context) {
    attachBaseContext(context);
    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
}

ContextWrapper.class(api29)
Context mBase;    
protected void attachBaseContext(Context base) {
    if (mBase != null) {
        throw new IllegalStateException("Base context already set");
    }
    mBase = base;
}    
```

结果非常符合我们的预测，先创建 Application 对象，再把 ContextImpl 通过 Application 的 attach 方法赋值给 Application。然后 Application 的 attach 方法调用了ContextWrapper 的 attachBaseContext 方法，因为 Application 也是继承自 ContextWrapper。这样，就把 ContextImpl 赋值给 Application 的mBase 属性 了。

再回到前面的逻辑，创建了 Application 之后需要回调 onCreate 方法：

```java
Instrumentation.class(api29)
public void callApplicationOnCreate(Application app) {
    app.onCreate();
}
```

简单粗暴，直接回到。到这里，Application 的创建以及 context 的创建流程就走完了。但是需要注意的是，**全局初始化需要在 onCreate 中进行，而不要在 Application 的构造器进行**。从代码中我们 可以看到 ContextImpl 是在  Application 被创建之后再赋值的。

## Activity

Activity 的 context 也是在Activity 创建的过程中被创建的，这个就涉及到 Activity 的启动流程，这里涉及到三个流程：应用程序请求 AMS，AMS 处理请求，应用程序响应 Activity 创建事务：

![](/picture/data-03.image)

依然，我们专注于 Activity 的创建流程，和 Application 一样，Activity 的创建是由 AMS 来控制的,AMS 向应用进程发送消息来执行具体的启动逻辑。最后会执行到 handleLaunchActivity 这个方法：

```java
ActivityThread.class(api29)
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ...
    final Activity a = performLaunchActivity(r, customIntent);
 ...
   return a;
}
```

最终的就是中间这句代码，进入看源码：

```java
ActivityThread.class(api29)
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
 // 创建Activity的ContextImpl
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        // 利用类加载创建activity实例
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        ...
    }
    try {
  // 创建Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
  ...
        if (activity != null) {
            ...
   // 把activity设置给context，这样context也可以访问到activity了
            appContext.setOuterContext(activity);
            // 调用activity的attach方法把contextImpl设置给activity
            activity.attach(appContext, this, getInstrumentation(), r.token,
                            r.ident, app, r.intent, r.activityInfo, title, r.parent,
                            r.embeddedID, r.lastNonConfigurationInstances, config,
                            r.referrer, r.voiceInteractor, window, r.configCallback,
                            r.assistToken);

            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                // 设置主题
                activity.setTheme(theme);
            }
            ...
   // 回调onCreate方法
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ...
        }
        ...
    }
 ...
    return activity;
}
```

代码的逻辑不是很复杂，首先创建 Activity 的 ContextImpl，利用类加载器创建 Activity 实例，然后再通过 LoadedApk 创建 Application，这个方法再前面讨论过，如果 Application 已经创建会直接返回已经创建的对象。然后把 Activity 上设置给 Context，这样 context就可以访问到 Activity了。这里要注意，前面井道使用 Activity 的 context 会造成内存泄漏，那么可不可以用 Activity 的 contextImpl 对象呢？答案是不可以，因为**ContextImpl 也会持有Activity 的引用**。需要特别注意一下，然后再调用 activity 的 attach 方法把 ContextImpl 设置给contextImpl 设置给 activity。后面是设置主题和回调 onCreate 方法，我们就不深入了，主要看看 attach 方法：

```java
Activity.class(api29)
final void attach(Context context,...) {
    attachBaseContext(context);
  ...   
}
```

这里省略了大量的代码，只保留一句关键：attachBaseContext ，是不是很熟悉？调用 ContextWrapper 的方法来给 mBase  赋值，和前面的 Application 是一样的，就不再赘述。

## Service

依然只关注关键代码流程，先看Service 的启动流程图：

![](picture/data-04.image)

Service 的创建过程也是受到 AMS的控制，同样我们看到创建 Service 的那一步，最终会调用到 handleCreateService 方法：

```java
private void handleCreateService(CreateServiceData data) {
    ...
    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
    } 
    ...
    try {
        ...
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);

        Application app = packageInfo.makeApplication(false, mInstrumentation);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        service.onCreate();
        mServices.put(data.token, service);
        ...
    } 
    ...
}
```

Service 的逻辑讲相对简单了，同样创建 service 实例，再创建 contextImpl，最后把 contextImpl通过 service 的 attach 方法赋值给 mBase 属性，最后回调 Service 的 onCreate 方法，过程和上面的很相似，这里就不再深入讲解了。

## 结束



























