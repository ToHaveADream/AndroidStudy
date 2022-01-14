# 1、基本使用

```java
    @BindView(R.id.recyclerView)
    RecyclerView recyclerView;
    GridLayoutManager gridLayoutManager;

    gridLayoutManager = new GridLayoutManager(ActivityMain.this, 2);
    gridLayoutManager.setOrientation(LinearLayoutManager.VERTICAL);
    //设置LayoutManager
    recyclerView.setLayoutManager(gridLayoutManager);
    //设置左上边距
    recyclerView.addItemDecoration(new RecyclerView.ItemDecoration() {
        @Override
        public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
            super.getItemOffsets(outRect, view, parent, state);
            outRect.left = 4;
            outRect.top = 4;
         }
    });
    //设置item动画
    recyclerView.setItemAnimator(new DefaultItemAnimator());
    MAdapter mAdapter = new MAdapter();
    //设置Adapter
    recyclerView.setAdapter(mAdapter);
    mAdapter.notifyDataSetChanged();

```

上面是 Recycler View 的基本使用，就是设置 LayoutManager 装饰 动画 Adapter，如果设置完成之前 Adapter 没有填充数据，UI 界面是不会发生变化的。如果设置完成之后填充数据需要调用 notifyDataSetChanged 方法请求绘制刷新

# 2、初始化-绘制之前的准备

## 2.1 setAdapter(Adapter adapter)

这一步是必须的，不设置 Adapter ，那么数据就无法和 UI 关联起来，也就谈不上什么绘制了，绘制都是需要基于数据的。

`setAdapter` 方法源码：

```java
    //为RecyclerView设置Adapter，为RecyclerView提供子itemView
    public void setAdapter(Adapter adapter) {
        // 设置布局绘制不被冻结，以便刷新界面UI
        setLayoutFrozen(false);
        //设置Adapter内部实现
        setAdapterInternal(adapter, false, true);
        //请求重新布局
        requestLayout();
    }
```

一共调用了三个方法，重点说下第二个方法：我们设置的数据有变化时，UI 界面需要即时的刷新。要实现这样的逻辑，就必须存在数据变更观察者和被观察者，就是 RecyclerView 需要观察 Adapter 中的数据变化。

`Adapter` 中有一个被观察者：

```java
 private final AdapterDataObservable mObservable = new AdapterDataObservable();  //默认就创建好了
```

`RecyclerView` 中有一个观察者：

```java
 private final RecyclerViewDataObserver mObserver = new RecyclerViewDataObserver();  //默认就创建好了
```

在 `RecyclerView` 的 `setAdapterInternal` 方法中观察者和被观察者就发生了关系：

```java
    //设置Adapter内部实现
private void setAdapterInternal(Adapter adapter, boolean compatibleWithPrevious, boolean removeAndRecycleViews) {
    if (mAdapter != null) {
        //移除之前观察者和被观察者之间的关系
        mAdapter.unregisterAdapterDataObserver(mObserver);
        //移除之前RecyclerView和Adapter之间的关系
        mAdapter.onDetachedFromRecyclerView(this);
    }
    if (!compatibleWithPrevious || removeAndRecycleViews) {
        //不兼容之前的并且移除所以的itemView
        removeAndRecycleViews();
    }
    //重置删除、移动、增加辅助工具类AdapterHelper
    mAdapterHelper.reset();
    //记录之前的Adapter
    final Adapter oldAdapter = mAdapter;
    //赋值新的Adapter
    mAdapter = adapter;
    if (adapter != null) { //新的Adapter不为空
        //注册观察者跟被观察者之间的关系
        adapter.registerAdapterDataObserver(mObserver);
        //注册RecyclerView与Adapter之间的关系
        adapter.onAttachedToRecyclerView(this);
    }
    if (mLayout != null) {
        //通知LayoutManager Adapter发生了变化
        mLayout.onAdapterChanged(oldAdapter, mAdapter);
    }
    //通知回收复用管理者Recycler Adapter发生了变化
    mRecycler.onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious);
    mState.mStructureChanged = true;
    markKnownViewsInvalid();
}
```

