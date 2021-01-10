# View的体系

### 结构

------

目前，Android的View体系都是基于view和viewGroup两个大类，同时viewGroup又是View的子类。其结构设计基于“组合模式”，ViewGroup是容器，View是叶子节点，这就意味着View Group中既可以包含VIew，又可以包含ViewGroup，但是反之则不行，这就保证了依赖关系的安全

所以View的树状结构关系所呈现的如下

![](D:\ToBeAProfessionalAndroidDeveloper\AndroidStudy\Android 基础\view\view1.png)

做过Android的同学都知道，写布局文件的时候，ViewGroup和View都是层层嵌套的，他们是如何展示到屏幕上面来的呢？可以看到View是树状的，由于树的特性，View的展示过程是从上到下逐级别绘制，可以说每一个有视图Activity都包含一个View Tree来描述呈现在屏幕上的View关系，这让开发者可以更好的定位到每一个 View 的位置。同时，View 的绘制起点并不是我们所定义的那一个XML视图文件，在其外层还有系统所提供的一层东西所包裹，那这层是什么东西呢？

### 顶层布局

![](D:\ToBeAProfessionalAndroidDeveloper\AndroidStudy\Android 基础\view\view2.png)

我们都知道在Activity的OnCreate方法中需要实现setContentView方法来绑定我们的实现的布局，那么绑定的局部又是绑定在什么地方呢？直接绑定在Activity上吗？这就需要看源码来分析了

### 分析的入口

一般我们的coding都是从Activity开始。在Activity中都有OnCreate方法用于初始化色湖之，还记得setContentView(R.Layout.xxx)吗？这就是设置Xml布局的地方，也是分析的入口：

```
@Override
protected void onCreate(Bundle xxx) {
    super.OnCreate(savedInstance);
    //入口
    setContentView(R.Layout.xxx);
```

### 创建过程

###### 分析1.Activity中的setContentView方法

首先会进入到代理中：

```
@Override
public void setContentView(int layoutResId) {
	getDelegate().setContentView(layoutResId)；
}
```

在进入到getDelegate()下的setContentView，会发现是一个抽象方法，接下来去找具体实现这个方法的对象

其实在抽象方法的注释上面也表明了要去哪里寻找：

```
/**
  * Should be called instead of {@Link Activity#setContentView(int)}
  */
  public abstract void setContentView(int resId) {};
```

由于新建的Activity最终都是继承于Activity类的，所以我们在Activity类中找到setContentView方法实现：

```
//Activity的setContentView方法
public void setContentView(int layoutResId) {
    //获取windows，设置布局--->分析2.从哪里获取window，怎样设置布局
    getWindow().setContentView(layoutResId);
    //初始化Actionbar
    initWinodowDecorActionBar();
}
```

由上可以看到，Acctivity中的setContentView有两个步骤，从字面上来看：

1. 获取window，设置布局
2. 初始化ActionBar

QA:

1. 获取了什么Window？
2. 初始化的ActionBar从哪里来的，Decor是啥？

下面来解惑了：

#### 分析2：从哪里获取的Window，怎么样设置的布局

通过代码search发现getWindow返回了一个mWindow对象：

```
public Window getWindow() {
	return mWindow;
}
```

那么这个mWindow是从哪里创建的呢？通过search发现这是一个PhoneWindow

```
//mWindow是一个PhoneWindow的实例
mWindow = new PhoneWindow(this, window, activityConfigCallBack);
```

这样我们就知道是通过PhoneWindow来进行布局的设置的，那么进入到PhoneWindow中，搜索setContentView方法：

```java
// PhoneWindow中的setContentView方法，省略无关的代码
@Override
public void setContentView(int layoutResID) {
    // 判断内容父布局是否为空，如果为空则创建DecorView --> 分析3.创建DecorView的过程
    if (mContentParent == null) {
        installDecor();
    } 
    // 将布局添加到内容父布局当中去 --> 展示过程
    mLayoutInflater.inflate(layoutResID, mContentParent);
    // 此外通过其它重载方法进行布局的设置会调用
    // mContentParent.addView(view, params);方法
    // 原理是一样的，只不过所获的参数不同需要通过不同的方式进行展示
    // 因为最终都会通过addView(view, params)方法进行添加，展示过程将以这个方法作为入口
    
    ···
}
```

#### 分析3.创建DecorView的过程

进入到installDecor方法中，来看一下DecorView这个东西具体是怎么被创建出来的。

```java
// 省略无关代码，只关注DecorView的创建过程
private void installDecor() {
    // 判断DecorView是否为空，为空则创建DecorView
    if (mDecor == null) {
        // 创建DecorView
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } 
    // 不为空，则将DecorView依附到PhoneWindow上
    else {
        mDecor.setWindow(this);
    }
    
    // 如果内容父布局为空，则根据DecorView生成布局
    if (mContentParent == null) {
        // 获取内容布局
        mContentParent = generateLayout(mDecor);

        ···
    }
}
```

这里有两个方法 **generateDecor**和**generateLayout**分别用来创建和生成DecorView，在这其中具体做了什么呢？

generateDecor方法，主要获取了DecorView所需要的相关参数之后，进行了DecorView的创建，并返回了DecorView 的实例

```java

protected DecorView generateDecor(int featureId) {
    Context context;
    if (mUseDecorContext) {
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            context = new DecorContext(applicationContext, getContext());
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    return new DecorView(context, featureId, this, getAttributes());
}
```

generateLayout方法，主要实现了DecorView中布局的创建：

```
protected ViewGroup generateLayout(DecorView decor) {
    ···
    // 布局资源
    int layoutResource;
    // 一般情况下会获得screen_simple布局
    layoutResource = R.layout.screen_simple;
    // 并加载到DecorView中
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

    // 获取内容布局并返回
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    
    ···
    return contentParent;
}
```

