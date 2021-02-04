# Android 全面解析之 WIndow 机制

Window，可能更多的认识是 Windows 系统的窗口。在 Windows 系统上，我们可以多个窗口同时运行，每个窗口代表着一个应用程序。但是在安卓系统上并没有这个东西，但是 android 不是由小窗口模式吗？但是那是 android 中 window 的一种表现方式，但是手机屏幕终究不能和电脑相比，因为屏幕太小了，小到只能操作一款应用，多个窗口就显得非常不习惯，所以 Android 上关于窗口方面的知识读者接触不多。

Android 框架层意义上的 Window 和我们认识的 Window 其实不一样。我们日常最直观的，每个应用界面，都有一个应用级别的 Window。再例如 popupwindow、Toast、dialog、menu 都是需要创建 window 来实现。所以其实 window 我们一直都见到，只是不知道那就是 window。了解 window 的机制原理，可以更好地了解 window，进而更好地了解 Android 是怎么管理屏幕上的 view。这样，当我们需要使用 dialog 或者 popupWindow  的时候，可以懂得他背后究竟做了什么，才能够更好的运用 dialog、popupWindow等

## 什么是 Window 机制

先假设没有 Window，会发生什么？

我们看到的界面 UI 是 view，如我们的应用布局，更简单是一个 Button。假如屏幕上现在有一个 Button,如图1，现在往屏幕中间添加一个 Text View，那么最终的结果是图2，还是图3：

![](../picture/window-01.image)

在上图的图2中，我们我要实现点击 text view 执行它的监听事件逻辑，点击不是 text view 的区域让 text view 消失，需要怎么实现呢？读者可能会说，我们可以在 Activity 中添加这部分的逻辑，那如果我们需要让一个悬浮框在所有界面显示呢，如上文所说的悬浮框，两个不同应用的 view，怎么确定他们的显示次序？又例如我们需要弹出一个 dialog 来提示用户，怎么样可以让 dialog 永远处于顶层呢？包括显示 dialog 期间应用弹出的如 popupWindow 必须显示在 dialog 的底下，但 toast 又必须显示在 dialog 上面。

很明显，我们的屏幕可以允许多个应用同时显示非常多的 view，他们的显示次序或者说显示高度是不一样的，如果没有一个统一的管理者，那么每一家应用都想要显示在最顶层，那么屏幕上的 view 就会非常的乱。

同时，当我们点击屏幕时，这个触摸事件应该传给哪个 view？很明显我们都知道应该传给最上层的 view，但是接受事件的是屏幕，是另一个系统服务，它怎么知道触摸位置的最上层是哪个 view 呢？即使知道，它又怎么把这个事件准确的传给它呢？

为了解决这些问题，需要有一个管理者来统一管理屏幕上的显示的 view，才能让程序有条不紊的走下去，而这，就是 Android 中的 window 机制。

> **window 机制就是为了管理屏幕上的 view 的显示以及触摸事件的传递问题**

#### 什么是 window？

在 Android 的 window 机制中，每个 view 树都可以看作是一个 window。为什么不是 view 呢？因为 view 树中每个 view 的显示次序是固定的，例如我们的 Activity 布局，每一个控件的显示都是已经安排好的，对于 window 来说，属于“不可再分割的 view”。

> 什么是 view 树？例如你在布局中给 Activity 设置了一个布局 xml，那么最顶层的布局如 LinearLayout 就是 view树的根，它包含的所有 view 就都是该 view 树的节点，所以这个 view 树就对应一个 window。
>
> 举几个例子：
>
> - 我们在添加 dialog 的时候，需要给它设置 view ，那么这个 view 它是不属于 activity 的布局内的，是通过 WindowManager 添加到屏幕上的，不属于 activity 的 view 树内，所以这个dialog 是一个独立的 view 树，所以它是一个 window。
> - PopupWindow 它也对应一个 window，因为它也是通过 windowManager 添加上去的，不属于 Activity 的 view 树。
> - 当我们使用 window Manager 在屏幕上添加的任何 view 都不属于 Activity 的布局 view 树，即使只添加一个 button

view 树（后面使用的 view 代称，后面说的 view 都是 view树）是 window 机制的操作单位，每一个view 对应一个 window，**view 是 window 的存在形式，window 是 view  的载体**，我们平时看到的应用界面、dialog、popupWindow 以及上面描述的悬浮框，都是 **window 的表现形式**。注意，我们看到的不是 window，而是 view。**Window 是 view 的管理者，同时也是 view  的载体。它是一个抽象的概念，本身并不存在，view 是 window 的表现形式**。这里的不存在，指的是我们在屏幕上是看不到 window 的，它不像 window 系统的窗口时可以看到的。

但是在 Android 中我们是无法感知的，我们只能看到 view，无法看到 window，window 是控制 view 怎么显示的管理者。每个成功的男人背后都有一个女人，每个 view 背后都有一个 window。

window 并不存在，它只是一个概念。举个例子，如班集体，就是一个概念，它的存在形式是这个班级的学生，当学生不存在这个班集体即使不存在的，但是它的好处就是得到了一个新的概念，我们可以以班级为单位来安排活动。因它不存在，也就很难从源码中找到它的痕迹，window 的操作单位都是 view，如果要说它在源码中的存在形式，目前的认知就是在 windowManagerService 中每一个 view 对应一个 window Status。WindowManagerService 在后面会进行讲解。读者可以慢慢的思考以下这个抽象的概念，后面会慢慢深入的帮助理解。

> - view 是 window 的存在形式，window 是 view 的载体
> - window 是view 的管理者。同时也是 view 的载体。它是一个抽象的概念，本身并不存在，view 是 window 的存在形式

思考：Android 中不是还有一个抽象类叫做 window 还有一个 PhoneWindow 实现类吗？他们不就是 window 的存在形式，为什么说 window 是抽象不存在的？

## Window 的相关属性

在了解 window 的操作流程之前，先补充说明一下 window 的相关属性

#### window 的type 属性

