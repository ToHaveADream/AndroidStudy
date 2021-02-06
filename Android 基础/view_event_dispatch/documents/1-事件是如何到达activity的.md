# 事件是如何到达 activity 的

> 事件分发，真的一定从 Activity 开始吗？

事件分发的流程：Activity -> window -> view。这个我们大致都知道，那事件是怎么到达 Activity 的呢？如果了解过 Window 机制的话会知道，事件分发也是 Window 的一部分，而 Activity 不属于 Window 机制内，那么触摸事件应该是从 Window 开始才对，怎么是从 Activity 开始呢？抱着这些疑问，重新学习下事件分发。

接下来需要探究的是，一个触摸信息从系统底层产生之后，一步步到达 Activity 进行分发的整体流程。

## 管理单位：window

Android 的view 管理是以 window 为单位的，每一个 window 对应一个 view 树。这里管理涉及到 view 的绘制以及事件的分发等。Window 机制不仅管理着 view 的显示，也负责 view 的事件分发。

首先了解一下一个概念：view 树。

我们的应用布局，一般是有多层的 viewGroup 和 view 的嵌套，如下图：

![](../picture/window-08.image)

而他们对应的结构关系如下图所示：

![](../picture/window-09.image)

此时，我们就可以称该布局是以一个 Linearlayout 为根的一颗 view 树。LinearLayout 可以直接访问 FrameLayout 和 Relativelayout,因为他们都是 LinearLayout 的子 view，同样的 ReleativeLayout 可以直接访问 Button。

每一棵树都有一个根，叫做 viewRootImpl，它负责管理这整一颗 view 树的绘制、事件分发等。

我们的应用界面一般会有多个 view 树，我们的activity 布局就是一个 view 树，其他应用的悬浮框也是一个 view 树、dialog 界面也是一个 view 树、我们使用 windowManager 添加的 view 也是一个 view 树等等。最简单的 view 树只有一个 view。

Android 中 view 的绘制和事件分发，是以 view 树为单位。**每一个 view 树，则为一个 window**。系统服务 Window ManagerService，管理界面的显示就是以 window 为单位，也可以说是以 view 树为单位。而 view 树是由 viewRootImpl 来负责管理的，所以可以说，wms 管理的是 viewRootImpl。如下图：

![](../picture/window-10.image)

- WMS 是运行在系统服务进程的，负责管理所有的 window 。应用程序和 WMS 的通信必须通过 binder
- 每个 viewRootImpl 在 wms 中都有一个 windowState 对应，WMS 可以通过 WindowState 找到对应的 viewRootImpl 进行管理

了解 Window 机制的一个重要原因是：**事件分发并不由 Activity 驱动，而是由系统服务驱动 ViewRootImpl 来进行分发**，甚至可以说，在框架层角度，和 Activity 没有关系。这将有助于我们对事件分发的本质理解

那么触摸信息都是如何一步步到达 ViewRootImpl？为什么说 viewRootImpl 是事件分发的起点？viewRootImpl 如何对触摸信息进行分发处理？

## 触摸信息是如何到达 viewRootImpl 的？

我们都知道，在手指触摸屏幕的时候，即产生了触摸信息。这个触摸信息是由屏幕这个硬件产生，被系统底层驱动获取，交给 Android 的输入系统服务：InputManagerService，也就是 IMS。

IMS会对这个触摸信息进行处理，通过 WMS 找到要分发的 Window，随后发送给对应的 viewRootImpl。所以发送触摸信息的并不是 WMS，WMS 提供的是 window 的相关信息。大体流程如下：

![](../picture/window-11.image)

**当 viewRootImpl 接收到触摸信息时，也正是应用程序进程事件分发的开始**。

## ViewRootImpl 是如何分发事件的？

前面讲到，viewRootImpl 管理的是一颗 view 树，view 树的最外层是 viewGroup，而 viewGroup 继承于 view。因此一整颗 view 树，从外部可以看作一个 view。viewRootImpl 接受到触摸信息之后，经过处理之后，封装成 Motion Event 对象发送给它所管理的 view，由 view进行分发。