至此，我们就得到了DecorView，那么问题又来了，DecorView中的布局是怎么样的呢？

这就需要找到上面生成DecorView布局的方法中使用的R.layout.screen_simple

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

可以看到DecorView中包含一个LinearLayout,其中包含两部分actionBar和FrameLayout（会被定义好的布局文件所代替）

那么整个嵌套关系就很清晰的展示出来了：Activity-->PhoneWindow-->DecorView-->LinearLayout-->包含ActionBar和定义的布局视图

创建了视图之后，自然就是需要展示出来了，下面来聊一聊展示的过程。

### 展示过程

回到PhoneWindow中的setContentView方法中，之前我们分析了DecorView的创建过程，接下来看一看DecorView创建之后，是怎样进行展示的？

```java
// PhoneWindow中的setContentView方法，省略无关的代码
@Override
public void setContentView(int layoutResID) {
    // 判断内容父布局是否为空，如果为空则创建DecorView --> 之前的分析.创建DecorView的过程
    if (mContentParent == null) {
        installDecor();
    } 
    // 将布局添加到内容父布局当中去 --> 展示过程
    mLayoutInflater.inflate(layoutResID, mContentParent);
    // 此外通过其它重载方法进行布局的设置会调用
    // mContentParent.addView(view, params);方法
    // 原理是一样的，只不过所获的参数不同需要通过不同的方式进行展示
    // 因为最终都会通过addView(view, params)方法进行添加，展示过程将以这个方法作为入口
    
    ···
}
```

展示过程最终都会通过ViewGroup的**addView**方法来进行绘制展示，在这部分内容中我们会进入到View的绘制流程中

```java
public void addView(View child, int index, LayoutParams params) {
    
    ···
    
    // 请求布局 --> 分析4.请求布局的过程
    requestLayout();
    // 无效化？？？ invalidate这个单词是无效化的意思 --> 分析5.真的是无效化吗？
    invalidate(true);
    addViewInner(child, index, params, false);
}
```

#### 分析4.请求布局的过程

首先对**requestLayout**方法进行分析，进入到方法中，核心方法就是mParent.requestLayout();的调用：

```java
// View的requestLayout方法
public void requestLayout() {
    
    if (mParent != null && !mParent.isLayoutRequested()) {
        // 调用ViewRootImpl的requestLayout方法
        mParent.requestLayout();
    }
    
    ···
}
```

这里出现了一个新的变量**mParent**，那么它是什么呢？通过代码追踪，我们发现这是一个接口。紧接着我们找到了接口的实现类**ViewRootImpl**,那我们进入到**ViewRootImpl**的**requestLayout**中来看看做了什么？

```java
// ViewRootImpl的requestLayout方法
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        // 检查线程，是否在UI线程
        checkThread();
        mLayoutRequested = true;
        // 遍历表 --> 分析6.最终的流程
        scheduleTraversals();
    }
}
```

关于**scheduleTraversals**分析6会覆盖到，不过在这之前，和**requestLayout**并列的**invalidate**方法需要进行分析：

#### 分析5.真的是无效化吗？

经过requestLayout();的过程后，代码执行到了invalidate(true)；真的是无效化吗？显然不是的。

```java
// 看到这个invalidateCache参数的名字有点明白这是什么意思了
public void invalidate(boolean invalidateCache) {
    // 具体的操作，这里将一些尺寸相关的参数传了进去
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}
```

看到形参的名字，猜测这是不是一个使得缓存的数据失效，并重新绘制的过程呢？

跟我走，进入到**invalidateInternal**方法中，核心代码其实只有一行。(基于Android10)

```java
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {

    ···

    // 无效化子控件，老版本的源码是这一行代码，标记为废弃了
    p.invalidateChild(this, damage);
    
    // 新的版本使用了其它方式，如下
    receiver.damageInParent();
}
```

那么进入到**damageInParent**方法中：(Descendant 后裔，子代，派生物:mask:)

```java
// View的方法
protected void damageInParent() {
    if (mParent != null && mAttachInfo != null) {
        mParent.onDescendantInvalidated(this, this);
    }
}
```

接着，进入到ViewRootImpl类中，看**onDescendantInvalidated**实现了什么：

```java
@Override
public void onDescendantInvalidated(@NonNull View child, @NonNull View descendant) {
    // TODO: Re-enable after camera is fixed or consider targetSdk checking this
    // checkThread();
    if ((descendant.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0) {
        mIsAnimating = true;
    }
    invalidate();
}
```

最后调用了**invalidate()**方法，进入到该方法中去：

```java
void invalidate() {
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        // 遍历表 --> 分析6.最终的流程
        scheduleTraversals();
    }
}
```

:smile:,饶了一大圈，又回到了scheduleTraversals这里，跟我去看看这里面发生了什么：

#### 分析6.最终的流程

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 注意这个mTraversalRunnable变量，是将某种操作进行了传递
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

注意其中**mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);**中的**mTraversalRunnable**变量，是将某种操作传递，那么去看看发生什么事情了：

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```

追踪到**doTraversal**里面：

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        // 真正进行遍历的地方，也就是之后的View绘制流程的入口
        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

代码就看到这里了，setContentView之后所做的事情我们大致就清楚了，之后就是具体的测量绘制流程了，这个接下来会进行总结

### 绘制功能

需要实现一个View，需要进行以下但不止以下事件的处理。

- 创建view
- 测量
- 布局
- 绘制
- 事件处理
- 焦点处理
- 显示处理
- 动画处理

:baby:看起来很多，但这才是要去做的意义呀

# Summary

总结了View的体系结构，以及view绘制的步骤，关于view的绘制详情，会接下来进行总结