前面我们讲到的 window 机制解决的一个问题就是 view  的显示次序问题，这个属性就决定了 window 的显示次序。window 是有分类的，不同类别的显示高度范围不同，例如我把1-1000m 高度称为低空，1001-2000m高度称为中空，2000以上称为高空。window 也是一样按照高度范围进行分类，它有一个变量 z-Order，决定了 window 的高度。window 一共可分为三类：

- 应用程序窗口：应用程序窗口一般位于最底层，Z-Order 在1-99
- 子窗口：子窗口一般是显示在应用窗口之上，Z-Order 在1000-1999
- 系统级窗口：系统级窗口一般位于最顶层，不会被其他的 window 遮住，如 Toast，Z-Order 在 2000-2999.**如果要弹出自定义系统级窗口需要动态申请权限**。

Z-Order越大，window 越靠近用户，也就显示越高，高度高的 window 会覆盖高度低的 window。

window 的 type 属性就是 Z-Order 的值，我们可以给 window  的 type 属性赋值来决定 window 的高度。系统为我们三类 window 都预设了静态常量，如下：

- 应用级 window

- ```java
  // 应用程序 Window 的开始值
  public static final int FIRST_APPLICATION_WINDOW = 1;
    
  // 应用程序 Window 的基础值
  public static final int TYPE_BASE_APPLICATION = 1;
    
  // 普通的应用程序
  public static final int TYPE_APPLICATION = 2;
    
  // 特殊的应用程序窗口，当程序可以显示 Window 之前使用这个 Window 来显示一些东西
  public static final int TYPE_APPLICATION_STARTING = 3;
    
  // TYPE_APPLICATION 的变体，在应用程序显示之前，WindowManager 会等待这个 Window 绘制完毕
  public static final int TYPE_DRAWN_APPLICATION = 4;
    
  // 应用程序 Window 的结束值
  public static final int LAST_APPLICATION_WINDOW = 99;
  ```

- 子 window

- ```java
  // 子 Window 类型的开始值
  public static final int FIRST_SUB_WINDOW = 1000;
    
  // 应用程序 Window 顶部的面板。这些 Window 出现在其附加 Window 的顶部。
  public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;
    
  // 用于显示媒体(如视频)的 Window。这些 Window 出现在其附加 Window 的后面。
  public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;
    
  // 应用程序 Window 顶部的子面板。这些 Window 出现在其附加 Window 和任何Window的顶部
  public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;
    
  // 当前Window的布局和顶级Window布局相同时，不能作为子代的容器
  public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;
    
  // 用显示媒体 Window 覆盖顶部的 Window， 这是系统隐藏的 API
  public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW + 4;
    
  // 子面板在应用程序Window的顶部，这些Window显示在其附加Window的顶部， 这是系统隐藏的 API
  public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;
    
  // 子 Window 类型的结束值
  public static final int LAST_SUB_WINDOW = 1999;
  ```

- 系统级 window