经过一系列的重置之后调用 adapter.registerAdapterDataObserver(mObserver) 使二者发现关系。之后请求绘制。

## 2.2 setLayoutManager(LayoutManager layout)

这一步也是必须的，用什么类型的 `LayoutManager` 来绘制 `RecyclerView`，就算设置了 `Adapter`，没有设置 `LayoutManager`，`RecyclerView` 也不知道如何绘制。

`setLayoutManager` 方法源代码：

```
    //设置LayoutManager
   public void setLayoutManager(LayoutManager layout) {
        if (layout == mLayout) { //设置的LayoutManager跟之前的一样就返回
            return;
        }
        stopScroll();  //马上停止滚动
        // TODO We should do this switch a dispatchLayout pass and animate children. There is a good
        // chance that LayoutManagers will re-use views.
        if (mLayout != null) {
            // end all running animations
            if (mItemAnimator != null) {
                mItemAnimator.endAnimations();  //马上结束动画
            }
            //LayoutManager负责移除回收所有的itemView
            mLayout.removeAndRecycleAllViews(mRecycler);
            //LayoutManager负责移除回收所有的已经废弃itemView缓存
            mLayout.removeAndRecycleScrapInt(mRecycler);
            //清除所有缓存
            mRecycler.clear();

            if (mIsAttached) {
                mLayout.dispatchDetachedFromWindow(this, mRecycler);
            }
            //重置LayoutManager中RecyclerView的状态
            mLayout.setRecyclerView(null);
            mLayout = null;
        } else {
            mRecycler.clear();
        }
        // this is just a defensive measure for faulty item animators.
        mChildHelper.removeAllViewsUnfiltered();
        mLayout = layout;
        if (layout != null) {
            if (layout.mRecyclerView != null) {
                throw new IllegalArgumentException("LayoutManager " + layout
                        + " is already attached to a RecyclerView: " + layout.mRecyclerView);
            }

            //把当前的RecyclerView交给设置进来的LayoutManager
            mLayout.setRecyclerView(this);

            if (mIsAttached) {
                mLayout.dispatchAttachedToWindow(this);
            }
        }
        //更新Recycler中的缓存大小
        mRecycler.updateViewCacheSize();
        //请求重绘
        requestLayout();
    }
```

做了一些重置后调用 `LayoutManager` 的 setRecyclerView(this) 来让两者进行关联，最后请求重绘。

## 2.3 addItemDecoration(ItemDecoration decor)

这一步不是必须的，给 ItemView 的绘制增加一些装饰，比如分割线，悬浮的指示等，通过重写 ItemDecoration 的 onDraw 方法可以实现很炫酷的 RecyclerView 显示效果。

`addItemDecoration` 方法源代码跟踪：

```java

    public void addItemDecoration(ItemDecoration decor) {
        //调用addItemDecoration方法
        addItemDecoration(decor, -1);
    }
            
    public void addItemDecoration(ItemDecoration decor, int index) {
        if (mLayout != null) {
            mLayout.assertNotInLayoutOrScroll("Cannot add item decoration during a scroll  or"
                    + " layout");
        }
        if (mItemDecorations.isEmpty()) { //保存ItemDecoration的ArrayList为空说明是第一次初始化
            setWillNotDraw(false);  //为第一次初始化设置 etWillNotDraw(false)保证RecyclerView的onDraw(canvas)方法可以调用，
                                    //应为RecyclerView所有Item的装饰是在RecyclerView的onDraw(canvas)里面被调用绘制的
        }
        if (index < 0) {  //把ItemDecoration加入ArrayList中
            mItemDecorations.add(decor);  
        } else {
            mItemDecorations.add(index, decor);
        }
        //使每一个itemView的装饰都可以被绘制
        markItemDecorInsetsDirty();
        //请求重绘
        requestLayout();
    }
```

