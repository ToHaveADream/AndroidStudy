# Android 全面解析之Activity 生命周期

#### 前言

本篇文章讲解的重点是 Activity 的生命周期，在文章的最后也会设计 Activity 的设计。

- 文章第一部分讲解关于 Activity 状态的认知
- 第二部分全面讲解 Activity 生命周期回调方法
- 第三部分是分析不同情况下的生命周期回调顺序
- 第四部分是源码分析
- 最后一部分是从更高的角度来思考 Activity 以及生命周期

Let's go!!!

## 生命状态概述

Activity 是一个很重要、很复杂的组件，他的启动不像我们平时直接 new 一个对象就完事了，他需要经历一系列的初始化，例如“刚创建状态”。“后台状态”，“可见状态”等等。当我们在界面之间进行切换的时候，Activity 也会在多种状态之间进行切换，例如 可见或不可见状态、前台或者后台状态。**当 Activity 在不同的状态之间切换时，会回调不同的生命周期方法。我们可以重写这一些方法。当进入不同的状态的时候，执行对应的逻辑**。

在 ActivityLifecycleItem 抽象类中定义了 9 种状态。这个抽象类有很多的子类，是 AMS 管理 Activity 生命周期的事务类。（其实就像是一个圣旨，AMS 丢给应用，那么应用程序就必须设置这个圣旨）Activity 主要使用其中6个（这里的6个在源码中明确的看到调用 setState 来设置状态，其他的三种并未看到调用 setState 方法来设置状态，这里主要讲6种），如下：

```java
// Activity刚被创建时
public static final int ON_CREATE = 1;
// 执行完转到前台的最后准备工作
public static final int ON_START = 2;
// 执行完即将与用户交互的最后准备工作
// 此时该activity位于前台
public static final int ON_RESUME = 3;
// 用户离开，activity进入后台
public static final int ON_PAUSE = 4;
// activity不可见
public static final int ON_STOP = 5;
// 执行完被销毁前最后的准备工作
public static final int ON_DESTROY = 6;
```

状态之间的跳转不是随意的，例如不能从 ON_CREATE 直接跳转到 ON_PAUSE 状态，状态之间的跳转受到 AMS 的管理。当 Activity 在这些状态之间进行切换的时候，就会回调对应的生命周期。这里的状态，画图来理解一下：

![](/picture/data-14.image)

这里按照可交互 可见 可存在 三个维度来区分 Activity 的生命状态。可交互则为是否可以与用户操作；可见则为是否显示在屏幕上；可存在，则为该 Activity 是否被系统杀死或者调用了 finish 方法。牵头的商法为进入对应状态会调用的方法。

> 在谷歌的官方文档中对于 onStart 方法是这样描述的：onStart  调用使Activity 对用户可见，因为应用会为 Activity 进入前台并支持互动做准备。而当 Activity 进入 ON_PAUSE 状态的时候，Activity 是可能依旧可见的，但是不可交互。如操作另一个应用的悬浮窗口的时候，当前应用的 Activity 会进入 ON_PAUSE 状态。
>
> 但是！在 Activity 启动的流程中，直到 onResume 方法被调用，界面依旧是不可见的。（后台可见）

**生命周期的一个重要作用就是让 Activity 在不同状态之间切换的时候，可以执行相应的逻辑**。举个例子：我们在界面A 使用了相机资源，当我们切换到下一个界面的时候，那么界面 A 就必须要释放相机资源，这样才不会导致界面 B 无法使用界面；而当我们切回界面 A 的时候，又希望界面A继续保持拥有相机资源的状态；那么我们就需要在界面不可见的时候释放资源，而在界面恢复的时候再次获取相机资源。每个 Activity一般情况下可以认为是一个界面或者说，一个屏幕。当我们在界面之间进行导航切换的时候，其实就是在切换 Activity。当界面在不同状态之间进行切换的时候，也就是 Activity状态的切换，就会回调 Activity 的相关的方法。例如当界面不可见的时候会回调 onStop 方法，恢复的时候会回调 onRestart 方法等。

