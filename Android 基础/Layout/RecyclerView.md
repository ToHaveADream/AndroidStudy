# RecyclerView 的整体架构设计

![](/picture/01.awebp)

RecyclerView 类的注释中对自己的解释：它是一种可以在有限区域内展示大量数据的 View。不妨先根据这句简介盲猜一波 RecyclerView 可能拥有或需要拥有的一些能力。

- RecyclerView 的本质是 ViewGroup，因此它需要将大量数据对象映射成 View 对象集合，将 View对象集合中的 View 作为自己的子 View 在自身区域中布局展示
- 由于自己的数据集可能是无限的，所以可能存在一个无限大的 View 对象集合，但 RecyclerView 自身的区域有限，无法同时在这个区域展示所有子 View，这意味着它需要提供按需在自身区域内布局出被展示的子 View 的能力，并通过手势操作替换正在展示的子 View的能力，相对应的，在替换展示的 View 需要即时回收已经不被展示的 View 对象。
- 如果数据集合中的大多数据，都可以通过同一种 View 形态展示出来，则意味着对每一个数据对象都创建一个对象的 View 是浪费的，如果用 type 来区分数据对象期望的 View 展示形态，RecyclerView 需要拥有根据 type 安排数据对象复用 View 的能力，即同一个 type 的不同数据对象可以在不同的时机复用同一个 View 对象

对上述能力，RecyclerView 安排不同的类进行处理，依次分别对应：

- Adapter
- LayoutManager
- Recycler

# RecyclerView 的重要成员

## 3.1 ViewHolder

ViewHolder 是 RecyclerView 的基本单位。如果用 Item 表示数据集合中的一个对象，ItemView 表示用来展示该 Item 对应数据的 View对象。ViewHolder 负责在 RecyclerView 中承载一个 ItemView，此外，还维护着 Item 位置（position，此处的位置指的是 Item 在数据集合中的次序）、Item 类型、item Id 等。大部分的时候，RecyclerView 内部或其辅助类并不会直接操作 View，而是对 ViewHolder 进行操作。

在阅读 RecyclerView 源码时发现，一些操作需要用 View 做参数，也有一些操作需要用 ViewHolder 做参数，实际上在 RecyclerView 中，可以通过一样拿到另外一样。
![](/picture/02.awebp)

一般认为，Adapter 负责根据数据的 type 来创建对应 ViewHolder; Recycler 负责管理 ViewHolder，根据实际情况创建新的 ViewHoler 或者复用已有的；LayoutManager 可以通过 Recycler 直接获取到 View，负责将其添加到 RecyclerView 布局中，并通过 Recycler 回收不再展示的 View。

![](/picture/03.awebp)

## 3.2 Adapter

Adapter 负责将数据映射为 View 对象，待展示的数据集维护在 Adapter 中，Adapter 除了负责将数据映射为 View 之外，还负责向外通知数据集中数据的变化。

将数据对象映射为可用来展示的 View 对象，在 RecyclerView 体系中被拆分为两步：

- 根据itemType 类型创建 ViewHolder 对象，此处更关注于 View 的结构样式
- 根据 position 从 Adapter 中维护的数据集合中获取数据对象，将数据对象和 ViewHolder 中的 View 进行绑定，此处更关注于 View 展示出来的内容

在代码实现中，RecyclerView.Adapter 基类有两个待使用者实现的回调方法，这两个方法会分别被 Adapter.createViewHolder 和 Adapter.bindViewHolder 调用。

```
public abstract VH onCreateViewHolder(@NonNull ViewGroup parent, int viewType);

public abstract void onBindViewHolder(@NonNull VH holder, int position);
```

现在 Adapter 已经有了将数据集合映射为 View 的能力，那么它是在什么时候执行这个能力呢？看下一步，Recycler

## 3.3 Recycler

Recycler 的谷歌翻译是：回收器。正如字面意思，Recycler 负责管理 ViewHolder，它可以回收已经不被展示的 ViewHolder，并在恰当的时候复用这些 ViewHolder。它最重要的一个能力就是根据 position 提供一个 ViewHolder，使用者无需关心这个 ViewHolder 是新建的还是复用已有的，Recycler 帮助 RecyclerView 处理 ViewHolder 的缓存。