经过一系列判断后把需要的装饰 `ItemDecoration` 加入到 `ArrayList` 中保存并请求重绘。

## 2.4 setItemAnimator

这一步也不是必须的，`RecyclerView` 的 `ItemView` 的动画，一般都是 `ItemView` 第一次创建展示的时候会播放动画，或者移除 `ItemView` 的时候。如果没有设置，也会有一个默认的动画。

`setItemAnimator` 方法源代码：

```java
    public void setItemAnimator(ItemAnimator animator) {
        if (mItemAnimator != null) {
            //如果之前的动画不为空就结束动画移除监听
            mItemAnimator.endAnimations();
            mItemAnimator.setListener(null);
        }
        //赋值
        mItemAnimator = animator;
        if (mItemAnimator != null) {
            //设置监听
            mItemAnimator.setListener(mItemAnimatorListener);
        }
    }
```

前面三個方法其实在最后都会调用 RequestLayout 去请求重绘。如果在设置 Adapter 的时候已经绑定了数据，其实就不用调用 `notifyDataSetChanged` 方法来绘制 UI 界面了。如果是之后绑定数据的话，那么就需要手动调用该方法来请求绘制。

`Adapter.notifyDataSetChanged`方法阅读源码：

```java
public final void notifyDataSetChanged() {
   mObservable.notifyChanged();
}
```

继续看：

```java
//mObservers是保存观察者的一个ArrayList
public void notifyChanged() {
    for (int i = mObservers.size() - 1; i >= 0; i--) {
        //通知所有的观察者数据发送改变了
        mObservers.get(i).onChanged();
    }
}
```

继续看：

```java
@Override
public void onChanged() {
    ......
    //Adapter目前没有待更新或者正在更新的操作才可以重新绘制
    if (!mAdapterHelper.hasPendingUpdates()) {
        requestLayout();
    }
}
```

调用 `requestLayout` 方法后就会走到下面的View 三大步骤了。

# 3. OnMesaure

```java
    @Override
    protected void onMeasure(int widthSpec, int heightSpec) {
        if (mLayout == null) {  //LayoutM为空就走默认测量然后返回
            defaultOnMeasure(widthSpec, heightSpec);
            return;
        }
        if (mLayout.mAutoMeasure) {  //是不是自动测量，我们创建LayoutManager时默认为支持自动测量
            final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);
            //当前RecyclerView的宽高是否都为精确值
            final boolean skipMeasure = widthMode == MeasureSpec.EXACTLY
                    && heightMode == MeasureSpec.EXACTLY;
            //委托LayoutManager来测量RecyclerView的宽高(还是走defaultOnMeasure方法)
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

            //如果RecyclerView的宽高都是写死的精确值或者是match_parent并且Adapter还没有设置就结束测量
            if (skipMeasure || mAdapter == null) {
                return;
            }

            //当RecyclerView的宽高设置为wrap_content时，skipMeasure=false 就是不跳过测量
            //当宽高为wrap_content时，就先不能确定RecyclerView的宽高，因为需要先测量子itemView的宽高后才可以确定自己的宽高
            if (mState.mLayoutStep == State.STEP_START) { //还未测量过
                //绘制第一个，收集
                dispatchLayoutStep1();
            }
            // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
            // consistency
            mLayout.setMeasureSpecs(widthSpec, heightSpec);
            mState.mIsMeasuring = true;
            //
            dispatchLayoutStep2();

            // now we can get the width and height from the children.
            //通过对子ItemView的测量布局来确定宽高为WARP_CONTENT的RecyclerView的宽高
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

            // if RecyclerView has non-exact width and height and if there is at least one child
            // which also has non-exact width & height, we have to re-measure.
            if (mLayout.shouldMeasureTwice()) {
                mLayout.setMeasureSpecs(
                        MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                        MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
                mState.mIsMeasuring = true;
                dispatchLayoutStep2();
                // now we can get the width and height from the children.
                mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
            }
        } else {
                  //如果是自定义LayoutManager就要自己实现
          }
    }
```