**在合适的生命周期做合适的工作会让 App 变得更加有鲁棒性**。避免当用户跳转到别的 App 的时候发生崩溃、内存泄漏、当用户切回来的时候失去进度、当用户旋转屏幕的时候失去进度或者崩溃等等。这些都需要我们对生命周期有一定的认知，才能在具体的场景下做出正确的选择。

这一部分概述没有展开讲生命周期，而是需要重点理解**状态与状态之间的切换，生命周期的回调就发生在不同的状态之间的切换**。我们学习生命周期的一个重要目的就是**能否在对应的 业务场景下做合适的工作**，例如资源的申请、释放、存储、恢复，让 App 更加有鲁棒性。

## 重要生命周期解析

关于 Activity 重要的生命周期方法，谷歌官方有一张非常重要的流程图。如下所示：

![](picture/data-15.image)

#### 主要生命周期

首先我们先看到最重要的七个生命周期，这7个生命周期是严格意义上的生命周期，他符合状态切换这四个定义。这部分内容建议结合概述部分的图一起理解。（onRestart 不涉及状态切换，但因为执行完他之后会马上执行 onRestart，所以在一起讲）

- onCreate：当Activity 创建实例完成，并调用 attach 方法赋值 PhoneWindow、ContextImpl 等属性之后，调用此方法。该方法在 Activity 生命周期内只会调用一次。调用该方法之后会进入 ON_CREATE 状态。

  > 该方法是我们最频繁使用的一个回调方法
  >
  > 我们需要在这个方法中初始化基础组件和视图，如 viewModel,textView，同时必须在该方法中调用 setContentView来给 activity 设置布局
  >
  > 该方法接收一个参数，该参数保留之前状态的数据。如果是第一次启动，则该参数为空。该参数来自 onSavedInstanceState 存储的数据。只有当 Activity 暂时销毁并且预期一定会被创建的时候才会被调用，如屏幕旋转、后台应用被销毁等

- onStart：当 Activity 准备进入前台时会调用该方法。调用后 Activity 会进入 ON_START 状态。

  > 要注意理解这里的前台的意思。虽然谷歌文档中表示调用该方法之后 Activity 可见，如下图：
  >
  > 但是我们前面说到，**前台并不意味着Activity可见，只是表示 Activity 处于活跃状态**。
  >
  > 前台 Activity 一般只有一个，所以也就意味着**其他的 Activity进入后台了**。这里的前后台需要结合 Activity 返回栈来理解。
  >
  > **这个方法一般用于从别的  Activity 切回本 Activity 的时候调用**。
  >
  > ![](/picture/data-16.image)

- onResume:当 Activity 准备与用户交互的时候调用。调用之后 Activity 进入 ON_RESUME 状态。

  > 注意，这个方法一直被认为是 Activity 一定可见，且准备好与用户交互的状态。但事实并不是**一直这样**。如果在 onResume 方法中弹出 popupWindow 你会收货一个异常：token is null，表示界面尚没有被添加到屏幕上。
  >
  > 但是，这种情况只出现在第一次启动 Activity 的时候。当 Activity 启动后 decorView 就已经拥有 token 了，再次在 onResume 方法中弹出 popupWindow 就不会出现问题了。
  >
  > 因此，**在 onResume 调用的时候 activity 是否可见要区分是否是第一次创建 Activity**。
  >
  > onStart 方法是前台和后台的区分，而这个方法是是否可交互的区分。使用场景最多是在当弹出别的 Activity 的窗口时，原 Activity 就会进入 ON_PAUSE 状态，但是仍然可见：当再次回到原 Activity 的时候，就会调用 onResume 方法了。

