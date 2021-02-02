# MotionEvent详解

今天学习一下 `MotionEvent`的用途

### 事件坐标的含义

我们都知道，每个触摸事件都代表用户在屏幕上的一个动作，而每个动作必定有其发生的位置。在`MotionEvent`中就有一系列与标触摸事件发生位置相关的函数：

-  `getX()`和`getY()`：由这两个函数获得的x,y值是相对的坐标值，相对于消费这个事件的视图的左上点的坐标。（相对于当前的 View）
-  `getRawX()`和`getRawY()`:有这两个函数获得的x,y值是绝对坐标，是相对于屏幕的。
    在之前的文章中，我们曾经分析过事件如何通过层层分发，最终到达消费它的视图手中。（相对于屏幕左上角）
- 其中`ViewGroup`的`dispatchTransformedTouchEvent`函数有如下一段代码:

```java
    final float offsetX = mScrollX - child.mLeft;
    final float offsetY = mScrollY - child.mTop;
    event.offsetLocation(offsetX, offsetY);
    handled = child.dispatchTouchEvent(event);
    event.offsetLocation(-offsetX, -offsetY);
```

这段代码清晰展示了父视图把事件分发给子视图时，`getX()`和`getY`所获得的相关坐标是如何改变的。当父视图处理事件时，上述两个函数获得的相对坐标是相对于父视图的，然后通过上边这段代码，调整了相对坐标的值，让其变为相对于子视图啦。

![](/picture/motionevent-01.webp)

#### 事件类型

涉及 `MotionEvent`使用的代码一般如下：

涉及`MotionEvent`使用的代码一般如下:

```java
    int action = MotionEventCompat.getActionMasked(event);
    switch(action) {
        case MotionEvent.ACTION_DOWN:
            break;
        case MotionEvent.ACTION_MOVE:
            break;
        case MotionEvent.ACTION_UP:
            break;
    }
```

这里就引入了关于`MotionEvent`的一个重要概念，事件类型。事件类型就是指`MotionEvent`对象所代表的动作。比如说，当你的**一个**手指在屏幕上滑动一下时，系统会产生一系列的触摸事件对象,他们所代表的动作有所不同。有的事件代表你手指**按下**这个动作,有的事件代表你手指在屏幕上**滑动**,还有的事件代表你手指**离开**屏幕。这些事件的事件类型就分别为`ACTION_DOWN`,`ACTION_MOVE`,和`ACTION_UP`。上述这个动作所产生的一系列事件，被称为一个事件流，它包括一个`ACTION_DOWN`事件，很多个`ACTION_MOVE`事件，和一个`ACTION_UP`事件。

![](/picture/motionevent-02.webp)

当然，除了这三个类型外，还有很多不同的事件类型,比如`ACTION_CANCEL`。它代表当前的手势被取消。要理解这个类型，就必须要了解`ViewGroup`分发事件的机制。一般来说，如果一个子视图接收了父视图分发给它的`ACTION_DOWN`事件，那么与`ACTION_DOWN`事件相关的事件流就都要分发给这个子视图，但是如果父视图希望拦截其中的一些事件，不再继续转发事件给这个子视图的话，那么就需要给子视图一个`ACTION_CANCEL`事件

#### Pointer

当用户两个或者多个手指在屏幕上滑动时，系统又会产生怎样的事件流呢？为了可以表示多个触摸点的动作，MotionEvent 中引入了 Pointer 的概念，一个pointer就代表一个触摸点，每个 pointer 都有自己的事件类型，也有自己的横轴坐标值。**一个 MotionEvent 对象中可能会存储多个 pointer 的相关信息，每个 pointer 都会有一个自己的 id 和 index。pointer 的 id 在整个事件流中时不会发生变化的，但是index 会发生变化。**

MotionEvent 类中的很多方法都是可以传入一个 int 值作为参数的，其实传入的就是 pointer 的 index 值。比如 getX(pointerIndex) 和 getY(pointerIndex) ，此时，他们返回的就是 index 所代表的触摸点相关事件坐标值

由于 pointer 的 index 值在不同的 MotionEvent 对象中会发生变化，但是 id 值不会发生变化。所以，当我们要记录一个触摸点的事件流时，就只需要保存其 id，然后使用 findPointerIndex(int) 来获得其 index 值，然后再获得其他信息