- ```java
  // 系统Window类型的开始值
  public static final int FIRST_SYSTEM_WINDOW = 2000;
    
  // 系统状态栏，只能有一个状态栏，它被放置在屏幕的顶部，所有其他窗口都向下移动
  public static final int TYPE_STATUS_BAR = FIRST_SYSTEM_WINDOW;
    
  // 系统搜索窗口，只能有一个搜索栏，它被放置在屏幕的顶部
  public static final int TYPE_SEARCH_BAR = FIRST_SYSTEM_WINDOW+1;
    
  // 已经从系统中被移除，可以使用 TYPE_KEYGUARD_DIALOG 代替
  public static final int TYPE_KEYGUARD = FIRST_SYSTEM_WINDOW+4;
    
  // 系统对话框窗口
  public static final int TYPE_SYSTEM_DIALOG = FIRST_SYSTEM_WINDOW+8;
    
  // 锁屏时显示的对话框
  public static final int TYPE_KEYGUARD_DIALOG = FIRST_SYSTEM_WINDOW+9;
    
  // 输入法窗口，位于普通 UI 之上，应用程序可重新布局以免被此窗口覆盖
  public static final int TYPE_INPUT_METHOD = FIRST_SYSTEM_WINDOW+11;
    
  // 输入法对话框，显示于当前输入法窗口之上
  public static final int TYPE_INPUT_METHOD_DIALOG= FIRST_SYSTEM_WINDOW+12;
    
  // 墙纸
  public static final int TYPE_WALLPAPER = FIRST_SYSTEM_WINDOW+13;
    
  // 状态栏的滑动面板
  public static final int TYPE_STATUS_BAR_PANEL = FIRST_SYSTEM_WINDOW+14;
    
  // 应用程序叠加窗口显示在所有窗口之上
  public static final int TYPE_APPLICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 38;
    
  // 系统Window类型的结束值
  public static final int LAST_SYSTEM_WINDOW = 2999;
  ```

  ### Window 的flags 参数

  flag 标志控制 window 的东西比较多，很多资料的描述是“控制 window 的显示”，但我觉得不够准确。flag 控制的范围包括了：各种情景下的显示逻辑（锁屏，游戏等）还有触控事件的处理逻辑。控制显示确实是它的很大部分功能，但是不是全部。下面看下一些常用的 flag，就知道 flag 的功能了：

  ```java
  // 当 Window 可见时允许锁屏
  public static final int FLAG_ALLOW_LOCK_WHILE_SCREEN_ON = 0x00000001;
  
  // Window 后面的内容都变暗
  public static final int FLAG_DIM_BEHIND = 0x00000002;
  
  // Window 不能获得输入焦点，即不接受任何按键或按钮事件，例如该 Window 上 有 EditView，点击 EditView 是 不会弹出软键盘的
  // Window 范围外的事件依旧为原窗口处理；例如点击该窗口外的view，依然会有响应。另外只要设置了此Flag，都将会启用FLAG_NOT_TOUCH_MODAL
  public static final int FLAG_NOT_FOCUSABLE = 0x00000008;
  
  // 设置了该 Flag,将 Window 之外的按键事件发送给后面的 Window 处理, 而自己只会处理 Window 区域内的触摸事件
  // Window 之外的 view 也是可以响应 touch 事件。
  public static final int FLAG_NOT_TOUCH_MODAL  = 0x00000020;
  
  // 设置了该Flag，表示该 Window 将不会接受任何 touch 事件，例如点击该 Window 不会有响应，只会传给下面有聚焦的窗口。
  public static final int FLAG_NOT_TOUCHABLE      = 0x00000010;
  
  // 只要 Window 可见时屏幕就会一直亮着
  public static final int FLAG_KEEP_SCREEN_ON     = 0x00000080;
  
  // 允许 Window 占满整个屏幕
  public static final int FLAG_LAYOUT_IN_SCREEN   = 0x00000100;
  
  // 允许 Window 超过屏幕之外
  public static final int FLAG_LAYOUT_NO_LIMITS   = 0x00000200;
  
  // 全屏显示，隐藏所有的 Window 装饰，比如在游戏、播放器中的全屏显示
  public static final int FLAG_FULLSCREEN      = 0x00000400;
  
  // 表示比FLAG_FULLSCREEN低一级，会显示状态栏
  public static final int FLAG_FORCE_NOT_FULLSCREEN   = 0x00000800;
  
  // 当用户的脸贴近屏幕时（比如打电话），不会去响应此事件
  public static final int FLAG_IGNORE_CHEEK_PRESSES    = 0x00008000;
  
  // 则当按键动作发生在 Window 之外时，将接收到一个MotionEvent.ACTION_OUTSIDE事件。
  public static final int FLAG_WATCH_OUTSIDE_TOUCH = 0x00040000;
  
  @Deprecated
  // 窗口可以在锁屏的 Window 之上显示, 使用 Activity#setShowWhenLocked(boolean) 方法代替
  public static final int FLAG_SHOW_WHEN_LOCKED = 0x00080000;
  
  // 表示负责绘制系统栏背景。如果设置，系统栏将以透明背景绘制，
  // 此 Window 中的相应区域将填充 Window＃getStatusBarColor（）和 Window＃getNavigationBarColor（）中指定的颜色。
  public static final int FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS = 0x80000000;
  
  // 表示要求系统壁纸显示在该 Window 后面，Window 表面必须是半透明的，才能真正看到它背后的壁纸
  public static final int FLAG_SHOW_WALLPAPER = 0x00100000;
  ```

  ### Window 的 solfInputMode 属性

  这一部分就是当软键盘弹起来的时候，window 的处理逻辑，这在日常中也经常遇到，如：我们在微信聊天的时候，点击输入框，当软键盘弹起来的时候输入框也会被顶上去。如果你不想被顶上去，也可以设置为被软键盘覆盖。下面介绍一下常见的属性：

  ```java
  // 没有指定状态，系统会选择一个合适的状态或者依赖于主题的配置
  public static final int SOFT_INPUT_STATE_UNCHANGED = 1;
  
  // 当用户进入该窗口时，隐藏软键盘
  public static final int SOFT_INPUT_STATE_HIDDEN = 2;
  
  // 当窗口获取焦点时，隐藏软键盘
  public static final int SOFT_INPUT_STATE_ALWAYS_HIDDEN = 3;
  
  // 当用户进入窗口时，显示软键盘
  public static final int SOFT_INPUT_STATE_VISIBLE = 4;
  
  // 当窗口获取焦点时，显示软键盘
  public static final int SOFT_INPUT_STATE_ALWAYS_VISIBLE = 5;
  
  // window会调整大小以适应软键盘窗口
  public static final int SOFT_INPUT_MASK_ADJUST = 0xf0;
  
  // 没有指定状态,系统会选择一个合适的状态或依赖于主题的设置
  public static final int SOFT_INPUT_ADJUST_UNSPECIFIED = 0x00;
  
  // 当软键盘弹出时，窗口会调整大小,例如点击一个EditView，整个layout都将平移可见且处于软件盘的上方
  // 同样的该模式不能与SOFT_INPUT_ADJUST_PAN结合使用；
  // 如果窗口的布局参数标志包含FLAG_FULLSCREEN，则将忽略这个值，窗口不会调整大小，但会保持全屏。
  public static final int SOFT_INPUT_ADJUST_RESIZE = 0x10;
  
  // 当软键盘弹出时，窗口不需要调整大小, 要确保输入焦点是可见的,
  // 例如有两个EditView的输入框，一个为Ev1，一个为Ev2，当你点击Ev1想要输入数据时，当前的Ev1的输入框会移到软键盘上方
  // 该模式不能与SOFT_INPUT_ADJUST_RESIZE结合使用
  public static final int SOFT_INPUT_ADJUST_PAN = 0x20;
  
  // 将不会调整大小，直接覆盖在window上
  public static final int SOFT_INPUT_ADJUST_NOTHING = 0x30;
  ```

  ### Window 的其他属性

  上面的三个属性是 Window 比较重要也是比较复杂的三个，初次之外还有几个日常经常使用的属性：

  - x 与 y属性：指定 window 的位置
  - alpha：window 的透明度
  - gravity：window 在屏幕中的位置，使用的是Gravity类的常量
  - format：window 的像素点格式，值定义在PixelFormat 中

  ------

  ### 如何给 window 属性赋值

  window 的属性的常量值大部分存储在 WindowManager.LayoutParams 类中，我们可以通过这个类获得这些常量。当然还有 Gravity 类和 PixelFormat 类等。

  一般情况下我们会通过以下方式来往屏幕中添加一个 Window：

  ```java
  // 在Activity中调用
  WindowManager.LayoutParams windowParams = new WindowManager.LayoutParams();
  windParams.flags = WindowManager.LayoutParams.FLAG_FULLSCREEN;
  TextView view = new TextView(this);
  getWindowManager.addview(view,windowParams);
  ```

  我们可以直接给 WindowManager.LayoutParams 对象设置属性。

  第二种赋值方法是直接给 Window 赋值，如

  ```java
  getWindow().flags = WindowManager.LayoutParams.FLAG_FULLSCREEN;
  ```

  除此之外，window 的 solfInputMode 属性比较特殊，它可以直接在AndroidManifest 中指定，如下：

  ```javascript
   <activity android:windowSoftInputMode="adjustNothing" />
  ```

  最后总结一下：

  > - window的重要属性有type、flags、solfInputMode、gravity等
  > - 我们可以通过不同的方式给window属性赋值
  > - 没必要去全部记下来，等遇到需求再去寻找对应的常量即可