- onPause：当前 Activity 窗口失去焦点的时候，会调用此方法。调用后 Activity 进入 ON_PAUSE 状态，并进入后台。

  > 这个方法一般在另一个 Activity 要进入前台前被调用。只有当前 Activity 进入后台，其他的 Activity 才能进入前台。所以，该方法不能做重量级的操作，不然会引起界面切换卡顿。
  >
  > 一般的使用场景为界面进入后台时的轻量级资源释放
  >
  > 最好理解这个状态就是弹出另一个 Activity 的窗口的时候，因为 前台 Activity 只能有一个，所以当前可交互的 Activity 变成另一个  Activity 的时候，原 Activity 就必须调用 onPause 方法；但是 仍然是可见的，只是无法进行交互。这里也可以更好的体会前台可交互和可见性的区别

- onStop：当 Activity 不可见的时候进行调用。调用后 Activity 进入 ON_STOP 状态。

  > 这里的不可见是严谨意义上的不可见。
  >
  > 当 Activity不可交互时会回调 onPause 方法并进入 ON_PAUSE 状态，但如果进入的是另一个全屏的Activity 而不是 小窗口，那么当新的Activity 界面显示出来的时候，原 Activity 会进入 ON_STOP 状态。同时，Activity第一次被创建的时候，界面是在 onResume 方法之后才显示出来，所以 onStop 方法会在新 Activity 的 onResume 方法回调之后被调用
  >
  > 注意，被启动的 Activity 并不是等待 onStop 执行完毕之后再显示。因而如果 onStop 方法里面做一些比较耗时的操作也不会导致被启动的 Activity 启动延时。
  >
  > onStop 方法的目的就是做资源释放操作。因为是在另一个 Activity 显示之后再被回调，所以这里可以做一些相对重量级别的资源释放操作，如中断网络请求、断开数据库连接、释放相机资源等。
  >
  > 如果一个应用的全部 Activity 都处于 ON_STOP 状态，那么这个应用是很有可能被系统杀死的。而如果一个 ON_STOP 状态的 Activity 被系统回收的话，系统会保留该Activity 中  view 的相关信息到 bundle 中，下一次恢复的时候，可以在 onCreate 或者 onRestoreInstanceState 中进行恢复。

- onRestart：当从另一个 Activity 切换会到 该 Activity 的时候会被调用。调用该方法后会立即调用onStart 方法，之后 Activity 进入 ON_START 方法。

  > 这个方法一般在 Activity 从 ON_STOP 状态被重新启动的时候会调用。执行该方法后会立即执行 onStart 方法，然后 Activity 进入 ON_START 状态，进入前台。