```java
    private final static int INVALID_ID = -1;
    private int mActivePointerId = INVALID_ID;
    private int mSecondaryPointerId = INVALID_ID;
    private float mPrimaryLastX = -1;
    private float mPrimaryLastY = -1;
    private float mSecondaryLastX = -1;
    private float mSecondaryLastY = -1;
    public boolean onTouchEvent(MotionEvent event) {
        int action = MotionEventCompat.getActionMasked(event);

        switch (action) {
            case MotionEvent.ACTION_DOWN:
                int index = event.getActionIndex();
                mActivePointerId = event.getPointerId(index);
                mPrimaryLastX = MotionEventCompat.getX(event,index);
                mPrimaryLastY = MotionEventCompat.getY(event,index);
                break;
            case MotionEvent.ACTION_POINTER_DOWN:
                index = event.getActionIndex();
                mSecondaryPointerId = event.getPointerId(index);
                mSecondaryLastX = event.getX(index);
                mSecondaryLastY = event.getY(index);
                break;
            case MotionEvent.ACTION_MOVE:
                index = event.findPointerIndex(mActivePointerId);
                int secondaryIndex = MotionEventCompat.findPointerIndex(event,mSecondaryPointerId);
                final float x = MotionEventCompat.getX(event,index);
                final float y = MotionEventCompat.getY(event,index);
                final float secondX = MotionEventCompat.getX(event,secondaryIndex);
                final float secondY = MotionEventCompat.getY(event,secondaryIndex);
                break;
            case MotionEvent.ACTION_POINTER_UP:
                xxxxxx(涉及pointer id的转换，之后的文章会讲解)
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                mActivePointerId = INVALID_ID;
                mPrimaryLastX =-1;
                mPrimaryLastY = -1;
                break;
        }
        return true;
    }
```

除了 pointer 的概念，MotionEvent 还引入了两个事件类型：

- ACTION_POINTER_DOWN：代表用户又使用一个手指触摸到屏幕上，也就是说，在已经有一个触摸点的情况下，又新出现了一个触摸点
- ACTION_POINTER_UP：代表用户的一个手指离开了触摸屏，但是还有其他手指还在触摸屏上，也就是说，在多个触摸点存在的情况下，其中一个触摸点消失了。它与 ACTION_UP 的区别时：它是在多个触摸点中的一个触摸点消失时（此时，还有触摸点存在，也就是说用户还有手指触摸屏幕）产生，而ACTION_UP 可以说是最后一个触摸点消失时产生。
- 那么，用户先两个手指先后触摸屏幕，同时滑动，然后再先后离开这一套动作所产生的事件流是怎样的呢？

它所产生的事件流如下：

- 先产生一个`ACTION_DOWN`事件，代表用户的第一个手指接触到了屏幕。
- 再产生一个`ACTION_POINTER_DOWN`事件，代表用户的第二个手指接触到了屏幕。
- 很多的`ACTION_MOVE`事件，但是在这些`MotionEvent`对象中，都保存着两个触摸点滑动的信息，相关的代码我们会在文章的最后进行演示。（ACTION_MOVE 可能保存着多点的信息）
- 一个`ACTION_POINTER_UP`事件，代表用户的一个手指离开了屏幕。
- 如果用户剩下的手指还在滑动时，就会产生很多`ACTION_MOVE`事件。
- 一个`ACTION_UP`事件，代表用户的最后一个手指离开了屏幕

![两个手指的 gif 图](/picture/motionevent-03.webp)

#### getAction 和 getActionMasked

一个 MotionEvent 对象中可以包含多个触摸点的事件。当 MotionEvent 对象只包含一个触摸点的事件时，上边两个函数的结果是相同的，但是包含多个触摸点时，两者的结果就不同了。

getAction 获得的 int 值是由 pointer 值和事件类型值组合而成的，而 getActionMasked 则只返回事件的类型值，举个例子：

```kotlin
getAction() returns 0x0105.
getActionMasked() will return 0x0005
其中0x0100就是pointer的index值。
```