------

## Window 的添加过程

通过理解源码之后，可以对之前的理论理解更加的透彻。window 的添加过程，指的是我们通过 WindowManagerImpl 的 addView 方法来添加 window 的过程。

想要添加一个 window，我们知道首先得有 view 和 WindowManager.LayoutParams 对象，才能去创建一个 window，这是我们常见的代码：

```java
Button button = new Button(this);
WindowManager.LayoutParams windowParams = new WindowManager.LayoutParams();
// 这里对windowParam进行初始化
windowParam.addFlags...
// 获得应用PhoneWindow的WindowManager对象进行添加window
getWindowManager.addView(button,windowParams);
```

然后接下来我们进入 addView 方法中查看。我们知道这个 windowmanager 的实现类是 WindowManagerImpl，上面讲过，进入它的 addView 方法看一看：

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

可以发现它把逻辑交给mGlobal 去处理了。这个 mGlobal 是 WindowManagerGlobal，是一个全局单例，是 WindowManager 接口的具体逻辑实现。这里运用的是桥接模式。那我们进入 WindowManagerGlobal 的方法看一下：

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    // 首先判断参数是否合法
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }
    
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    // 如果是子窗口，会对其做参数的调整
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }
    
	synchronized (mLock) {
        ...
        // 这里新建了一个viewRootImpl，并设置参数
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);

        // 添加到windowManagerGlobal的三个重要list中，后面会讲到
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // 最后通过viewRootImpl来添加window
        try {
            root.setView(view, wparams, panelParentView);
        } 
        ...
    }  
}
```

代码有点长，一步步看：

- 首先对参数的合法性进行检查
- 然后判断该窗口是不是子窗口，如果是的话需要对窗口进行调整，这个好理解，子窗口要跟随父窗口的特性。
- 接着新建viewRootImpl对象，并把view、viewRootImpl、params三个对象添加到三个list中进行保存
- 最后通过viewRootImpl来进行添加

> 补充一点关于WindowManagerGlobal中的三个list，他们分别是：
>
> ```java
> private final ArrayList<View> mViews = new ArrayList<View>();
> private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
> private final ArrayList<WindowManager.LayoutParams> mParams =
>      new ArrayList<WindowManager.LayoutParams>();
> ```
>
> 每一个window所对应的这三个对象都会保存在这里，之后对window的一些操作就可以直接来这里取对象了。当window被删除的时候，这些对象也会被从list中移除。

可以看到添加 window 的逻辑就交给 ViewRootImpl 了。viewRootImpl 是 window 和 view  之间的桥梁，viewRootImpl 可以处理两边的对象，然后连接起来。下面看一下 viewRootImpl 是怎么处理的：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        ...
        try {
            mOrigWindowType = mWindowAttributes.type;
            mAttachInfo.mRecomputeGlobalAttributes = true;
            collectViewAttributes();
            // 这里调用了windowSession的方法，调用wms的方法，把添加window的逻辑交给wms
            res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                    getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                    mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                    mTempInsets);
            setFrame(mTmpFrame);
        } 
        ...
    }
}
```

viewRootImpl 的逻辑很多，重要的是调用了 mWindowSession 的方法调用了 WMS 的方法。这个 mWindowSession 很重要，重点学习一下。

> mWindowSession是一个IWindowSession对象，看到这个命名很快地可以像到这里用了AIDL跨进程通信。IWindowSession是一个IBinder接口，他的具体实现类在WindowManagerService，本地的mWindowSession只是一个Binder对象，通过这个mWindowSession就可以直接调用WMS的方法进行跨进程通信。
>
> 那这个mWindowSession是从哪里来的呢？我们到viewRootImpl的构造器方法中看一下：
>
> ```java
> public ViewRootImpl(Context context, Display display) {
> 	...
>  	mWindowSession = WindowManagerGlobal.getWindowSession();
>  	...
> }
> ```
>
> 可以看到这个session对象是来自WindowManagerGlobal。再深入看一下：
>
> ```java
> public static IWindowSession getWindowSession() {
>  synchronized (WindowManagerGlobal.class) {
>      if (sWindowSession == null) {
>          try {
>              ...
>              sWindowSession = windowManager.openSession(
>                      new IWindowSessionCallback.Stub() {
>                          ...
>                      });
>          } 
>          ...
>      }
>      return sWindowSession;
>  }
> }
> ```
>
> 这熟悉的代码格式，可以看出来这个session是一个单例，也就是**整个应用的所有viewRootImpl的windowSession都是同一个，也就是一个应用只有一个windowSession**。对于wms而言，他是服务于多个应用的，如果说每个viewRootImpl整一个session，那他的任务就太重了。WMS的对象单位是应用，他**在内部给每个应用session分配了一些数据结构如list，用于保存每个应用的window以及对应的viewRootImpl**。当需要操作view的时候，通过session直接找到viewRootImpl就可以操作了。

后面的逻辑就交给了 WMS 去处理了，WNS 就会创建 window，然后结合参数计算 window 的高度等等，最后使用 viewRootImpl 进行绘制。这后面的代码逻辑就不讲了，这是深入到 WMS 的内容。

我们知道 windowManager 接口是继承 viewManager 接口的，viewManager 还有另外两个接口：removeView、updateView。

最后做个总结：

> window的添加过程是通过PhoneWindow对应的WindowManagerImpl来添加window，内部会调用WindowManagerGlobal来实现。WindowManagerGlobal会使用viewRootImpl来进行跨进程通信让WMS执行创建window的业务。
>
> 每个应用都有一个windowSession，用于负责和WMS的通信，如ApplicationThread与AMS的通信。

## Window 机制的关键类

下面看一张图：

![](../picture/window-02.image)

这基本上是这篇学习涉及到的所有的关键类（图中绿色的 window 并不是一个类，而是真正意义上的 window）