- onDestory：当 Activity 被系统杀死或者 调用 finish 方法之后，会回调该方法。调用该方法之后 Activity 进入 ON_DESTORY 状态

  > 这个方法是 Activity 在被销毁前调用的最后一个方法。我们需要在这个方法中释放所有的资源，防止造成内存泄漏问题。
  >
  > 回调该方法后的 Activity 就等待被系统回收了，如果再次打开该 Activity 需要从onCreate 开始执行，重新创建 Activity。

  #### 其他生命周期回调方法

  - onActivityResult

  这个方法也很常见，他需要结合 startActivityForResult 一起使用。

  使用的场景是：启动一个 Activity，并期望在该 Activity 结束的时候返回数据。

  当启动的 Activity 结束的时候，返回原Activity，原 Activity 就会回调 onActivityResult 方法了。**该方法在执行在其他所有的生命周期方法前**。

  - onSaveInstanceState/onRestoreInstanceState

  这两个方法，主要用于在 Activity 被意外杀死的情况下进行界面数据存储与恢复，什么叫做意外杀死呢？

  如果你主动点击返回键，调用 finish 方法，从多任务列表清楚后台应用等等，这些操作表示用于想完整的退出Activity，那么就没有必要保留界面数据了，所以也不会调用这个这两个方法。而当应用被系统意外杀死，或者系统配置修改导致的 Activity 销毁，这个时候当用户返回 Activity时，期望界面的数据还在，则会通过回调 onSaveInstanceState 方法来保留界面数据，则在 Activity 重新创建并运行的时候调用 onRestoreInstanceState 方法来恢复数据。事实上，onRestoreInstanceState 方法的参数和 onCreate  方法的参数是一致的，只是他们两个方法回调的时机不同。因此，判断是否执行的关键因素就是**用户是否期望返回该 Activity 时界面数据仍然存在**。

  这里需要注意几个点：

  1. 不同 Android 版本下，onSavedInstanceState 方法的调用时机是不同的。目前所看的源码是 API30，在官方注释中可以看到这一段话：

  > /*If called, this method will occur after {@link #onStop} for applications
  >  * targeting platforms starting with {@link android.os.Build.VERSION_CODES#P}.
  >  * For applications targeting earlier platform versions this method will occur
  >  * before {@link #onStop} and there are no guarantees about whether it will
  >  * occur before or after {@link #onPause}.
  >  */

  翻译过来的意思就是，在 api28 及以上的版本 onSaveInstanceState 是在 onStop 之后调用的，但是在低版本中，他是在onStop 方法之前被调用的，且与onPause 之间的孙旭是不确定。

  2.当Activity 进入后台的时候，onSaveInstanceState 方法则会被调用，而不是在异常情况下在会调用 onSaveInstanceState 方法，因为并不确定在后台的时候，Activity是否会被系统杀死，所以最保险的方法，先保存数据。当确实是因为异常情况被杀死时，返回 Activity用户期望界面需要恢复数据，才会调用onRestoreInstanceState 来恢复数据。但是，Activity直接按返回键或者 调用 finish 方法直接结束 Activity 的时候，是不会回调 onSaveInstanceState 方法，因为非常明确下一次返回该 Activity 用户期望的是一个干净的 Activity

  3、onSaveInstanceState 不能做重量级的数据存储。onSaveInstanceState 存储数据的原理是把数据序列化到磁盘中，如果存储的数据过大，会导致界面卡顿，掉帧等情况出现。

  4、正常情况下，每个 view 都会重写这两个方法，当 Activity 的这两个方法被调用的时候，会向上委托 window 去调用顶层 viewGroup 的这两个方法；而 viewGroup 会递归调用子 view 的 onSaveInstanceState onRestoreInstanceState 方法，这样所有 view 的状态就恢复了。

  #### onPostCreate

  这个方法其实和 onPostResume 是一样的，同样的还有 onContentChange 方法，这三个方法都是不常用的。

  onPostCreate 方法发生在 onRestoreInstanceState 之后，onResume 之前，他代表着界面数据已经完全恢复，就差显示出来与用户交互了。在 onStart 方法被调用时这些操作尚未完成。

  onPostResume 是在 Resume 方法被完全执行之后的回调。

  onContentChange 是在 setContentView 之后的回调。

  ##### onNewIntent

  这个方法涉及到的场景也是重复启动，但是与 onRestart 方法被调用的场景是不同的。

  我们知道 Activity 是有很多种启动模式的，其中 singleInstance、singleTop、singleTask都保证了在一定情况下得 单例状态。如 singleTop，如果我们启动一个正在处于栈顶且启动模式为 singleTop 的 activity，那么他并不会再创建一个 activity 实例，而是会回调该 Activity 的onNewIntent 方法，该方法接收一个 intent 参数，该参数就是新的启动  intent 实例。

  ## 场景生命周期流程

  这一部分主要讲解在一些场景下，生命周期方法的回调顺序。对于单个 Activity 而言，上述流程图已经展示了各种情况下的生命周期回调孙旭了。但是，当启动另外一个 Activity 的时候，到底是 onStop 限制性，还是被启动的 onStart 先执行呢？这些就变得难以确定

  验证生命周期回调顺序的最好方法就是写 demo 验证，通过日志打印、

  #### 正常启动和结束

  > onCreate onStart onResume onPause onStop onDestory

  #### Activity 切换

  > Activity1:onPause Activity2:onCreate onStart onResume Activity1:onStop

  当切换到另一个 activity  的时候，本 Activity会先调用 onPause 方法，进入后台；被启动的Activity依次调用三个回调方法或准备与用户交互；这时原 Activity 在调用 onStop 方法变得不可见，最后被启动的 Activity才会显示出来

  理解这个生命周期顺序只需要记住两个点：前后台，是否可见。onPause 调用之后，activity会进入后台，而前台交互的 activity只能有一个，所以原 activity必须先进入后台，目标 activity才能启动并进入前台。onStop 调用之后 activity 变得不可见，因而只有目标activity 即将要与用户交互的时候，需要进行显示了，原 activity 才会调用onStop 状态进入不可见状态

  下面看一下切换到另一个 Activity 的生命周期日志打印：

  ![](/picture/data-17.image)

  这里我们看到最后回调了 onSaveInstanceState 方法，前面我们讲到了，当 Activity 进入后台的时候，会回调该方法来保存数据。因为并不知道在后台时activity 是否会被系统杀死。下面再看一下从 activity2 返回的时候，生命周期的打印日志：

  ![](/picture/data-18.image)

  #### 屏幕旋转

  > running onPause onStop onSaveInstance onDestory
  >
  > onCreate onStart onRestoreInstanceState onResume

  当因为资源配置改变时，activity会销毁重建，最常见的就是屏幕旋转。这个时候属于异常情况的 Activity生命结束。所以，在销毁的时候，会调用 onSaveInstanceState 来保存数据，在重新创建新的 Activity 的时候，会调用 onRestoreInstanceState 来恢复数据。

  来看一下日志打印：

  ![](/picture/data-19.image)

  #### 应用后台被杀死

  > onDestory
  >
  > onCreate onStart onRestoreInstanceState onResume

  这个流程跟上面的资源配置更改是很像的，只是每个 Activity不可见的时候，会回调onSaveInstanceState 提前保存数据，那么在被后台杀死的时候，就不需要再次保存数据了

  #### 具体返回值的启动

  > onActivityResult onRestart onResume

  这里主要针对使用 startActivityForResult 方法启动另一个 Activity，当该 Activity 销毁并返回时，原 Activity 的 onActivityResult 方法的执行时机，大部分流程和 Activity 的切换时一样的。但是在返回原 Activity时，onActivityResult 方法会在其他所有的生命周期方法执行前被执行。看下日志打印：

  ![](/picture/data-20.image)

  #### 重复启动

  这个流程是比较容易在学习生命周期的时候被忽略的。主要是记得如果当前 Activity正处于栈顶，那么会先回调 onPause 之后再回调 onNewIntent。看一下日志打印：

  ![](/picture/data-21.image)

  ## 从源码看生命周期

  到这里关于生命周期的一些应用只是就已经讲的差不多了，这一部分是深入源码，去探究生命周期在源码中是如何实现的。这样对生命周期的理解会更加的深刻。

  这一部分的内容一共分为两个部分：第一部分是概述一下ActivityThread中关于每个生命周期的调用方法，这样大家就懂得如何去寻找对应的源码来研究；第二部分是拿onResume这个方法来举例讲解，同时解释为什么在第一次启动时，当onResume被调用时界面依然不可见。

  #### 从 ActivityThread 看生命周期

  我们都知道，Activity 的启动是受AMS调配的，那具体的调配方式是怎样的呢？

  通过 Handler机制我们知道，Android 的程序执行是使用 Handler 机制来实现消息驱动型的。AMS想要控制 Activity 的生命周期，就必须不断的向主线程发送Message；而程序想要执行 AMS 的命令，就必须handle 这些message 执行逻辑，两端配合，才能达到这种效果。

  打个比方，领导要吩咐下属去工作，他肯定不会把工作的具体流程都给下属，而只是会发个命令：如明天的演讲做个PPT，给我预约个下星期的飞机等等。那么下属，就必须根据这些命令来执行具体的逻辑。所以，在 Android 程序，肯定有一系列的逻辑，来分别执行来自 AMS的命令。这就是 ActivityThread 中的一系列 handlexxx 方法。

  当然，应用程序不止收到AMS的管理，同样还有 WMS、PMS等等系统服务。系统服务是运行在系统服务进程的，当系统服务需要控制应用程序的时候，会通过 binder 跨进程通信把消息发送给应用程序。应用程序的 Binder 线程会把消息发送给主线程去执行。因而，从这里可以看出，当应用程序刚被创建的时候，必须初始化的有主线程、binder 线程、主线程 Handler、以及提前编写了命令的执行逻辑的类 ActivityThread。画个图感受一下：

  ![](/picture/data-22.image)

  回到我们的生命周期主题。关于生命周期命令的执行方法主要有：