一般来说，getAction & ACTION_POINTER_INDEX_MASK 就获得了 pointer 的事件类型，等同于 getActionMasked

#### 批处理

为了效率，Android 系统在处理 ACTION_MOVE 事件时会将连续的几个多触点移动事件打包到一个 MotionEvent 对象中。我们可以通过 getX(int) 和 getY(int) 来获得最近发生的一个触摸点事件的坐标，然后使用 getHistorical(int,int) 和 getHistoricalXX(int, int) 来获得事件稍早的触点事件的坐标，二者是发生事件先后的关系。所以，我们应该先处理通过 getHistoricalXX 相关函数获得的事件信息，然后再处理当前的事件信息。

### 单点触控

单点触控非常简单

| 事件           | 简介                                       |
| -------------- | ------------------------------------------ |
| ACTION_DOWN    | 手指 **初次接触到屏幕** 时触发。           |
| ACTION_MOVE    | 手指 **在屏幕上滑动** 时触发，会多次触发。 |
| ACTION_UP      | 手指 **离开屏幕** 时触发。                 |
| ACTION_CANCEL  | 事件 **被上层拦截** 时触发。               |
| ACTION_OUTSIDE | 手指 **不在控件区域** 时触发。             |

和以下的几个方法:

| 方法        | 简介                                |
| ----------- | ----------------------------------- |
| getAction() | 获取事件类型。                      |
| getX()      | 获得触摸点在当前 View 的 X 轴坐标。 |
| getY()      | 获得触摸点在当前 View 的 Y 轴坐标。 |
| getRawX()   | 获得触摸点在整个屏幕的 X 轴坐标。   |
| getRawY()   | 获得触摸点在整个屏幕的 Y 轴坐标。   |

#### ACTION_CALCEL

`ACTION_CANCEL` 的触发条件是事件被上层拦截，**其实，只有上层 View 回收事件处理权的时候，ChildView 才会收到一个 ACTION_CANCEL 事件。**

> 例如：上层 View 是一个 RecyclerView，它收到了一个 `ACTION_DOWN` 事件，由于这个可能是点击事件，所以它先传递给对应 ItemView，询问 ItemView 是否需要这个事件，然而接下来又传递过来了一个 `ACTION_MOVE` 事件，且移动的方向和 RecyclerView 的可滑动方向一致，所以 RecyclerView 判断这个事件是滚动事件，于是要收回事件处理权，这时候对应的 ItemView 会收到一个 `ACTION_CANCEL` ，并且不会再收到后续事件。
>
> **通俗一点？**
>
> RecyclerView：儿砸，这里有一个 `ACTION_DOWN` 你看你要不要。
> ItemView ：好嘞，我看看。
> RecyclerView：噫？居然是移动事件`ACTION_MOVE`，我要滚起来了，儿砸，我可能要把你送去你姑父家(缓存区)了，在这之前给你一个 `ACTION_CANCEL`，你要收好啊。
> ItemView ：…...
>
> 这是实际开发中最有可能见到 `ACTION_CANCEL` 的场景了。

#### ACTION_DOWN

`ACTION_DOWN` 的触发条件更加奇葩。看下官方解释：

>  A movement has happened outside of the normal bounds of the UI element. This does not provide a full gesture, but only the initial location of the movement/touch.
>
> 一个触摸事件已经发生了UI元素的正常范围之外。因此不再提供完整的手势，只提供 运动/触摸 的初始位置。

我们知道，正常情况下，如果初始点击位置在该视图区域之外，该视图根本不可能会收到事件，然而，万事万物都不是绝对的，肯定还有一些特殊情况，你可曾还记得点击 Dialog 区域外关闭吗？Dialog 就是一个特殊的视图(没有占满屏幕大小的窗口)，能够接收到视图区域外的事件(虽然在通常情况下你根本用不到这个事件)，除了 Dialog 之外，你最可能看到这个事件的场景是悬浮窗，当然啦，想要接收到视图之外的事件需要一些特殊的设置。