`RecyclerView`的 onMeasure 方法測量分为两种：

- 当 `RecyclerView` 的宽高设置为 `match_parent` 或 具体值的时候，`skipMeasure=true`,此时会只需要测试自身的宽高既可以知道 `RecyclerView` 的大小，这时 `onMeasure` 方法测量结束
- 当 `RecyclerView` 的宽高为 `match_parent` 时，`skipMeasure=false`, `onMeasure` 会继续执行下面的 `dispatchLayoutStep2`，其实就是测量 `RecyclerView`的子视图的大小最终确定 RV 的实际大小，这种情况真正的测量操作都是在方法 `dispatchLayoutStep2`里执行的。

看下这两个方法：

`dispatchLayoutStep1`:

```java
private void dispatchLayoutStep1() {
    mState.assertLayoutStep(State.STEP_START);
    mState.mIsMeasuring = false;
    //禁止布局请求
    eatRequestLayout();
    //清空itemView信息保存类
    mViewInfoStore.clear();
    onEnterLayoutOrScroll();
    //1.重排序所有UpdateOp，比如move操作会排到末尾
    //2.依次执行所有UpdateOp事件，更新VH的position（这里是前移mPosition，mPreLayoutPosition不变），如果VH被remove了标记它。
    /*在2中，“会决定是否在prelayout之前把更新告诉LM”，
    这里把更新告诉LM指的是把更新反应在VH的mPreposition上（VH中有mPosition、mPreLayoutPosition等成员，注意，prelayout中是使用的mPreLayoutPosition）mPosition是一定会更新的，mPreLayoutPosition则不一定。
    如果RV决定不把更新再prelayout之前告诉LM，则会对VH更新时的参数applyToPreLayout传入false，mPosition更新了而mPreLayoutPosition则是旧值，反之mPreLayoutPosition则和mPosition同步。
    当然，如何“决定”我们就不说了，有兴趣可以看下原文和源码。*/
    processAdapterUpdatesAndSetAnimationFlags();
    processAdapterUpdatesAndSetAnimationFlags();
    //保存焦点信息
    saveFocusInfo();
    //对RecyclerView的情况存储类State赋值
    mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
    mItemsAddedOrRemoved = mItemsChanged = false;
    mState.mInPreLayout = mState.mRunPredictiveAnimations;
    //我们需要展示的所有ItemView个数 就是我们写Adapter给的getItemCount返回值
    mState.mItemCount = mAdapter.getItemCount();
    //找到屏幕上可以绘制的最小position和最大position
    findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);

    if (mState.mRunSimpleAnimations) {
        // Step 0: Find out where all non-removed items are, pre-layout

        //获得界面上所以显示的itemView的个数
        int count = mChildHelper.getChildCount();

        for (int i = 0; i < count; ++i) {

            ......略

            //保存所有ViewHolder的动画信息
            mViewInfoStore.addToPreLayout(holder, animationInfo);
            if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
                    && !holder.shouldIgnore() && !holder.isInvalid()) {
                long key = getChangedHolderKey(holder);
               
                //如果ViewHolder有变更就保存起来
                mViewInfoStore.addToOldChangeHolders(key, holder);
            }
        }
    }
    if (mState.mRunPredictiveAnimations) {

        ......略
        final boolean didStructureChange = mState.mStructureChanged;
        mState.mStructureChanged = false;  //设置RecyclerView的结构没有改变，因为这里是为了测量ReyclerView宽高而预布局子ItemV
        // 让LayoutManager布局所有ItemView的位置
        mLayout.onLayoutChildren(mRecycler, mState);
        //恢复本来的状态
        mState.mStructureChanged = didStructureChange;
        ......略
        // 其实走到这里把子View的位置和大小测量位置完了，但是没有去draw子itemView，界面应该还是不可见的
        for (int i = 0; i < mChildHelper.getChildCount(); ++i) {

                ......略

                //为每一个ViewHolder创建一个ItemHolderInfo，保存ViewHolder的位置信息动画信息等
                final ItemHolderInfo animationInfo = mItemAnimator.recordPreLayoutInformation(
                        mState, viewHolder, flags, viewHolder.getUnmodifiedPayloads());
                ......略

            }
        }
        // we don't process disappearing list because they may re-appear in post layout pass.
        clearOldPositions();
    } else {
        clearOldPositions();
    }
    //恢复绘制锁定
    resumeRequestLayout(false);
    mState.mLayoutStep = State.STEP_LAYOUT;
}
```