前面讲到，view 树的根节点可以是一个 viewGroup，也可以是一个单独的 view，因此，这里的派发就有两种不同的方式：直接给 view 进行处理 或者 viewGroup 进行事件分发。viewGroup 继承自 view，view 中有一个方法用于分发事件：`dispatchTouchEvent`。子类可重写该方法来实现自己的分发逻辑，viewGroup 重写了该方法。

我们的应用布局界面或者 dialog 的布局界面，顶层的 viewGroup 为 DecorView，因此会调用 DecorView 的 `dispatchTouchEvent` 方法进行分发。DecorView 重写了该方法。逻辑比较简单，仅仅做了一个判断：

```java
DecorView.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```

1. 如果 window callback 对象不为空，则调用 callBack 对象的分发方法进行分发
2. 如果 window callback 对象为空，则调用父类 ViewGroup 的事件分发方法进行分发

这里的 windowCallBack 是一个接口，它包含了一些 window 变化的回调方法，其中就有 diapatchTouchEvent，也就是 事件分发方法。

Activity 实现了 Window.CallBack 接口，并在创建布局的时候，把自己设置给了 DecorView，因此在 Activity 的布局界面中，DecorView 会把事件分发给 Activity 进行处理。同理，在 Dialog 的布局界面中，会分发给实现了 callback 接口的 dialog。

如果顶层的 viewGroup 不是 DecorView，那么调用对应 view 的 dispatchTouchEvent 方法进行分发。例如，顶层的 view 是一个 Button，那么会直接调用 Button 的 dispatchTouchEvent 方法；如果顶层 viewGroup 子类没有重写  dispatchTouchEvent 方法，那么会调用 viewGroup 默认的 dispatchTouchEvent 方法。

整体的流程如下：

![](../picture/window-12.image)

1、viewRootImpl 会直接调用管理的 view 的 dispatchEvent 方法，会根据具体的 view 的类型，调用具体的方法。

2、view 树 的根 view 可能是一个 view，也可能是一个 viewGroup，view 会直接处理事件，而 viewGroup 则会进行分发。

3、DecorView 重写了 dispatchTouchEvent 方法，会先判断是否存在 callback，优先调用 callback 的方法，也就是把事件传递给了 Activity。

4、其他的 view Group 子类会根据自身的逻辑进行分发

因此，触摸事件一定是从 Activity 开始的吗？不是，Activity 只是其中的一种情况，只有 Activity 自己负责的那一颗 view 树，才一定会到达 Activity，而其他的 window ，则不一定会经过 Activity。触摸事件是从 viewRootImpl，而不是 Activity。

## 控件对于事件的分发

到这里，我们知道触摸事件是先发送到 viewRootImpl,然后由 viewRootImpl 调用其所管理的 view 的方法进行事件分发。按照正常的流程，view 会按照控件树向下去分发，而事件却到了 activity、dialog，就是因为 DecorView 这个“叛徒”的存在。

前面讲到，DecorView 和其他的 view Group 很不一样，它有一个 windowCallBack，会优先把触摸事件发送给 callBack，从而导致触摸事件脱离了控件树。那么，这些 callback 是如何处理触摸事件的呢？触摸事件又是如何再一次回到控件树进行分发的呢？

了解具体的分发之前，需要先了解一个类：PhoneWindow。

PhoneWindow 继承自抽象类 Window，但是，它本身并不是 window。而是一个窗口功能辅助类。我们知道，一个 view 树，或者说控件树，就是一个 window。PhoneWindow 内部维护着一个控件树和一些 window 参数，这个控件树的根 view ，就是 DecorView。他们和 Activity 的关系如下图：

![](../picture/window-13.image)

我们的 Activity 通过直接持有 PhoneWindow 实例从而来管理这个控件树。DecorView 可以认为是一个界面模板，它的布局大概如下：

![](../picture/window-14.image)

我们的 Activity 布局，就添加到内容栏中，属于 DecorView 控件树的一部分，这样 Activity 可以通过 PhoneWindow 间接管理自身的界面，把 window 相关的操作都托管给 PhoneWindow，减轻自身负担。