```java
handleLaunchActivity;
handleStartActivity;
handleResumeActivity;
handlePauseActivity;
handleStopActivity;
handleDestroyActivity;
```

具体的方法当然不止这么多，只是列出一些比较常用的。这些方法都在ActivtyThread 中。ActivityThread 每个应用程序有且只有一个。它是系统服务命令的执行者。

了解了AMS如何调配之后，那么他们的执行顺序如何确定呢？AMS 是先发送 handleStartActivity 命令呢，还是先发送 handleResumeActivity 命令呢？这里就需要对 Activity 的启动流程有一定的认识。

最后再延申一下，ActivityThread 可不可以自己决定执行逻辑，而不理会 AMS 的命令呢？答案是不，如果再公司里面，没有老板的同意，你就不能动用老板的资源。没有 AMS 的授权，应用程序时无法得到系统资源的，所以 AMS 就保证了每一个程序必须符合一定的规范。关于这方面，可以看下 context 的涉及。

接下来主要介绍下 handleResumeActivity 方法。

#### 解析 onResume 源码

根据我们前面的学习，handleResumeActivity 肯定时在 handleLaunchActivity 和 handleStartActivity 之后被执行的，我们直接来看源码：

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    ...
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ...;
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        ...
        if (r.activity.mVisibleFromClient) {
            r.activity.makeVisible();
        }
    }
    ...
}
```

代码截取了两个非常重要的部分。`performResumeActivity`最终会执行 `onResume`方法；`activity.makeVisible()`;是真正让界面显示在屏幕上的方法，我们看一下`makeVisible()`:

```java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