`dispatchLayoutStep2` 方法：

```java
private void dispatchLayoutStep2() {
    //锁定布局请求
    eatRequestLayout();
    onEnterLayoutOrScroll();
    //设置RecyclerView为布局和动画状态
    mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    mAdapterHelper.consumeUpdatesInOnePass();
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;

    // Step 2: Run layout
    mState.mInPreLayout = false;  //预布局已经完成，dispatchLayoutStep1方法中此值为true，为false的时候才会去真正的测量子View
    mLayout.onLayoutChildren(mRecycler, mState);  //LayoutManager去测量布局子ItemView

    mState.mStructureChanged = false;  //设置RecyclerView结构没变化
    mPendingSavedState = null;

    ......略
    //恢复布局锁定
    resumeRequestLayout(false);
}
```

三步测量解释：

-  `dispatchLayoutStep1`: `Adapter`的更新; 决定该启动哪种动画; 保存当前`View`的信息(getLeft(), getRight(), getTop(), getBottom()等); 如果有必要，先跑一次布局并将信息保存下来。
-  `dispatchLayoutStep2`: 真正对子`View`做布局的地方。
-  `dispatchLayoutStep3`: 为动画保存`View`的相关信息; 触发动画; 相应的清理工作。
   但是在onMeasure方法中没有调用dispatchLayoutStep3方法，下面的onLayout方法中会调用到这个方法。

# 3. Layout