下面介绍下 View 被添加到 RecyclerView 后的几种状态变化，在介绍 Recycler 中对 View 的两种处理方法，最后介绍 RecyclerView 的缓存机制。

### 3.3.1 View 的 detach vs remove

先看下 View 被添加到 ViewGroup 后其状态的流转。View 被 add 到 Parent 后，除了可以被 Remove 之外，还有一个更加轻量级别的 detach 操作，detached 表示一种临时的状态，意味着这个 View 在之后会马上重新 attach 或 remove。如果一个 View 处于 detach 状态，像被 remove 一样它也无法通过其 Parent 的 getChildAt 方法获取。

![](/picture/04.awebp)

### 3.3.2 Recycle 的 scrap vs recycle

相应的，Recycler 对 ViewHolder 也有两种处理方式：scrap 和 recycle

scrap 经常和 detach 操作共同使用，如果使用 Recycler 对一个 View 进行 scrap 操作，表示期望该 View 已经处于 detach 状态，持有这个 View 的 ViewHolder 会被标记为 scrap 状态，然后临时存放到 Recycler.mAttachedScrap 列表，等待进一步处理（unscrap 或 Recycle）。scrap 是一种临时操作，通常表示该 View 之前在屏幕中展示，并且之后大概率也会被继续展示，不希望被 remove 回收掉。mAttachedScrap 是一个 ArrayList，存放着没有被 remove 的 子View 的 ViewHolder。

recycle 通常和 remove 操作共同使用，如果使用 Recycler 对一个 View 进行 Recycle 操作，表示期望该 View 已经从其 Parent 中 remove 掉，并且持有该 View 的 ViewHolder 是 unScrap 状态。当 ViewHolder 及其 View 状态已经满足的时候，RecyclerView 会将这个 ViewHolder 放入 Recycler 的缓存池中。recycle 操作只针对已经被 remove 的 View，它之前是被展示在屏幕中的，但是由于滑动或者数据集改变等因素，该 View 不再被展示，此时它可以被回收起来等待复用。

### 3.3.3 缓存与回收

了解 detach remove 和 scrap recycle 的区别后，RecyclerView 的缓存机制变得更易读一些，缓存实际上是 Recycler 中存放 ViewHolder 的 集合的变量，Recycler 中用来表示三级缓存的变量的优先级从高到低分别为：mCacheViews、mViewCacheExtension 和 mRecyclerPool。其中 mViewCacheExtension 是自定义缓存，本文不做展开，只看 mCacheView 和 mRecyclerPool ，首先需要明确，这两级缓存的内容都是已经不再屏幕内展示的 ViewHolder。

mCacheViews 是更高效的缓存，即不需要创建 ViewHolder，也不需要重新绑定 ViewHolder 步骤，这意味着只有在数据完全匹配的时候，即待展示的数据 item 和 与缓存的 ViewHolder 中的 item 完全匹配的时候，才会复用 mCacheViews 中的 ViewHolder。

mRecyclerPool 中缓存的 ViewHolder 对象的使用条件，相较于 mCacheViews 要求更低，只需 itemType 匹配，即可复用 ViewHolder，但是需要重新绑定 ViewHolder。

简单介绍下 mCacheViews 和 mRecyclerPool 数据结构上的区别。mCacheViews 是一个 ArrayList，可以存放 ViewHolder 类型的对象，mRecyclerPool 是 RecyclerViewPool 对象，可以简单理解为一种 Map 的数据结构

Recycler 回收 ViewHolder 的规则为：

- 如果 mCacheViews 未达到最大 的 size,则该 ViewHoler 被添加到 mCacheViews 中；如果已经达到最大值，则 移除 mCacheViews 中被先 add 的 ViewHolder，再将待回收的 ViewHolder 添加到 mCacheViews 中
- 如果 mCacheViews 达到最大值，则被其移除的 ViewHolder 会尝试 添加到 mRecyclerPool 中，如果可以存放，则直接 add，否则直接抛弃

![](/picture/05.awebp)

### 3.34 Recycler 获取 View

最后介绍 Recycler 获取 ViewHolder 的步骤，Recycler 可以根据一个给定的 position 获得一个可以直接用来展示的 ViewHolder。

![](/picture/06.awebp)