如果尚未添加到屏幕上，那么会调用 windowManager 的 addView 方法来添加，之后，activity界面才真正显示在屏幕上。回应之前的问题：为什么在onResume 弹出 popupWindow 会抛出异常而弹出 dialog却不会？原因就是这个时候activity 的界面尚未添加到屏幕上，而 popupWindow 需要依附于父界面，这个时候弹出就会抛出 token is null异常了。而dialog 属于应用级窗口，不需要依附于任何窗口，所以 dialog 在 onCreate 方法中弹出都是没有问题的。为了验证我们的判断，我们可以在生命周期中打印 decorView 的windowToken，当 decorView 被添加到屏幕上之后，就会被赋值 token了，看日志打印：

![](/picture/data-23.image)

可以看到，直到 onPostResume 方法执行，界面依旧没有显示在屏幕上。而直到 onWindowFocusChange 被执行时，界面才是真正显示在屏幕上了。

好了，让我们再回到一开始的源码，深入 performResumeActivity 方法中看看，在哪里执行了 onResume 方法：

```java
public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
        String reason) {
    ...
    try {
        ...
        if (r.pendingIntents != null) {
            // 判断是否需要执行newIntent方法
            deliverNewIntents(r, r.pendingIntents);
            r.pendingIntents = null;
        }
        if (r.pendingResults != null) {
            // 判断是否需要执行onActivityResult方法
            deliverResults(r, r.pendingResults, reason);
            r.pendingResults = null;
        }
        // 回调onResume方法
        r.activity.performResume(r.startsNotResumed, reason);

        r.state = null;
        r.persistentState = null;
        // 设置状态
        r.setState(ON_RESUME);

        reportTopResumedActivityChanged(r, r.isTopResumedActivity, "topWhenResuming");
    } 
    ...
}
```

这个方法的重点就是，先判断是否需要执行 onNewIntent 或者 onActivityResult 的场景，如果没有则执行调用 `peformResume`方法，我们深入 `performResume`方法看一下：