onLayout 方法阅读源代码：

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    //直接调用dispatchLayout()方法布局
    dispatchLayout();
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}
```

`dispatchLayout`方法阅读代码：

```java
void dispatchLayout() {
    if (mAdapter == null) {
        Log.e(TAG, "No adapter attached; skipping layout");
        // leave the state in START
        return;
    }
    if (mLayout == null) {
        Log.e(TAG, "No layout manager attached; skipping layout");
        // leave the state in START
        return;
    }
    //当Adapter或者LayoutManager为空的时候就直接返回，还测量布局个毛线球
    mState.mIsMeasuring = false;  //设置RecyclerView布局完成状态，前面已经设置预布局完成了。
    if (mState.mLayoutStep == State.STEP_START) { //如果没在OnMeasure阶段提前测量子ItemView
        dispatchLayoutStep1();  // 收集ItemView的ViewHolder的信息并保存
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();  //正在测量布局子ItemView
    } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
            || mLayout.getHeight() != getHeight()) {  
        // First 2 steps are done in onMeasure but looks like we have to run again due to
        // changed size.
        mLayout.setExactMeasureSpecsFrom(this);
        //有更新操作或者宽期望的宽高跟目前的宽高不一致就重新测量所有的子ItemView
        dispatchLayoutStep2();
    } else {
        // always make sure we sync them (to ensure mode is exact)
        mLayout.setExactMeasureSpecsFrom(this);
    }
    //执行测量第三部
    dispatchLayoutStep3();
}
```

测量布局总结：

如果 RecyclerView 的宽高是写死或者是 `match_parent`，那么在 onMeasure 界面不会提前测量布局子 ItemView。如果宽或高是 `wrap_content`的，则需要提交测量布局子 ItemView。

`dispatchLayoutStep1` `dispatchLayoutStep2` `dispatchLayoutStep3`这三步都一定会执行，只是在`RecyclerView`的宽高是`写死`或者是`match_parent`的时候会提前执行`dispatchLayoutStep1` `dispatchLayoutStep2`者两个方法。会在`onLayout`阶段执行`dispatchLayoutStep3`第三步。

在`RecyclerView` `写死宽高`的时候onMeasure阶段很容易，直接设定宽高。但是在onLayout阶段会把`dispatchLayoutStep1` `dispatchLayoutStep2` `dispatchLayoutStep3`三步依次执行。

下面是 `dispatchLayoutStep3` 这个方法，其注释如下：

> The final step of the layout where we save the information about views for animations,trigger animations and do any necessary cleanup.
>  最后一步的布局,我们保存触发动画和做任何必要的清理。

# 3. onDraw

onDraw 方法阅读源代码：

```java
@Override
public void onDraw(Canvas c) {
    super.onDraw(c); //所有的ItemView先绘制

    //子ItemView绘制完了以后再绘制装饰
    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) { //遍历所有的装饰依次绘制
        mItemDecorations.get(i).onDraw(c, this, mState);
    }
}
```

# LinearLayout 填充、测量、布局过程

在上面我们得知 RecyclerView 绘制过程中 dispatchLayoutStep 三个方法的调用。看到代码可以知道是在 `dispatchLayoutStep2` 方法中调用的 `onLayoutChildren` 方法来布局 `ItemView` 的。

```java
    private void dispatchLayoutStep2() {
      
        ......略

        // Step 2: Run layout
        mState.mInPreLayout = false;
        // 调用`LayoutManager`的`onLayoutChildren`方法来布局`ItemView`
        mLayout.onLayoutChildren(mRecycler, mState);

        ......略      

    }
```

下面是 `LinearLayoutManager`对循环布局所有的 `ItemView` 的流程图。

![](picture/14.awebp)

虽然在 `RecyclerView` 的源码中有三部绘制处理，但是都不是真正做绘制布局的地方，真正的绘制布局测量都放在了不同的 LayoutManager 中，以 `LinearLayoutManager` 为例。

`LinearLayoutManager` 布局从 `onLayoutChildren` 开始：

```java
   
   //LinearLayoutManager布局从onLayoutChildren方法开始
    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {

        // layout algorithm:  布局算法
        // 1) by checking children and other variables, find an anchor coordinate and an anchor item position. 
        // 通过检查孩子和其他变量，找到锚坐标和锚点项目位置   mAnchor为布局锚点 理解为不具有的起点.
        // mAnchor包含了子控件在Y轴上起始绘制偏移量（coordinate）,ItemView在Adapter中的索引位置(position)和布局方向(mLayoutFromEnd)
        // 2) fill towards start, stacking from bottom 开始填充, 从底部堆叠
        // 3) fill towards end, stacking from top 结束填充,从顶部堆叠
        // 4) scroll to fulfill requirements like stack from bottom. 滚动以满足堆栈从底部的要求

        ......略

        ensureLayoutState();
        mLayoutState.mRecycle = false;
        // resolve layout direction 设置布局方向(VERTICAL/HORIZONTAL)
        resolveShouldLayoutReverse();

        //重置绘制锚点信息
        mAnchorInfo.reset();

        // mStackFromEnd需要我们开发者主动调用，不然一直未false
        // VERTICAL方向为mLayoutFromEnd为false HORIZONTAL方向是为true   
        mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;

        // calculate anchor position and coordinate
        // ====== 布局算法第 1 步 ======： 计算更新保存绘制锚点信息
        updateAnchorInfoForLayout(recycler, state, mAnchorInfo);

        ......略

        // HORIZONTAL方向时开始绘制

        if (mAnchorInfo.mLayoutFromEnd) {
            //  ====== 布局算法第 2 步 ======： fill towards start 锚点位置朝start方向填充ItemView
            updateLayoutStateToFillStart(mAnchorInfo);
            mLayoutState.mExtra = extraForStart;
            // 填充第一次
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
            final int firstElement = mLayoutState.mCurrentPosition;
            if (mLayoutState.mAvailable > 0) {
                extraForEnd += mLayoutState.mAvailable;
            }
            //  ====== 布局算法第 3 步 ======： fill towards end 锚点位置朝end方向填充ItemView
            updateLayoutStateToFillEnd(mAnchorInfo);
            mLayoutState.mExtra = extraForEnd;
            mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
            // 填充第二次
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;
            ......略
        } else {

            // VERTICAL方向开始绘制

            //  ====== 布局算法第 2 步 ======： fill towards end 锚点位置朝end方向填充ItemView
            updateLayoutStateToFillEnd(mAnchorInfo);
            mLayoutState.mExtra = extraForEnd;
            // 填充第一次
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;
            final int lastElement = mLayoutState.mCurrentPosition;
            if (mLayoutState.mAvailable > 0) {
                extraForStart += mLayoutState.mAvailable;
            }
            //  ====== 布局算法第 3 步 ======： fill towards start 锚点位置朝start方向填充ItemView
            updateLayoutStateToFillStart(mAnchorInfo);
            mLayoutState.mExtra = extraForStart;
            mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
            // 填充第二次
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;

            ......略
      }

        //  ===布局算法第 4 步===： 计算滚动偏移量，如果有必要会在调用fill方法去填充新的ItemView
        layoutForPredictiveAnimations(recycler, state, startOffset, endOffset);
    
    }