> 设置视图的 WindowManager 布局参数的 flags为[`FLAG_WATCH_OUTSIDE_TOUCH`](http://developer.android.com/reference/android/view/WindowManager.LayoutParams.html#FLAG_WATCH_OUTSIDE_TOUCH)，这样点击事件发生在这个视图之外时，该视图就可以接收到一个 `ACTION_OUTSIDE` 事件。
>
> 参见StackOverflow：[How to dismiss the dialog with click on outside of the dialog?](http://stackoverflow.com/questions/8384067/how-to-dismiss-the-dialog-with-click-on-outside-of-the-dialog)

由于这个事件用到的几率比较小，此处就不展开叙述了，以后用到的时候再详细讲解。

### 多点触控

Android 在 2.2 版本的时候开始支持多点触控，一旦出现了多点触控，很多东西就突然之间变得麻烦起来了，首先要解决的问题就是 **多个手指同时按在屏幕上，会产生很多的事件，这些事件该如何区分呢？**

为了区分这些事件，工程师们用了一个很简单的办法－－**编号，当手指第一次按下时产生一个唯一的号码，手指抬起或者事件被拦截就回收编号，就这么简单。**

**第一次按下的手指特殊处理作为主指针，之后按下的手指作为辅助指针**，然后随之衍生出来了以下事件(注意增加的事件和事件简介的变化)：

| 事件                      | 简介                                                   |
| ------------------------- | ------------------------------------------------------ |
| ACTION_DOWN               | **第一个** 手指 **初次接触到屏幕** 时触发。            |
| ACTION_MOVE               | 手指 **在屏幕上滑动** 时触发，会多次触发。             |
| ACTION_UP                 | **最后一个** 手指 **离开屏幕** 时触发。                |
| **ACTION_POINTER_DOWN**   | 有非主要的手指按下(**即按下之前已经有手指在屏幕上**)。 |
| **ACTION_POINTER_UP**     | 有非主要的手指抬起(**即抬起之后仍然有手指在屏幕上**)。 |
| 以下事件类型不推荐使用    | －－－－－－－－－－－－－－－－－－                   |
| ~~ACTION_POINTER_1_DOWN~~ | 第 2 个手指按下，已废弃，不推荐使用。                  |
| ~~ACTION_POINTER_2_DOWN~~ | 第 3 个手指按下，已废弃，不推荐使用。                  |
| ~~ACTION_POINTER_3_DOWN~~ | 第 4 个手指按下，已废弃，不推荐使用。                  |
| ~~ACTION_POINTER_1_UP~~   | 第 2 个手指抬起，已废弃，不推荐使用。                  |
| ~~ACTION_POINTER_2_UP~~   | 第 3 个手指抬起，已废弃，不推荐使用。                  |
| ~~ACTION_POINTER_3_UP~~   | 第 4 个手指抬起，已废弃，不推荐使用。                  |

和以下方法：

| 方法                            | 简介                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| getActionMasked()               | 与 `getAction()` 类似，**多点触控必须使用这个方法获取事件类型**。 |
| getActionIndex()                | 获取该事件是哪个指针(手指)产生的。                           |
| getPointerCount()               | 获取在屏幕上手指的个数。                                     |
| getPointerId(int pointerIndex)  | 获取一个指针(手指)的唯一标识符ID，在手指按下和抬起之间ID始终不变。 |
| findPointerIndex(int pointerId) | 通过PointerId获取到当前状态下PointIndex，之后通过PointIndex获取其他内容。 |
| getX(int pointerIndex)          | 获取某一个指针(手指)的X坐标                                  |
| getY(int pointerIndex)          | 获取某一个指针(手指)的Y坐标                                  |

由于多点触控部分涉及内容比较多，也很复杂，我准备单独用一篇文章进行详细叙述，所以这里只叙述一些基础的内容作为铺垫：

### getAction() 与 getActionMasked()

当多个手指在屏幕上按下的时候，会产生大量的事件，如何在获取事件类型的同时区分这些事件就是一个大问题了。

一般来说我们可以通过为事件添加一个int类型的index属性来区分，为了添加一个通常数值不会超过10的index属性就浪费一个int大小的空间简直是不能忍受的，于是工程师们将这个index属性和事件类型直接合并了。

int类型共32位(0x00000000)，他们用最低8位(0x000000**ff**)表示事件类型，再往前的8位(0x0000**ff**00)表示事件编号，以手指按下为例讲解数值是如何合成的:

> ACTION_DOWN 的默认数值为 (0x00000000)
> ACTION_POINTER_DOWN 的默认数值为 (0x00000005)

| 手指按下      | 触发事件(数值)                       |
| ------------- | ------------------------------------ |
| 第1个手指按下 | ACTION_DOWN (0x0000**00**00)         |
| 第2个手指按下 | ACTION_POINTER_DOWN (0x0000**01**05) |
| 第3个手指按下 | ACTION_POINTER_DOWN (0x0000**02**05) |
| 第4个手指按下 | ACTION_POINTER_DOWN (0x0000**03**05) |

**注意：**
上面表格中用粗体标示出的数值，可以看到随着按下手指数量的增加，这个数值也是一直变化的，进而导致我们使用 `getAction()` 获取到的数值无法与标准的事件类型进行对比，为了解决这个问题，他们创建了一个 `getActionMasked()` 方法，这个方法可以清除index数值，让其变成一个标准的事件类型。
**1、多点触控时必须使用 getActionMasked() 来获取事件类型。**
**2、单点触控时由于事件数值不变，使用 getAction() 和 getActionMasked() 两个方法都可以。**
**3、使用 getActionIndex() 可以获取到这个index数值。不过请注意，getActionIndex() 只在 down 和 up 时有效，move 时是无效的。**

目前来说获取事件类型使用 `getActionMasked()` 就行了，但是如果一定要编译时兼容古董版本的话，可以考虑使用这样的写法:

```java
final int action = (Build.VERSION.SDK_INT >= Build.VERSION_CODES.FROYO)
                ? event.getActionMasked()
                : event.getAction();
switch (action){
    case MotionEvent.ACTION_DOWN:
        // TODO
        break;
}
```

### PointId

虽然前面刚刚说了一个 actionIndex，可以使用 getActionIndex() 获得，但通过 actionIndex 字面意思知道，这个只表示事件的序号，而且根据其说明文档解释，这个 ActionIndex 只有在手指按下(down)和抬起(up)时是有用的，在移动(move)时是没有用的，事件追踪非常重要的一环就是移动(move)，然而它却没卵用，这也太不实在了 (￣Д￣)ﾉ

**郑重声明：追踪事件流，请认准 PointId，这是唯一官方指定标准，不要相信 ActionIndex 那个小婊砸。**

PointId 在手指按下时产生，手指抬起或者事件被取消后消失，是一个事件流程中唯一不变的标识，可以在手指按下时 通过 `getPointerId(int pointerIndex)` 获得。 (参数中的 pointerIndex 就是 actionIndex)

关于事件流的追踪等问题在讲解多点触控时再详细讲解。

## 历史数据(批处理)

由于我们的设备非常灵敏，手指稍微移动一下就会产生一个移动事件，所以移动事件会产生的特别频繁，为了提高效率，系统会将近期的多个移动事件(move)按照事件发生的顺序进行排序打包放在同一个 MotionEvent 中，与之对应的产生了以下方法：

| 事件                              | 简介                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| getHistorySize()                  | 获取历史事件集合大小                                         |
| getHistoricalX(int pos)           | 获取第pos个历史事件x坐标 (pos < getHistorySize())            |
| getHistoricalY(int pos)           | 获取第pos个历史事件y坐标 (pos < getHistorySize())            |
| getHistoricalX (int pin, int pos) | 获取第pin个手指的第pos个历史事件x坐标 (pin < getPointerCount(), pos < getHistorySize() ) |
| getHistoricalY (int pin, int pos) | 获取第pin个手指的第pos个历史事件y坐标 (pin < getPointerCount(), pos < getHistorySize() ) |

**注意：**

1. pin 全称是 pointerIndex，表示第几个手指，此处为了节省空间使用了缩写。
2. 历史数据只有 ACTION_MOVE 事件。
3. 历史数据单点触控和多点触控均可以用。

## 获取事件发生的时间

获取事件发生的时间。

| 方法                            | 简介                     |
| ------------------------------- | ------------------------ |
| getDownTime()                   | 获取手指按下时的时间。   |
| getEventTime()                  | 获取当前事件发生的时间。 |
| getHistoricalEventTime(int pos) | 获取历史事件发生的时间。 |

> 1. pos 表示历史数据中的第几个数据。( pos < getHistorySize() )
> 2. 返回值类型为 long，单位是毫秒。

## 获取压力(接触面积大小)

MotionEvent支持获取某些输入设备(手指或触控笔)的与屏幕的接触面积和压力大小，主要有以下方法：

> 描述中使用了手指，触控笔也是一样的。

| 方法                                     | 简介                                               |
| ---------------------------------------- | -------------------------------------------------- |
| getSize ()                               | 获取第1个手指与屏幕接触面积的大小                  |
| getSize (int pin)                        | 获取第pin个手指与屏幕接触面积的大小                |
| getHistoricalSize (int pos)              | 获取历史数据中第1个手指在第pos次事件中的接触面积   |
| getHistoricalSize (int pin, int pos)     | 获取历史数据中第pin个手指在第pos次事件中的接触面积 |
| getPressure ()                           | 获取第一个手指的压力大小                           |
| getPressure (int pin)                    | 获取第pin个手指的压力大小                          |
| getHistoricalPressure (int pos)          | 获取历史数据中第1个手指在第pos次事件中的压力大小   |
| getHistoricalPressure (int pin, int pos) | 获取历史数据中第pin个手指在第pos次事件中的压力大小 |

> 1. pin 全称是 pointerIndex，表示第几个手指。(pin < getPointerCount() )
> 2. pos 表示历史数据中的第几个数据。( pos < getHistorySize() )

**注意：**

**1、获取接触面积大小和获取压力大小是需要硬件支持的。**
**2、非常不幸的是大部分设备所使用的电容屏不支持压力检测，但能够大致检测出接触面积。**
**3、大部分设备的 getPressure() 是使用接触面积来模拟的。**
**4、由于某些未知的原因(可能系统版本和硬件问题)，某些设备不支持该方法。**

我用不同的设备对这两个方法进行了测试，然而不同设备测试出来的结果不相同，之后经过我多方查证，发现是系统问题，有的设备上只有 `getSize()` 能用，有的设备上只有 `getPressure()` 能用，而有的则两个都不能用。

**由于获取接触面积和获取压力大小受系统和硬件影响，使用的时候一定要进行数据检测，以防因为设备问题而导致程序出错。**

## 鼠标事件

由于触控笔事件和手指事件处理流程大致相同，所以就不讲解了，这里讲解一下与鼠标相关的几个事件：

| 事件               | 简介                                                         |
| ------------------ | ------------------------------------------------------------ |
| ACTION_HOVER_ENTER | 指针移入到窗口或者View区域，但没有按下。                     |
| ACTION_HOVER_MOVE  | 指针在窗口或者View区域移动，但没有按下。                     |
| ACTION_HOVER_EXIT  | 指针移出到窗口或者View区域，但没有按下。                     |
| ACTION_SCROLL      | 滚轮滚动，可以触发水平滚动(AXIS_HSCROLL)或者垂直滚动(AXIS_VSCROLL) |

注意：

1、这些事件类型是 安卓4.0 (API 14) 才添加的。
2、使用 `getActionMasked()` 获得这些事件类型。
3、这些事件不会传递到 [onTouchEvent(MotionEvent)](https://developer.android.com/reference/android/view/View.html#onTouchEvent(android.view.MotionEvent)) 而是传递到 [onGenericMotionEvent(MotionEvent)](https://developer.android.com/reference/android/view/View.html#onGenericMotionEvent(android.view.MotionEvent)) 。

## 输入设备类型判断

输入设备类型判断也是安卓4.0 (API 14) 才添加的，主要包括以下几种设备：

| 设备类型          | 简介     |
| ----------------- | -------- |
| TOOL_TYPE_ERASER  | 橡皮擦   |
| TOOL_TYPE_FINGER  | 手指     |
| TOOL_TYPE_MOUSE   | 鼠标     |
| TOOL_TYPE_STYLUS  | 手写笔   |
| TOOL_TYPE_UNKNOWN | 未知类型 |

**使用 getToolType(int pointerIndex) 来获取对应的输入设备类型，pointIndex可以为0，但必须小于 getPointerCount()。**























































































