```java
final void performResume(boolean followedByPause, String reason) {
    dispatchActivityPreResumed();
    performRestart(true /* start */, reason);
	...
    mInstrumentation.callActivityOnResume(this);
    ...
    onPostResume();
   	...
}

public void callActivityOnResume(Activity activity) {
    activity.mResumed = true;
    activity.onResume();
    ...
}
```

同样只看重点。首先会调用performRestart 方法，这个方法内部会判断是否需要执行 onRestart 方法和 onStart 方法，如果是从别的 activity返回这里肯定要执行的。然后使用 Instrumentation 来回调 Activity 的onResume 方法。当onResume 回调完成后，会再调用 onPostResume 方法。

到这里关于 handleResumeActivity 的方法就讲完了，为什么在 onResume 甚至 onPostResume 方法被回调的时候界面尚未显示，也有了更加深刻的认识。

## 从系统设计看 Activity与其生命周期

我认为，每一个知识，都是在具体场景下为了解决具体的问题，通过权衡各种条件设计出来的。学习了一个知识之后，需要反过来思考一下这一块知识的底层设计思想是什么，他需要解决什么问题，权衡了什么条件，通过不断地思考从更高的角度来看待每一个知识点。

要理解生命周期地设计，首先要理解 Activity本身，想一下，如果没有 Activity，那么我们该如何编写程序？**Activity 类是Android 应用的关键组件，而 Activity的启动和组合方式则是该平台应用模型的基本组成部分**。

功能模块的应用模型从 main 方法进入主功能模块，而 Android 程序从 ActivityThread 的mai'n 方法开始，接收AMS的调度启动 LaunchActiivty，也就是我们在 AndroidManifest 中配置 为 main 的activity，当应用启动的时候，就会首先打开这个 Activity。那么第一个界面被打开，其他的界面就根据用户的操作来依次跳转了。

那如何做到每个界面之间彼此解耦，各自的显示不发生混乱、界面之前的跳转有条不紊等等？这些工作，官方都帮我们做好了，Activity就是在这个设计思想下开发出来的。我们在 Activity上开发的时候，就已经沿用了这种设计思想，当我们开发一个 App 的时候，最开始要考虑的，是界面如何设计。设计好界面之后，就是考虑如何开发每个界面了，那我们如何自定义好每一个界面？如何根据我们的需求去设计每个界面的功能？Activity并没有main 方法，我们的代码该写在哪里被执行？答案就是：**生命周期回调方法**。

到这里，你应该可以理解为什么启动 Activity 并不是一句 new 就可以解决的吧？Activity 承担的责任非常多，需要初始化的逻辑也非常多。当 Activity被启动，他会根据自身的启动情况，来回调不同的生命周期方法。其中**承担初始化整个界面以及各个功能组件的初始化任务**的就是onCreate 方法。有点类似于我们功能模块的入口函数，在这里我们通过 setContentView 来设计我们界面的布局，通过 setOnClickListener 来给每个 view 设置监听等等。在 MVVM 设计模式中，还需要初始化  viewModel、绑定数据等等。这是生命周期的第一个非常重要的意义所在。

而当界面的显示、退出，我们需要为之申请或者释放资源。如上面举到的相机例子，我在微信扫一扫申请了相机资源，如果进入到后台的时候没有释放资源，那么打开系统相机就无法使用了，资源被占领了。因此，生命周期的另一个作用，就是：**做好资源的申请和释放，避免内存泄露**。

这一部分的重点就是理解**Android应用程序是以 Activity为基本组成部分的应用模型**这个点。当界面的启动以及不同界面之间进行切换的时候，也就可以更加感知生命周期的作用了。

## 最后

关于 Activity 生命周期的内容，在一片文章讲解完成，是不现实的。当研究的越深，涉及到的内容就越多。每个知识点就是像瓜藤架上的一个瓜，如果是单纯的摘瓜，那他就是一个瓜；如果顺藤往外拔，整个架子都会被扯出来。