- Recycler先尝试从mAttachedScrap中获取可用的ViewHolder（可以认为该ViewHolder在复用前与复用后对应着同一个Item数据对象，且这个数据对象无变化），这里获取到的ViewHolder可以直接使用，既不需要执行Adapter.createViewHolder，也不需要执行Adapter.bindViewHolder。

- 如果未从mAttachedScrap中取到可用的ViewHolder，Recycler会尝试去缓存中获取，本文省略自定义缓存一层的介绍，Recycler会先从mCacheViews中尝试获取到符合要求的ViewHolder对象，与从mAttachedScrap中获取到的ViewHolder相似，该ViewHolder可以直接使用。

- 如果mCacheViews中依然没有满足条件的ViewHolder，则尝试从mRecyclerPool中获取到符合要求的ViewHolder，这里获得的ViewHolder itemType可以匹配，即View的结构样式满足需求，但需要重新进行数据绑定，即不需要执行Adapter.createViewHolder，但需要执行Adapter.bindViewHolder。

- 如果Recycler没有从缓存中得到符合要求的ViewHolder，会完整的执行Adapter的两个步骤

### 3.3.5 Recycler 小结

此时我们已经大致了解怎么通过 position 来获得一个可用的 ViewHolder，并且也清楚 Recycler 拥有两种操作 View/ViewHolder 的能力：scrap 和 recycle, 来临时缓存或保存一些 ViewHolder。那么是谁，在什么时候获取到可以用来展示的 ViewHolder?又是谁在什么时候调用 Recycler 临时保存或回收 ViewHolder？下面来看下 LayoutManager。

## 3.4 LayoutManager

LayoutManager 是 RecylerView 中决定 ItemView 摆放规则和滑动规则的执行者，甚至可以决定 ItemView 的一些布局参数。LayoutManager 中有几个待实现的抽象函数，给使用者充分的自由扩展 LayoutManager来实现自己想要的摆放和滑动效果。

```
// 创建ItemView默认的LayoutParams
public abstract LayoutParams generateDefaultLayoutParams();
// 布局RecyclerView的子View
public void onLayoutChildren(Recycler recycler, State state) {
    Log.e(TAG, "You must override onLayoutChildren(Recycler recycler, State state) ");
}
// RecyclerView是否支持水平滑动
public boolean canScrollHorizontally() {
    return false;
}
// RecyclerView是否支持垂直滑动
public boolean canScrollVertically() {
    return false;
}
// 处理RecyclerView的水平滑动
public int scrollHorizontallyBy(int dx, Recycler recycler, State state) {
    return 0;
}
// 处理RecyclerView的垂直滑动
public int scrollVerticallyBy(int dy, Recycler recycler, State state) {
    return 0;
}
```

### 3.5 ItemDecoration

ItemDecoration 可以让使用者向 ItemView 添加特殊的绘制 和布局偏移，此处先不对绘制进行展开，看下布局偏移。ItemDecoration 可以通过重写 getItemOffsets 方法自定义 ItemView 的间距，getItemOffsets 方法使用 Rect 记录四个值，这四个值类似于 ItemView 的 padding 或 margin 的概念，分别对应 left|right|top|bottom。该方法会在 LayoutManager 测量 ItemView 的时候调用，并将值对应的添加到 ItemView measure 后的结果中。

![](/picture/07.awebp)

需要注意，为 RecyclerView 设置 ItemDecoration 的方法是 add 而不是 set，因为其维护的是一个 Decoration 集合。这个要注意下。

## 3.6 ItemAnimator

最后再简单介绍 ItemAnimator，它用来定义 Adapter 中维护的数据集发生变化的时候 ItemView 需要执行的动画效果，例如删除某个正在展示中的 ItemView 对应的 Item 数据时，该 ItemView 需要执行消失动画，以及由于它的消失而引起的列表需要执行位移的动画。

基于后面几节的分析我们可以知道 RecyclerView 在正常的列表滑动下是不会触发 ItemAnimator 中定义的动画，只有 RecyclerView 布局的时候才会触发 ItemAnimator 的动画。

## 4.LayoutManager 的工作

LayoutManager 负责 RecyclerView 的布局，帮助 RecyclerView 决定 子 ItemView 的位置，并且这项工作并不一定只在 RecyclerView.onLayout 方法中完成。