```

Layout algotithm:布局算法：

- 通过检查孩子和其他变量，找到锚坐标和锚点项目位置，mAnchor 为布局锚点，理解为不具有的起点，mAnchor 包含了子控件在 Y 轴上起始绘制偏移量（coordinate), ItemView 在 Adapter 中的索引位置（position）和 布局方向（mLayoutFromEnd)。
- 开始填充，从底部堆叠
- 结束填充，从顶部堆叠
- 滚动以满足堆栈冲突底部的要求

至于为什么会有好几次调用 fill 方法，什么 fromEnd, fromStart ，这个请看图：

![ItemView 的布局方向](picture/15.awebp)

图形红点就是布局算法在第一步 `updateAnchorInfoForLayout` 方法中计算出来的填充锚点的位置。

第一种情况是屏幕显示的位置在 `RecyclerView` 的最底部，那么也就只有一种填充方向为 `FromEnd`
第二种情况是屏幕显示的位置在 `RecyclerView` 的顶部，那么也只有一种填充方向为 `fromStart`

第三种情况是最常见的，屏幕显示的位置在 `RV` 的中间，那么填充方向就有两个 `End 和 start`

# 二、fill 开始布局 ItemView

`fill`核心就是一个 `while`循环，执行了一个很核心的方法就是：

`layoutChunk`,此方法执行一次就填充一个 ItemView 到屏幕

看一下 `fill` 的代码：其下一步的核心代码是 `layoutChunk`

```java
    // fill填充方法， 返回的是填充ItemView需要的像素，以便拿去做滚动
    int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
        // 填充起始位置
        final int start = layoutState.mAvailable;
        if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
            //如果有滚动就执行一次回收
            recycleByLayoutState(recycler, layoutState);
        }
        // 计算剩余可用的填充空间
        int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
        // 用于记录每一次while循环的填充结果
        LayoutChunkResult layoutChunkResult = mLayoutChunkResult;

        // ================== 核心while循环 ====================

        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            layoutChunkResult.resetInternal();
            
            // ====== 填充itemView核心填充方法 ====== 屏幕还有剩余可用空间并且还有数据就继续执行

            layoutChunk(recycler, state, layoutState, layoutChunkResult);

        }

        ......略

        // 填充完成后修改起始位置
        return start - layoutState.mAvailable;
    }