PhoneWindow 并不是 Activity 专属的，其他的如 Dialog 也是自己创建了一个 PhoneWindow。PhoneWindow 仅仅只是作为一个窗口功能辅助类，帮助控件更好的创建和管理界面。

前面讲到，DecorView 接受到事件之后，会调用 windowCallBack 的方法进行事件分发，我们先来看看 Activity 是如何分发的：

**Activity**

我们先看 Activity 对于 callBack 接口方法的实现：

```java
Activity.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    // down事件，回调onUserInteraction方法
    // 这个方法是个空实现，给开发者去重写
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    // getWindow返回的就是PhoneWindow实例
    // 直接调用PhoneWindow的方法
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    // 如果前面分发过程中事件没有被处理，那么调用Activity自身的方法对事件进行处理
    return onTouchEvent(ev);
}
```

可以看到 Activity 对于事件的分发逻辑还是比较简单的，直接调用 PhoneWindow 的方法进行分发。如果事件没有被处理，那么自己处理这个事件。接下来看下 PhoneWindow 如何处理：

```java
PhoneWindow.java api29
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

这里的 mDecor 就是 PhoneWindow 内部维护的 DecorView 了，直接调用 DecorView 的方法进行分发。看到 DecorView 的方法：

```java
DecorView.java api29
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

:smile:DecorView 对于事件也是没有做任何处理，直接调用父类的方法进行分发。DecorView 继承于 FrameLayout，但是 FrameLayout 并没有重写 `dispatchTouchEvent` 方法，所以调用的就是 viewGroup 类的方法了。所以到这里，事件就交给 viewGroup 去分发控件树了。

我们来回顾一下：DecorView 交给 Activity 处理，Activity 直接交给 PhoneWindow 处理，PhoneWindow 直接交给内部的 DecorView 处理，而 DecorVIew 则直接调用父类 ViewGroup 的方法进行分发，ViewGroup 则会按照具体的逻辑分发到整个控件树中感兴趣的子控件。

从 DecorView 开始，绕了一圈，又回到控件树进行分发了。接下来看看 Dialog 是如何分发的：

**Dialog**

直接看到 Dialog 的 `dispatchTouchEvent` 代码：

```java
Dialog.java api29
public boolean dispatchTouchEvent(@NonNull MotionEvent ev) {
    if (mWindow.superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

这里的 mWindow，就是 Dialog 内部维护的 PhoneWindow 实例，接下去的逻辑就和 Activity 的流程一样了，这样不再赘述了。

而如果没有使用 DecorView 作为模板的窗口，流程就会和上述不一样了，例如 PopupWindow:

**PopupWindow**

PopupWinodw 它的根 View 是 `PopupDecorView` ,·而不是 `DecorView`。虽然它的名字带有 DecorView，但是却和 DecorView 一点关系都没有，它是直接继承于 FrameLayout。我们看到它的事件分发方法：

```java
PopupWindow.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (mTouchInterceptor != null && mTouchInterceptor.onTouch(this, ev)) {
        return true;
    }
    return super.dispatchTouchEvent(ev);
}
```

`mTouchInterceptor` 是一个拦截器，我们可以手动给 PopupWindow 设置拦截器。事件会优先交给拦截器处理，如果没有拦截器或者拦截器没有消费事件，那么才会交给 viewGroup 去进行分发。

## 总结

最后对整个流程进行一个回顾：

![](../picture/window-15.image)

1、IMS 从系统底层接受到事件之后，会从 WMS 中获取 window 信息，并将事件信息发送到对应的 viewRootImpl

2、viewRootImpl 接受到事件信息，封装成 motionEvent 对象后，发送给管理的 view。

3、view 会根据自身的类型，对事件进行分发还是自己处理

4、顶层 viewGroup 一般是 DecorView，DecorView 会根据自身 callBack 的情况，选择调用 callBack 或者调用父类 viewGroup 的方法

5、而不管顶层 viewGroup 的类型如何，最终都会到达 viewGroup 对事件进行分发。

到这里，虽然触摸事件的“去脉”我们还不清楚，但是它的“来龙”就已经非常清楚了。