## 4.1 RecyclerView 如何实现布局和绘制

为了了解 LayoutManager 是在什么时机开始布局 ItemView，可以先回到 RecyclerView 中；RecyclerView 作为一个 ViewGroup，肯定少不了 Measure、Layout、draw 三大流程。

### 4.1.1 measure

RecyclerView 为 LayoutManager 提供了自定义 onMesaure 方法的机会，如果 LayoutManager 期望 RecyclerView 使用自定义的 onMeasure 方法，可以通过重写 isAutoMesaureEnabled 方法返回 false 禁用 RecyclerView 的 AutoMeasure 策略，实际上，该方法默认返回 false，但是大多数情况下，常用的 LayoutManager 返回 TRUE。需要特别注意的是，当 isAutoMesaureEnabled 返回 true 时，不应该重写 onMesaure 方法。

![](/picture/08.awebp)

RecyclerView 的 onMeasure 的 autoMesaure 为 true 的时候，包含以下两个分支：

- 如果 RecyclerView 为固定宽高，则调用 RecyclerView 的 defaultOnMesaure 方法 结束 onMesaure。
- 如果 RecyclerView 是自适应宽高，则需要提前布局 ItemView，才可以确定 RecyclerView 的宽高，因此 RecyclerView 的 onMeasure 方法中会提前进行 layout 的部分过程

### 4.1.2 layout

RecyclerView 的 layout 步骤分为三部分，且三个步骤对应的方法命名也非常的简单：

- dispatchLayoutStep1
- dispatchLayoutStep2
- dispatchLayoutStep3

与之相对应的是 RecyclerView 中 state 类（State 类中记录各种可能会使用到的信息）中的 mLayoutStep 变量可能的三个取值：

- STEP_START
- STEP_LAYOUT
- STEP_ANIMATIONS

![](/picture/09.awebp)

RecyclerView 中一次完整的 layout 过程需要至少调用一次 dispatchLayoutStep1、dispatchLayoutStep2、dispatchLayoutStep3，其中dispatchLayoutStep2 可能被调用多次。在上面说，onMeasure 可能会提前执行 layout 的部分流程，是指 dispatchLayoutStep1 和 dispatchLayoutStep2, 如果 onMeasure 中已经完成 onLayout 的前两步工作，大多数情况 onLayout 中仅需执行 dispatchLayoutStep3 即可，如果 onMeasure 中未提前进行 layout 的前两步，则需要在 onLayout 中执行一次完整的 layout 过程。

![](/picture/10.awebp)

虽然经常说 RecyclerView 将 layout 的能力交给了 LayoutManager 处理，但实际上 RecyclerView 只是将布局子 view 的能力交给了 LayoutManager，RecyclerView 在 layout 过程中还会还会进行 pre-layout 预布局等操作。

在 layout 过程中，第二步即 dispatchLayoutStep2 中会调用 LayoutManager 中的 onLayoutChildren 方法，这一步通常也被认为是实际的布局过程 post-layout，在这一步将需要展示在屏幕上的 itemView 添加到 RecyclerView 中，并进行 ItemView 的 measure 和 layout ；layout 过程中的第一步和第三步主要是服务于 RecyclerView 的动画（ItemAnimator)，在第一步先进行 pre-layout，再在第三步比较 pre-layout 和 post-layout 的区别，进而触发 ItemAnimator 的动画执行。

### 4.1.3 draw

RecylerView 的绘制过程中特殊处理相对较少，本文只对 ItemDecoration 相关的流程进行介绍。在绘制 ItemView 之前，RecyclerView 会先遍历其维护的 ItemDecoration 列表，执行 ItemDecoration 的 onDraw 方法，绘制出来的内容在 ItemView 下层；ItemView 完成绘制后，执行 ItemDecoration 的 onDrawOver 方法，绘制出来的内容在 ItemView 上层。

![](/picture/11.awebp)

## 4.2 滚动处理