#### window 相关

window 的实现类只有一个：PhoneWindow，它继承自 Window 抽象类。

**Window Manager 相关**

顾名思义，windowManager 就是 Window 管理类。这一部分的关键类有 WindowManager，viewManager, winodwManagerImpl，WindowManagerGlobal。WindowManager 是一个接口，继承自 Viewmanager。ViewManager 中包含了我们非常熟悉的三个接口：addView，removeView，updateView。WindowManagerImpl 和 PhoneWindow 是成对出现的，前者负责管理后者。WindowManagerImpl 是 windowManager 的实现类，但它本身并没有真正实现逻辑，而是交给了 WindowManagerGlobal。WindowManagerGlobal 是全局单例，windowmanagerImpl 内部使用桥接模式，它是 windowManager 接口逻辑的真正实现。

### View

这里有个很关键的类：ViewRootImpl。每个view 树都会有一个。当我使用 windowManager 的 add View 方法时，就会创建一个 ViewRootImpl。ViewRootImpl 的作用很关键：

- 负责连接 view 和 window 的桥梁事务
- 负责和 WindowManagerService 的联系
- 负责管理和绘制 view 树
- 事件的中转站

每个 window 都会有一个 ViewRootImpl，viewRootImpl 是负责绘制这个 view 树 和 window 与 view 的桥梁，每个 window 都会有一个 ViewRootImpl。

### WindowManagerService

这个是 window 的真正管理者，类似于 AMS（ActivityManagerService）管理四大组件。所有的 window 创建最终都要经过 WindowManagerService。整个 Android 的 window 机制中，WMS 绝对是核心，它决定了屏幕所有的 window 该如何显示、如何分发点击事件等等。

### Window 与 PhoneWindow 的关系

> 解释一下标题，window 是指 window 机制中 window 这个概念。而 PhoneWindow 是指 PhoneWindow 这个类。后面再讲的时候，如果是指类，我会再后面加个类字。如window 是指 window 这个概念，window 类是指 window 这个抽象类。

还记得在讲 Window 概念的时候留了一个思考吗？

> 思考：Android中不是有一个抽象类叫做window还有一个PhoneWindow实现类吗，他们不就是window的存在形式，为什么说window是抽象不存在的

这里我再抛出几个问题：

- 有一些资料认为PhoneWindow就是window，是view容器，负责管理容器内的view，windowManagerImpl可以往里面添加view，如上面我们讲过的addView方法。但是，同时它又说每个window对应一个viewRootImpl，但却没解释为什么每次addView都会新建一个viewRootImpl，前后矛盾。
- 有一些资料也是认为PhoneWindow是window，但是他说addView方法不是添加view而是添加window，同时拿这个方法的名字作为论据证明view就是window，但是他没解释为什么在使用addView方法创建window的过程却没有创建PhoneWindow对象。

我们一步步来看。我们首先先看一下源码中对于 window 抽象类的注释：

```java
 Abstract base class for a top-level window look and behavior policy.  An
 instance of this class should be used as the top-level view added to the
 window manager. It provides standard UI policies such as a background, title
 area, default key processing, etc.
     
顶层窗口外观和行为策略的抽象基类。此类的实例应用作添加到窗口管理器的顶层视图。
它提供标准的UI策略，如背景、标题区域、默认键处理等。
```

大概意思就是：这个类是顶层窗口的抽象基类，顶级窗口必须继承它，它负责窗口的外观如背景、标题、默认按键处理等。这个类的实例被添加到 windowmanager 中，让 windowManager 对它进行管理。PhoneWindow 是一个 top-level window（顶级窗口），它被添加到顶级窗口管理器的顶层视图，其他的 window，都需要添加到这个顶层视图中，所以准确的说，PhoneWindow 并不是 view 容器，而是 window 容器。

那 PhoneWindow 的存在意义是什么？

第一、提供 DecorView 模板。如下图：

![](../picture/window-03.image)

我们的 Activity 是通过 setContentView 把布局设置到 DecorView 中，那么 DecorView 本身的布局，就成为了Activity 界面的背景。同时 DecorView 分为标题栏和内容两部分，所以也可以设置标题栏。同时，由于我们的界面是添加在 DecorView 中，属于 DecorView 的一部分。那么对于 DecorView 的 window 属性设置也会对我们的布局界面生效。还记得谷歌的官方对 window类的注释的最后一句话吗：`它提供标准的UI策略，如背景、标题区域、默认键处理等。`这些都可以通过DecorView实现，这是PhoneWindow的第一个作用。

第二、抽离 Acitivity 中关于 window 的逻辑。Activity 中的逻辑非常的多，如果所有的事情都自己做，那么会造成本身代码很臃肿。阅读过 Activity 启动的代码，AMS 也通过 ActivityStarter 这个类来抽离启动 Activity 的逻辑。这样关于 window 的事情，就交给 PhoneWindow 去处理了。（事实上，Activity 调用的是 WindowManagerImpl , 但因 PhoneWindow 和 WindowManagerImpl 两者是成对存在的，他们共同处理 window 的相关事务，所以这里简单写成了交给 Phone Window 处理）。当 Activity 需要添加布局的时候，只需要一句 setContentView，调用了 Phone Window 的 setContentView方法，就把布局设置到屏幕上去了。具体怎么完成，Activity 也不必管。

第三、限制组件添加 window 的权限。PhoneWindow 内部有一个 token 属性，用于验证一个 PhoneWindow 是否允许添加 window。在 Activity 创建 PhoneWindow 的时候，就会把从 AMS 传过来的 token 赋值给它，从而它有了添加 token 的权限。而其他的 PhoneWindow 则没有这个权限，因此无法添加 token。

总结一下：

> PhoneWindow本身不是真正意义上的window，他更多可以认为是辅助Activity操作window的工具类。
>
> windowManagerImpl并不是管理window的类，而是管理PhoneWindow的类。真正管理window的是WMS。
>
> PhoneWindow可以配合DecorView可以给其中的window按照一定的逻辑提供标准的UI策略
>
> PhoneWindow限制了不同的组件添加window的权限。