```

# 三、`layoutChunk`  创建 填充 测量 布局 `ItemView`

`layoutChunk`方法主要功能标题已经说了 创建、填充、测量、布局 ItemView，一共有四部

1. layoutState.next(recycler) 方法从一二级缓存中获取或者是创建一个 ItemView
2. `addView``方法加入一个ItemView`到 `ViewGroup`
3. `measureChildWithMargins`方法测量一个ItemView
4. `layoutDecorationWithMargins` 方法布局一个 `ItemView`。布局之前会计算好一个 `ItemView`的 left，top，right，bottom 位置。

其实这就是四大关键步骤：

```javascript
    void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
        LayoutState layoutState, LayoutChunkResult result) {

        // ====== 第 1 步 ====== 从一二级缓存中获取或者是创建一个ItemView
        View view = layoutState.next(recycler);
        if (view == null) {
            if (DEBUG && layoutState.mScrapList == null) {
                throw new RuntimeException("received null view when unexpected");
            }
            // if we are laying out views in scrap, this may return null which means there is
            // no more items to layout.
            result.mFinished = true;
            return;
        }

        // ====== 第 2 步 ====== 根据情况来添加ItemV，最终调用的还是ViewGroup的addView方法
        LayoutParams params = (LayoutParams) view.getLayoutParams();
        if (layoutState.mScrapList == null) {
            if (mShouldReverseLayout == (layoutState.mLayoutDirection
                    == LayoutState.LAYOUT_START)) {
                addView(view);
            } else {
                addView(view, 0);
            }
        } else {
            if (mShouldReverseLayout == (layoutState.mLayoutDirection
                    == LayoutState.LAYOUT_START)) {
                addDisappearingView(view);
            } else {
                addDisappearingView(view, 0);
            }
        }

        // ====== 第 3 步 ====== 测量一个ItemView的大小包含其margin值
        measureChildWithMargins(view, 0, 0);
        result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);

        // 计算一个ItemView的left, top, right, bottom坐标值
        int left, top, right, bottom;
        if (mOrientation == VERTICAL) {
            if (isLayoutRTL()) {
                right = getWidth() - getPaddingRight();
                left = right - mOrientationHelper.getDecoratedMeasurementInOther(view);
            } else {
                left = getPaddingLeft();
                right = left + mOrientationHelper.getDecoratedMeasurementInOther(view);
            }
            if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
                bottom = layoutState.mOffset;
                top = layoutState.mOffset - result.mConsumed;
            } else {
                top = layoutState.mOffset;
                bottom = layoutState.mOffset + result.mConsumed;
            }
        } else {
            top = getPaddingTop();
            bottom = top + mOrientationHelper.getDecoratedMeasurementInOther(view);

            if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
                right = layoutState.mOffset;
                left = layoutState.mOffset - result.mConsumed;
            } else {
                left = layoutState.mOffset;
                right = layoutState.mOffset + result.mConsumed;
            }
        }
        // We calculate everything with View's bounding box (which includes decor and margins)
        // To calculate correct layout position, we subtract margins.
        // 根据得到的一个ItemView的left, top, right, bottom坐标值来确定其位置


        // ====== 第 4 步 ====== 确定一个ItemView的位置
        layoutDecoratedWithMargins(view, left, top, right, bottom);
        if (DEBUG) {
            Log.d(TAG, "laid out child at position " + getPosition(view) + ", with l:"
                    + (left + params.leftMargin) + ", t:" + (top + params.topMargin) + ", r:"
                    + (right - params.rightMargin) + ", b:" + (bottom - params.bottomMargin));
        }
        // Consume the available space if the view is not removed OR changed
        if (params.isItemRemoved() || params.isItemChanged()) {
            result.mIgnoreConsumed = true;
        }
        result.mFocusable = view.hasFocusable();
    }
```