我们知道，用户的手指在屏幕上滑动的时候，会导致列表滑动，因此我们到 RecyclerView.onTouchEvent 的 ACTION_MOVE 分支中，观察它有没有与 scroll 相关的处理，发现 RecyclerView 在接收到 ACTION_MOVE 的消息之后，经过了一系列的计算和判断，可以得到手势滑动导致的列表水平方向和垂直方向的位移 dx dy，然后调用 RecyclerView 内部的 scrollInternal 方法处理滚动的位移值 dx 和 dy，最终进入 scrollStep 方法，根据 dx 和 dy 分别调用 LayoutManager 的 scrollHorizontallyBy 方法，把滚动导致的子 View 的移动和布局工作外包给了 LayoutManager，同时 LayoutManager 在处理滚动时也需要及时的使用 Recycler 处理不再屏幕中显示的子 View。

![](/picture/12.awebp)

关于 RecyclerView 滚动需要注意的是，以谷歌提供的 LinearLayoutManager 为例，它在处理滚动时，是调用 View 提供的 offsetTopAndBottom 方法平移已经展示在屏幕上的 ItemView，并使用 fill 方法向滚动产生的空白区域添加 view 和处理不再屏幕上展示的 view，在这个过程中，与 LayoutManager.onLayoutChildren 方法无关。一次正常的滚动并不会导致 RecyclerView 的重复布局。因此也不会触发 ItemAnimator 的任何动画。

## 4.3 数据更新处理

RecyclerView 在设置 Adapter 的时候，会创建 RecylerViewDataObserver 对象注册监听 Adapter 中的 Observable。RecylerViewDataObserver 做的事情其实就是在 Adapter 的数据集发送改变或其中某个数据发生改变时，在合适的情况下 requestLayout，重新完成一次 RecyclerView 的layout 过程，这个才是触发 ItemAnimator 动画的时机。

首先明确数据更新的几种类型：

- 数据集全量更新（DataSetChanged)

- 数据集局部更新

  - 局部 Item 改变（ItemChanged/ItemRangeChaned)
  - 新的 item 插入 (ItemInserted)
  - 已有 item 删除 (ItemRemoved)
  - 已有 item 移动 (ItemMoved)

  观察 RecyclerViewDataObserver 中用来处理数据更新的方法，发现这些方法中都使用了同一个帮助类：AdapterHelper。在 AdapterHelper 中，将数据更新行为抽象为 UpdateOp 类，每个 UpdateOp 类表示一次数据更新操作，AdapterHelper 中维护者一个待处理的数据更新操作列表 mPendingUpdates。

  如果 Adapter 触发了一次全量更新，那么 RecyclerViewDataObserver 中的处理方法会在 PendingList 为空的时候 requestLayout，进而触发 RV 重新布局；如果是局部更新，那么会在Pending 列表的 size 为 1 的时候 requestLayout，从而触发 RV 的重新布局。

  ![](/picture/13.awebp)

  

  # 4 小结

  1. Adapter根据数据对象的type提供View，并提供View和数据间的绑定关系，LayoutManager不需要与Adapter打交道。
  2. Recyler可以根据position提供一个可以直接用来展示的View，它还负责管理已经不被展示的View。LayoutManager需要直接与Recycler打交道，它在onLayoutChildren时向Recycler索要可以用来展示的View，并在处理滑动时将不再展示的View交由Recycler处理。
  3. ItemDecoration可以处理ItemView的布局偏移，LayoutManager在measure ItemView时会将其计算在内。
  4. ItemAnimator用来定义数据集发生改变时ItemView需要执行的动画，LayoutManager与其并无直接的联系。ItemAnimator定义的动画的执行时机是由RecyclerView的layout过程触发的，正常的列表滑动不会触发RecyclerView的重复布局，因此列表滑动时也不会触发ItemAnimator的执行。

  另外，从上述描述中可以知道LayoutManager需要完成的两个重要工作：

  1. 在onLayoutChildren方法中处理ItemView的布局。
  2. 在scrollHorizontallyBy和scrollVerticallyBy方法中处理列表滚动时ItemView的平移以及ItemView的补充和回收。

  此时我们了解到，LayoutManager可以处理子View的measure和layout过程，它可以按自己的需要measure child，并把子View放在它期望的位置上（甚至可以把所有子View都叠放在同一个位置）；LayoutManager还可以接管处理滚动的过程（如果愿意的话我们甚至可以在scroll方法中重新布局子View而不触发RecyclerView的layout过程）。

  