## 常见组件的 window 创建流程

上面讲的是通过 windowManagerImpl 创建 window 的过程，我们通过前面的讲解了解到，WindowManagerImpl 是管理 PhoneWindow的，他们是成对出现的，因而有两种创建 window 的方式：

- 已经存在的 PhoneWindow，直接通过 WindowManagerImpl 创建 Window
- PhoneWindow 尚未存在，先创建 PhoneWindow，再利用 WindowManagerImpl 来创建 Window

当我们在 Activity 中使用的 getWindowManager 方法获取到的就是应用的 PhoneWindow 对应的 WindowmangerImpl。下面讲一下不同组件是如何创建 window 的。

### Activity

如果阅读过 Activity 的启动流程，会知道 Activity 的启动最后来到了 ActivityThread 的 `handleLaunchActivity` 这个方法。

直接来看下这个方法的代码：

```java
public void handleLaunchActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    ...;
    // 这里对WindowManagerGlobal进行初始化
    WindowManagerGlobal.initialize();

   	// 启动Activity并回调activity的onCreate方法
    final Activity a = performLaunchActivity(r, customIntent);
    ...
}


private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    try {
        // 这里创建Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
		...
        if (activity != null) {
            ...
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            appContext.setOuterContext(activity);
            // 这里将window作为参数传到activity的attach方法中
            // 一般情况下这里window==null
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);  
            ...
            // 最后这里回调Activity的onCreate方法
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
        }
    
    ...
}
```

`handleLaunchActivity` 的代码中首先对 WindowManagerGlobal 进行初始化，然后调用了 `performLaunchActivity` 方法。代码很多，这里只截取了重要部分。首先会创建 Application 对象，然后再调用 Activity 的 attach 方法，把 window 作为参数传进去，最后回调 activity 的 onCreate 方法。所以这里最有可能创建 window 的方法就是 Activity 的 `attach` 方法了。进入看一下：

```java
final void attach(...,Context context,Window window, ...) {
    ...;
 	// 这里新建PhoneWindow对象，并对window进行初始化
	mWindow = new PhoneWindow(this, window, activityConfigCallback);
    // Activity实现window的callBack接口，把自己设置给window
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);    
    ...
    // 这里初始化window的WindowManager对象
	mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);        
}
```

同样只截取了重要代码，attach 方法参数非常多，只留下了 window 相关的参数。在这方法里首先利用传进来的 window 创建了 PhoneWindow。Activity 实现 window 的 callback 接口，可以把自己设置给 window 做观察者。当 window发生变化的时候可以通知 Activity。然后再创建 WindowManager 和 PhoneWindow 绑定再一起，这样我们就可以通过 windowManager 操作 PhoneWindow 了。（这里不是 setWindowManager 吗？windowManager 是什么时候创建的？）进入 `setWindowManager` 方法看一下：

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated;
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    // 这里创建了windowManager
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

这个方法里首先会获得到应用服务的 WindowManager（实现类也是 WindowManagerImpl)，然后通过这个应用服务的 Window Manager 创建了新的 WindowManager。

> 从这里可以看到是利用系统服务的windowManager来创建新的windowManagerImpl，因而这个应用所有的WindowManagerImpl都是同个内核windowManager，而创建出来的仅仅是包了个壳。

这样 PhoneWindow 和 WindowManagerImpl 就绑定再一起了。Activity 可以通过 WindowManagerImpl 来操作 PhoneWindow。

------

到这里 Activity 的 PhoneWindow 和 WindowManagerImpl 对象就创建完成了，接下来是如何把 Activity 的布局文件设置给 PhoneWindow 的呢？在上面讲到 Activity 的 attach 方法之后，会回调 Activity 的 onCreate方法，在 onCreate 方法我们会调用 `setContentView` 来设置布局，如下：

```java
public void setContentView(View view, ViewGroup.LayoutParams params) {
    getWindow().setContentView(view, params);
    initWindowDecorActionBar();
}
```

这里的 getWindow 就是获取到我们上面创建的 PhoneWindow 对象。我们继续看下去：

```java
// 注意他有多个重载的方法，要选择参数对应的方法
public void setContentView(int layoutResID) {
    // 创建DecorView
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        // 这里根据布局id加载布局
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        // 回调activity的方法
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

同样看重点代码：

- 首先看 decorView 创建了没有，没有的话创建 DecorView
- 把布局加载到 DecorView 中
- 回调 Activity 的 callBack 方法

> 这里补充一下什么是 DecorView。DecorView 是在 PhoneWindow 中预设好的一个布局，这个布局长这样：
>
> 它是一个垂直排列的布局，上面是 actionbar,下面是 ContentView，它是一个 Framelayout。我们的 Activity 布局就加载到 ContentView 里进行显示。所以 DecorView 是 Activity 布局最顶层的 ViewGroup

看下是怎么初始化 DecorView 的：

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        // 这里创建了DecorView
        mDecor = generateDecor(-1);
        ...
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        // 对DecorView进行初始化，得到ContentView
        mContentParent = generateLayout(mDecor);
        ...
    }
}
```

`installDecor`方法中主要是新建一个DecorView对象，然后加载预设好的布局对DecorView进行初始化，（预设好的布局就是上面讲述的布局）并获取到这个预设布局的ContentView。好了然后我们再回到window的setContentView方法中，初始化了DecorView之后，把Activity布局加载到DecorView的ContentView中如下代码：

```java
// 注意他有多个重载的方法，要选择参数对应的方法
public void setContentView(int layoutResID) {
    ...
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        // 这里根据布局id加载布局
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    ...
   	mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        // 回调activity的方法
        cb.onContentChanged();
    }
}
```

所以可以看到Activitiy的布局确实是添加到DecorView的ContentView中，这也是为什么onCreate中使用的是setContentView而不是setView。最后会回调Activity的方法告诉Activity，DecorView已经创建并初始化完成了

------

到这里 DecorView 创建完成了，但还缺少最重要的一步：**把 DecorView** 作为 window 添加到屏幕上。从上面的介绍我们知道添加 window 需要用到 WindowManagerImpl 的 addView 方法。这一步是在 ActivityThread 的`handleResumeActivity` 方法中执行：

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward, String reason) {
    // 调用Activity的onResume方法
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ...
    // 让decorView显示到屏幕上
	if (r.activity.mVisibleFromClient) {
        r.activity.makeVisible();
  	}
```

这一步方法有两个重点：回调 onResume 方法，把 devorView 添加到屏幕上。我们看一下 `makeVisible` 方法做了什么：

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

是不是非常熟悉？直接调用WindowManagerImpl的addView方法来吧decorView添加到屏幕上，至此，我们的Activity界面就会显示在屏幕上了。

------

总结一下：

> - 从Activity的启动流程可以得到Activity创建Window的过程
>
> - 创建PhoneWindow -> 创建WindowManager -> 创建decorView -> 利用windowManager把DecorView显示到屏幕上
>
> - 回调onResume方法的时候，DecorView还没有被添加到屏幕，所以当onResume被回调，指的是屏幕即将到显示，而不是已经显示

### PopupWindow

popupWindow 日常使用的也比较多，最常见的需求是弹一个菜单出来等。popupWindow 也是利用 WindowManager 来往屏幕上添加 window，但，popupWindow 是依附于Activity 而存在的，当 Activity 未运行时，是无法弹出 popupWindow 的，通过源码可以知道，当调用 onResume 方法的时候，其实后续还有很多事情在做，这个时候 Activity 也是尚未完全启动，所以 PopupWindow 不能在 onCreate、onStart、onResume  方法中弹出。

弹出 popupWindow 的过程分为两个：创建 view；通过 windowManager 添加 window。首先看到 PopupWindow 的构造方法：

```java
public PopupWindow(View contentView, int width, int height, boolean focusable) {
    if (contentView != null) {
        mContext = contentView.getContext();
        mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
    }

    setContentView(contentView);
    setWidth(width);
    setHeight(height);
    setFocusable(focusable);
}
```

它有多个重载方法，但最终都会调用到这个有四个参数的方法。主要是前面的得到 Context 和根据 context 获得 WindowManager。

------

然后我们看到它的显示方法。显示方法有两个：`showAtLocation` 和 `showAsDropDown`。主要是处理显示的位置不同，其他都是相似的。我们看到第一个方法：

```java
public void showAtLocation(View parent, int gravity, int x, int y) {
    mParentRootView = new WeakReference<>(parent.getRootView());
    showAtLocation(parent.getWindowToken(), gravity, x, y);
}
```

逻辑很简单，父 View 的根布局存储了起来，然后调用另外的重载方法：

```java
public void showAtLocation(IBinder token, int gravity, int x, int y) {
    // 如果contentView是空直接返回
    if (isShowing() || mContentView == null) {
        return;
    }

    TransitionManager.endTransitions(mDecorView);
    detachFromAnchor();
    mIsShowing = true;
    mIsDropdown = false;
    mGravity = gravity;
	// 得到WindowManager.LayoutParams对象
    final WindowManager.LayoutParams p = createPopupLayoutParams(token);
    // 做一些准备工作
    preparePopup(p);

    p.x = x;
    p.y = y;
	// 执行popupWindow显示工作
    invokePopup(p);
}
```

这个方法的逻辑主要有：

- 判断 contentView 是否为空或者是否进行显示
- 做一些准备工作
- 进行 popupWindow 显示工作

这里看一下它的准备工作做了什么：

```java
private void preparePopup(WindowManager.LayoutParams p) {
    ...  
    if (mBackground != null) {
        mBackgroundView = createBackgroundView(mContentView);
        mBackgroundView.setBackground(mBackground);
    } else {
        mBackgroundView = mContentView;
    }
	// 创建了DecorView
    // 注意，这里的DecorView并不是我们之前讲的DecorView，而是他的内部类：PopupDecorView
    mDecorView = createDecorView(mBackgroundView);
    mDecorView.setIsRootNamespace(true);
    ...
}
```

接下来看它的显示工作：

```java
private void invokePopup(WindowManager.LayoutParams p) {
    ...
   	// 调用windowManager添加window
    mWindowManager.addView(decorView, p);
    ...
}
```

到这里 popupWindow 就会被添加到屏幕上了。

最后总结一下：

> - 根据参数构建popupDecorView
> - 把popupDecorView添加到屏幕上

### Dialog

dialog 的创建过程和 Activity 比较像：创建 PhoneWindow、初始化 DecorView，添加 DecorView。这里简单讲一下。首先看到它的构造方法：

```java
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    ...
    // 获取windowManager
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
	// 构造PhoneWindow
    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    // 初始化PhoneWindow
    w.setCallback(this);
    w.setOnWindowDismissedCallback(this);
    w.setOnWindowSwipeDismissedCallback(() -> {
        if (mCancelable) {
            cancel();
        }
    });
    w.setWindowManager(mWindowManager, null, null);
    w.setGravity(Gravity.CENTER);
    mListenersHandler = new ListenersHandler(this);
}
```

这里和前面的 Activity 创建过程非常像，但是有个重点需要注意 mWindowManager 其实是 Activity 的 WindowManager，这里的 context 一般是 Activity（实际上也只能是 Activity，非 Activity 会抛出异常），我们看到 activity 的 getSystemService 方法：

```java
public Object getSystemService(@ServiceName @NonNull String name) {
    if (getBaseContext() == null) {
        throw new IllegalStateException(
                "System services not available to Activities before onCreate()");
    }
	// 获取activity的windowManager
    if (WINDOW_SERVICE.equals(name)) {
        return mWindowManager;
    } else if (SEARCH_SERVICE.equals(name)) {
        ensureSearchManager();
        return mSearchManager;
    }
    return super.getSystemService(name);
}
```

可以看到这里的 windowManager 确实是 Activity 的 WindowManager。接下来看到它的 show 方法：

```java
public void show() {
   ...
    // 回调onStart方法，获取前面初始化好的decorview
    onStart();
    mDecor = mWindow.getDecorView();
    ...
    WindowManager.LayoutParams l = mWindow.getAttributes();
    ...
    // 利用windowManager来添加window    
    mWindowManager.addView(mDecor, l);
    ...
    mShowing = true;
    sendShowMessage();
}
```

注意这里的 mWindowManager 是 Activity 的 WindowManager，所以实际上，这里是添加到了 Activity 的 PhoneWindow 中。接下来的和前面的添加流程一样。

总结一下：

> dialog和popupWindow不同，dialog创建了新的PhoneWindow，使用了PhoneWindow的DecorView模板。而popupWindow没有
>
> dialog的显示层级数更高，会直接显示在Activity上面，在dialog后添加的popUpWindow也会显示在dialog下
>
> dialog的创建流程和activity非常像

## 从 Android 架构角度看 Window

前面我们介绍过关于 PhoneWindow 和 Window 之间的关系，了解到 PhoneWindow 其实不是Window，只是一个 window 容器。为什么 Google 要建一个不是 window 但却名字是 window 的类？要了解这个问题，先来回顾一下 Android 的 window 机制结构。

首先从 WindowManagerService 开始，我们知道 WMS 是 Window 的最终管理者，在 WMS 中为每一个应用持有一个 session，关于 session 前面我们讲过，每个应用都是全局单例，负责和 WMS 通信的 binder 对象。WMS 为每个 window 都建立了一个 windowStatus 对象，同一个应用的 window 使用同个 session 进行跨进程通信，结构大概如下：

![](../picture/window-04.image)

而负责与 WMS 通信的，是 viewRootImpl。前面我们讲过每个 view 树即为一个 window，viewRootImpl 负责和 WMS 进行通信，同时也负责 view 的绘制。如果把上面的图画仔细一点就是：

![](../picture/window-05.image)

图中每一个 windowStatus 对应一个 viewRootImpl，WMS 通过 viewRootImpl 来控制 view。这也就是 window 机制的管理结构。当我们需要添加 window 的时候，最终的逻辑实现是 WindowManagerGlobal，它的内部使用自己的 session 创建一个 ViewRootImpl，然后向 WMS 申请添加 window，结构图如下：

![](../picture/window-06.image)

windowManagerGlobal 使用自己的 IWindowSession 创建viewRootImpl，这个 IWindowSession 是全局单例。viewRootImpl 和 WMS 申请创建 window，然后 WMS 允许之后，再通知 viewRootImpl 绘制 view，同时 WMS 通过 windowStatus 存储了 viewRootImpl 的相关信息，这样如果 WMS 需要修改 view，直接通过 viewRootImpl 就可以修改 view 了。

从上面的描述中可以发现没有提及 PhoneWindow 和 WindowManagerImpl。这是因为**它们不属于 window 机制内的类，而是封装于 window 机制之上的框架 **。假设如果没有 PhoneWindow 和WindowManager 我们该如何添加一个 window？首先需要调用 WindowGlobal 获取 session，再创建 viewRootImpl，再访问 WMS，然后再利用 viewRootImpl 绘制 view，是不是很复杂，而这仅仅只是整体的步骤。而 WindowmanagerImpl 正是这个功能。它内部拥有 WIndowManageGlobal 单例，然后帮助我们完成一系列的步骤。同时， **windowManagerImpl 也是只有一个实例，其他的 windowManagerImpl 都是建立在 windowManagerImpl 单例上**。这一点在前面介绍过。

另外，上面讲到 PhoneWindow 并不是 window 而是一个辅助 Activity 管理的工具类，那为什么它不要命名为 windowUtils 呢？首先，PhoneWindow 这个类是 Google 给 window 机制进行更上一层的封装。PhoneWindow 内部拥有一个 DecorView,我们的布局 view 都是添加到 decorView 中的，因为我们可以通过给 devorView 设置背景，宽高度，标题栏，按键反馈等等，来间接给我们的布局view 设置。这样一来，Phone Window 的存在，向开发者屏蔽真正的 window，暴露给开发者一个”存在的” window。我们可以认为 PhoneWindow 就是一个 window，window 是 view 的容器。当我们需要在屏幕上添加 view 的时候，只需要获得应用 window 对应 的windowManagerImpl，然后直接调用 addView 方法添加 view即可。这里也可以解释为什么 windowmanager 的接口方法是 addView 而不是 addWindow，一是 window 确实是以 view 的存在形式。二是为了向开发者屏蔽真正的 window，让我们以为是在往 window 添加 view，window 是真实存在的东西。他们的关系如下：

![](../picture/window-07.image)

黄色部分属于 Google 提供给开发者的 window 框架，而绿色 是真正的 window 机制结构。通过 PhoneWindow 我们可以很方便的进行 window 操作，而不需了解底层是怎么工作的。Phone Window 的存在，更是让 window 的可见性得到了实现，让 window 变成了一个 “view容器”。

总结一下：

> Android内部的window机制与谷歌暴露给我们的api是不一样的，谷歌封装的目的是为了让我们更好地使用window。
>
> dialog、popupWindow等框架更是对具体场景进行更进一步的封装。
>
> 我们在了解window机制的时候，需要跳过应用层，看到window的本质，才能更好地帮助我们理解window。
>
> 在android的其他地方也是一样，利用封装向开发者屏蔽底层逻辑，让我们更好地运用。但如果我们需要了解他的机制的时候，就需要绕过这层封装，看到本质

参考文献：

- [直面底层：探索 Android 中的 Window](https://mp.weixin.qq.com/s/1j1lsciZFnmh5y_1_CUD8g)
- [直面底层：WindowManager 视图绑定以及体系结构](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650832409&idx=2&sn=a9f16b2134b0a4e7f34a315d6704a401&chksm=80b7aa87b7c02391036ad82825a4b9cbf68483e6d22cb8fc88385076bb485d2e23475ed015eb&scene=21#wechat_redirect)
- [奇妙的Window之旅](https://mp.weixin.qq.com/s/gl5qaHrbXKrvz257QJmQnw)
- [Android进阶必备，Window机制探索](https://mp.weixin.qq.com/s/IrwjQqlDoLp3xQZbthncVg)









