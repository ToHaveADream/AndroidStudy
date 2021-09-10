> 1、Fragment 的过去、现在和未来

##### 1.1 Fragment 解决了什么问题？(过去)

Fragment 可以将 Activity 视图拆分为多个区块进行模块化管理，避免了 Activity 视图代码过度臃肿混乱。虽然自定义 View 或 Window 也可以在一定程度上拆分 Activity 界面，但事实上他们的职责不同：View / Window 的职责是封装某种视图功能，而 Fragment 是在更高的层次使用控制自定义 view。此外，Fragment 还可以更方便地管理声明周期和事务（虽然我们会通过 MVP 或 MVVM 模式分离业务逻辑，但是对于复杂页面，我们还是无法避免 Activity 视图代码演化地非常臃肿混乱）。	

![](/picture/fragment01.PNG)

需要主义的是，Fragment 不能脱离 Activity 独立存在，必须由 Activity 或另一个 Fragment 托管，Fragment#onCreateView 实例化的视图最终会被嵌入到宿主的视图树中。

| 类       | 角色       | MVVM 分层 | 生命周期感知 |
| -------- | ---------- | --------- | ------------ |
| Activity | 视图控制器 | View 层   | 感知         |
| Fragment | 视图控制器 | View 层   | 感知         |
| View     | 视图       | View 层   | 不感知       |
| Window   | 视图       | View 层   | 不感知       |

##### 1.2 Fragment 存在什么问题？（现在）

Fragment 的最初设计理念是“一个微型 Activity”的角色，很多专门为 Activity 设计的 API 也需要加入到 Fragment 中，比如运行时权限，多窗口模式切换等新 API。这无疑是在无限制地扩充 Fragment 的职责边界，也在扩大 Fragment 设计的复杂度，要知道 Fragment 的本质思想是界面模块化而已。

##### 1.3 Fragment 2.0（未来）

随着 AndroidX Fragment 版本更新，称为新版 2.0.已知的新特性包括：

- FragmentScenario:Fragment 的测试框架
- FragmentFactory: 统一的 Fragment 实例化组件
- FragmentContainerView: Fragment 专属视图容器
- OnBackPressedDispatcher: 在 Fragment 或其他组件中处理返回事件

> 说一下 Fragment 的整体结构

现在，我们正式开始讨论 Fragment 的核心工作原理，分析的过程中，将会结合源码来进行更加清晰的、全面理解 Fragment 的工作原理。在这之前，我们先梳理一下 Fragment 源码的整体结构设计，免得深入源码无法自拔。

> 2.1 代码框架

`FragmentActivity.java`

```java
final FragmentController mFragments = FragmentController.createController(new HostCallbacks());

protected void onCreate(@Nullable Bundle savedInstanceState) {
    mFragments.attachHost(null /*parent*/);
    super.onCreate(savedInstanceState);

    ...
    // FragmentController 中定义了很多 dispatchXX() 方法
    mFragments.dispatchCreate();
}
```

如下 UML 类图描述了 Fragment 整体的代码框架：

![](/picture/fragment02.PNG)

要点如下：

- FragmentActivity 是 Activity 支持 Fragment 的基础，其中持有一个 FragmentController 中间类，它是 FragmentActivity 和 FragmentManager 的中间桥接者，对 Fragment 的最终操作是分发到 FragmentManager 来处理：
- FragmentManager 承载了 Fragment 的核心逻辑，负责对 Fragment 执行添加、移除或者替换等操作，以及添加到返回堆栈。它的实现类 FragmentManagerImpl 是我们的主要分析对象：
- FragmentGHostCallback 是 FragmentManager 向 Fragment 宿主的回调接口，Activity 和 Fragment 中都有内部类实现该接口，所以 Activity 和 Fragment 都可以作为另一个 Fragment 的宿主。（Fragment 宿主 和 FragmentManager 是1 ：1 的关系）；
- FragmentTransaction 是 Fragment 事务抽象类，它的实现类 BackStackRecord 是事务管理的主要分析对象。

> 下面描述了每个宿主与关联的 FragmentManager 的关系：

![](/picture/fragment03.PNG)

> 阅读源码我们一定要拨开表面看本质，抓流程，不要拘泥于细枝末节

> 谈一下 Fragment 声明周期

​	生命周期感知是 Fragment 的最基础的功能，也是面试的重灾区，但还是需要很好的理解：

- Q1: 什么是生命周期，生命周期回调方法（比如 onCreateView()) 是生命周期的本质吗？

  An: 不然。状态转移才是生命周期的本质。生命周期方法的本质是 Fragment 状态的转移，当生命周期方法被调用，说明 Fragment 从一个状态转移到另外一个状态，而所谓的”生命周期回调“只是 Fragment 提供给开发者的 Hook 点，用于在状态转移的时候执行自定义逻辑。

- Q2：你提到状态转移，那你说下 Fragment 有哪几种状态？

  An：从源码来看，Android X Fragment 定义了物种状态，相对于早期的 Support Fragment 少了 STOPPED 状态，这是因为 Google 认为这些状态是可以对称使用的，例如 STOPPED 状态和 STARTED 状态其实没有本质区别

  `Fragment.java`

  ```
  static final int INITIALIZING = 0;     初始状态，Fragment 未创建
  static final int CREATED = 1;          已创建状态，Fragment 视图未创建
  static final int ACTIVITY_CREATED = 2; 已视图创建状态，Fragment 不可见
  static final int STARTED = 3;          可见状态，Fragment 不处于前台
  static final int RESUMED = 4;          前台状态，可接受用户交互
  ```

- Q3: Fragment 生命周期和宿主是同步的吗？如果不是，是独立的吗？

  An：不然。Fragment 的生命周期受【宿主】、【事务】、【setRetainInstance() API】三个因素影响：当宿主生命周期发生变化时，会触发 Fragment 状态转移到 宿主的最新状态。不过，使用事务和 setRetainInstance() API 也可以使 Fragment 在一定程度上与宿主状态不同步。（注意：宿主依然在一定程度上形成约束）。

  下面这张图完整描绘了 Fragment 生命周期：

  ![](/picture/fragment04.png)

  > 3、宿主如何改变 Fragment 状态

  > 3.1、Activity 与 Fragment 生命周期的同步关系

  ​	当宿主生命周期发生变化时，Fragment 的状态会同步到宿主的状态。从源码看，体现在宿主生命周期回调中会调用 FragmentManager 中一系列 dispatchXXX() 方法来触发 Fragment 状态转移。

  `FragmentActivity`

```
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    mFragments.attachHost(null /*parent*/);
    ...
    mFragments.dispatchCreate(); // 最终调用 FragmentManager#dispatchCreate()
}
```

下表总结了 Activity 生命周期与 Fragment 生命周期的关系：

| Activity 生命周期 | FragmentManager                       | Fragment 状态转移                           | Fragment 生命周期回调                                        |
| ----------------- | ------------------------------------- | ------------------------------------------- | ------------------------------------------------------------ |
| onCreate          | dispatchCreate                        | INITIALIZING -> CREATE                      | -> onAttach -> onCreate                                      |
| onStart（首次）   | dispatchActivityCreated dispatchStart | CREATE -> ACTIVITY_CREATED -> STARTED       | -> onCreateView -> onViewCreated -> onActivityCreated -> onStart |
| onStart（非首次） | dispatchStart                         | ACTIVITY_CREATED -> STARTED                 | -> onStart                                                   |
| onResume          | dispatchResume                        | STARTED -> RESUMED（Fragment 可交互）       | -> onResume                                                  |
| onPause           | dispatchPause                         | RESUMED -> STARTED                          | -> onPause                                                   |
| onStop            | dispatchStop                          | STARTED -> ACTIVITY_CREATED                 | -> onStop                                                    |
| onDestroy         | dispatchDestroy                       | ACTIVITY_CREATED -> CREATED -> INITIALIZING | -> onDestroyView -> onDestroy -> onDetach                    |

![](/picture/fragment05.png)

> 3.2 状态转移核心源码分析

FragmentManager 中一系列 dispatchXXX() 方法会触发 Fragment 状态转移，我们去看看：

`FragmentManager.java`

```java
（源码方法跳转太多，我直接帮你梳理出核心流程，跟你直接看源码会不同，但逻辑是相同的）
public void dispatchCreate() {
    mStateSaved = false;
    mStopped = false;
    moveToState(Fragment.CREATED, false);
    4、处理未执行的事务（见第 4 节）
    execPendingActions();
}

void moveToState(int newState, boolean always) {
    1、状态判断
    if (nextState == mCurState) {
        return;
    }
    mCurState = nextState;

    2、执行添加的 Fragment
    // Must add them in the proper order. mActive fragments may be out of order
    for (int i = 0; i < mAdded.size(); i++) {
        Fragment f = mAdded.get(i);
        // 更新 Fragment 到当前状态
        moveFragmentToExpectedState(f);
    }

    3、执行未添加，但是准备移除的 Fragment
    // Now iterate through all active fragments. These will include those that are removed and detached.
    for (int i = 0; i < mActive.size(); i++) {
        Fragment f = mActive.valueAt(i);
        if (f != null && (f.mRemoving || f.mDetached) && !f.mIsNewlyAdded) {
            // 更新 Fragment 到当前状态
            moveFragmentToExpectedState(f);
        }
    }
}
```

其中，moveFragmentToExceptedState() 最终调用到 moveToState(Fragment,int):

`FragmentManager.java`

```java
-> moveFragmentToExpectedState 最终调用到 
-> 更新 Fragment 到当前状态
void moveToState(Fragment f, int newState) {
    1、准备 Detatch Fragment 的情况，不再与宿主同步，进入 CREATED 状态
    if ((!f.mAdded || f.mDetached) && newState > Fragment.CREATED) {
        newState = Fragment.CREATED;
    }
    
    2、移除 Fragment 的情况，Fragment 不再与宿主同步
    if (f.mRemoving && newState > f.mState) {
        if (f.isInBackStack()) {
            2.1 移除动作添加到返回栈，则进入 CREATED 状态
            newState = Math.min(nextState, Fragment.CREATED);
        } else {
            2.1 移除动作添加到返回栈，则进入 DESTROY 状态
            newState = Math.min(nextState, Fragment.INITIALIZING);
        }
    }

    3、真正执行状态转移
    if (f.mState <= newState ) {
        switch (f.mState) {
            case Fragment.INITIALIZING:
                if (nextState> Fragment.INITIALIZING) {
                    ...
                }
            // fall through
            case Fragment.CREATED:
                ...
                // fall through
            case Fragment.ACTIVITY_CREATED:
                ...
                // fall through
            case Fragment.STARTED:
                ...
        }
    } else {
         switch (f.mState) {
            case Fragment.RESUMED:
                if (newState < Fragment.RESUMED) {
                    ...
                }
            // fall through
            case Fragment.STARTED:
            ...
            // fall through
            case Fragment.ACTIVITY_CREATED:
            ...
            // fall through
            case Fragment.CREATED:
            ...
        }
    }
    ...
}
```

**【总结一下】**

触发状态转移时，首先判断 Fragment，如果已经处于目标状态 newState，则会跳过状态转移。然而，并不是 FragmentManager 里所有的 Fragment 都会执行状态转移，只有**mAdded 为真 && mDetached 为假】**的 Fragment 才会更新到目标状态，其他 Fragment 会脱离宿主状态。最后，状态转移完成后会处理未执行的事务 `exePendingActions()`,可见每次 dispatchXXX() 都会执行一次事务执行的窗口。

不同 Fragment 标志位（Detach Remove 返回栈）与最终状态的关系总结如下表：

| 情况 | 判断                             | 描述                 | 最终状态                   |
| ---- | -------------------------------- | -------------------- | -------------------------- |
| 1    | f.mRemoving                      | 移除                 | Fragment.INITIALIZING      |
| 2    | f.mRemoving && f.isInBackStack() | 移除，但添加进返回栈 | Fragment.CREATED           |
| 3    | !f.mAdded \|\| f.mDetached       | 未添加               | Fragment.CREATED           |
| 4    | f.mAdded                         | 已添加               | newState（同步到宿主状态） |

> 这些标志位可以通过事务进行干涉

> 3.3 典型生命周期场景

基本规律：Activity 状态转移触发 Fragment状态转移

```java
首次启动：
Activity - onCreate
Fragment - onAttach
Fragment - onCreate
Fragment - onCreateView
Fragment - onViewCreated
Activity - onStart
Fragment - onActivityCreated
Fragment - onStart
Activity - onResume
Fragment - onResume
-------------------------------------------------
退出：
Activity - onPause
Fragment - onPause
Activity - onStop
Fragment - onStop
Activity - onDestroy
Fragment - onDestroyView
Fragment - onDestroy
Fragment - onDetach
-------------------------------------------------
回到桌面：
Activity - onPause
Fragment - onPause
Activity - onStop
Fragment - onStop
-------------------------------------------------
返回：
Activity - onStart
Fragment - onStart
Activity - onResume
Fragment - onResume
```

> 4、Fragment 事务管理

现在，我们来讨论影响 Fragment 状态转移的第二个因素：事务

> 4.1 事务概述

- 问题1：事务的特性是什么？事务是恢复和并发的基本单位，具备四个特性：
  - 原子性：事务不可分割，要么全部完成，要么全部失败回滚
  - 一致性：事务执行前后数据都具有一致性
  - 隔离性：事务执行过程中，不受其他事务干扰
  - 持久性：事务一旦完成，对数据的改变就是永久的。在 Android 中体现为 Fragment 状态保存之后，commit() 提交事务就会抛异常，因为这部分新提交的事务影响的状态无法保存
- Fragment 事务的作用：使用事务 FragmentTransaction 可以动态改变 Fragment 状态，使得 Fragment 在一定程度脱离宿主的状态。不过，事务依然收到宿主状态约束，例如：当前 Activity 处于 STARTED 状态，那么 addFragment 不会使得 Fragment 进入 RESUME 状态。只有将来 Activity 进入 RESUME 状态时，才会同步 Fragment 到最新状态。

> 4.2 知道不同事务的区别吗？

- add & remove：Fragment 状态在 INITIALIZING 与 RESUMED 之间转移
- detach & attach: Fragment 状态在 CREATE 与 RESUMED 之间转移
- replace: 先移除所有 containerId 中的实例，再 add 一个 Fragment；
- show & hide: 只控制 Fragment 隐藏或显示，不会触发状态转移，也不会销毁 Fragment 视图或实例；
- hide & detach & remove 的区别：hide 不会销毁实例和视图、detach 只销毁视图不销毁实例、remove 会销毁实例（自然也销毁视图）。不过，如果 remove 的时候将事务添加到回退栈，那么 Fragment 实例就不会被销毁，只会销毁视图

下图描述了 Fragment 状态转移与宿主和事务的简单关系：

![](/picture/fragment06.png)

这里有一个让人摸不着头脑的问题，detach Fragment 并不会回调 onDetach(), 因为 detach 只会转移到 CREATE  状态，而回调 onDetach 需要转移到  INITIALIZING。

```
detach Fragment：
Fragment - onPause
Fragment - onStop
Fragment - onDestroyView
```

> 4.3 说说看不同事务提交方式的区别？

FragmentTransaction 定义了 5 种提交方式：

| API                              | 描述                         | 是否同步 |
| -------------------------------- | ---------------------------- | -------- |
| **commit()**                     | 异步提交事务，不允许状态丢失 | 异步     |
| **commitAllowingStateLoss()**    | 异步提交事务，允许状态丢失   | 异步     |
| **commitNow()**                  | 同步提交事务，不允许状态丢失 | 同步     |
| **commitNowAllowingStateLoss()** | 同步提交事务，允许状态丢失   | 同步     |
| **executePendingTransactions()** | 同步执行事务队列中的全部事务 | 同步     |

需要注意的地方：

- **onSaveInstanceState() 保存状态后，事务形成的新状态是不会被保存的。在状态保存之后调用 commit 或 commitNow 会抛异常**

`FragmentManageImpl.java`

```
private void checkStateLoss() {
    if (mStateSaved || mStopped) {
        throw new IllegalStateException("Can not perform this action after onSaveInstanceState");
    }
}
```

- **使用 commitNow 或 CommitNowAllowingStateLoss 提交的事务不允许加入回退栈**

为什么有这个设计呢？可能是 Google 考虑到同步提交和异步提交的事务，并且两个事务都要加入回退栈时，无法确定哪个在上哪个在下是符合预期的，所以干脆禁止 commitNow() 加入回退栈。如果确实有需要同步执行 + 回退栈的应用场景，可以采用 `commit() + executePendingTransaction` 的取巧方法。相关源码体现如下：

`BackStackRecord.java`

```java
@Override
public void commitNow() {
    disallowAddToBackStack();
    mManager.execSingleAction(this, false);
}

@Override
public void commitNowAllowingStateLoss() {
    disallowAddToBackStack();
    mManager.execSingleAction(this, true);
}
@NonNull
public FragmentTransaction disallowAddToBackStack() {
    if (mAddToBackStack) {
        throw new IllegalStateException("This transaction is already being added to the back stack");
    }
    mAllowAddToBackStack = false;
    return this;
}    
```

- **commitNow() 和 executePendingTransactions() 都是同步执行，有区别吗？**

commitNow() 是同步执行当前事务，而 executePendingTransactions() 是同步执行事务队列中的全部事务。

> 5、如何把 Fragment 加载到界面上？

> 5.1 添加方法

有两种方式可以将 Fragment 添加到 Activity 视图上：静态加载 + 动态加载

1. 静态加载：静态加载是指在布局文件中使用 标签添加 Fragment 的方式，要点总结如下：

| 属性         | 描述                                                      |
| ------------ | --------------------------------------------------------- |
| class        | Fragment 的全限定类名                                     |
| android:name | Fragment 的全限定类名（与 class 没有差别，但 class 优先） |
| android:id   | Fragment 唯一标识                                         |
| android:tag  | Fragment 唯一标识（id 和 tag 至少设置一个）               |

举例：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <fragment android:name="com.example.TestFragmentFragment"
            android:id="@+id/list"
            android:layout_weight="1"
            android:layout_width="0dp"
            android:layout_height="match_parent" />
</LinearLayout>
```

- **动态加载**：动态加载是指在代码中使用事务 FragmentTransaction 添加 Fragment 的方式。例如：

  ```java
  TextFragment fragment = new TextFragment();
  fragmentTransaction.add(R.id.containerId, fragment);
  fragmentTransaction.commit();
  ```

> 5.2 Fragment 静态加载源码分析

从布局文件添加 Fragment 本质上是 xml 解析为视图树的过程，这个过程由 LayoutInflater 完成。最终，标签的解析工作最终是交给 FragmentManager# onCreateView() 处理的，让我们来看看具体是如何处理的，源码如下：

`FragmentManagerImpl.java`

```java
（已简化）
public View onCreateView(@Nullable View parent, @NonNull String name, @NonNull Context context, @NonNull AttributeSet attrs) {
    1、解析属性
    String fname = 解析 class 属性
    if (fname == null) {
        fname    = 解析 android:name 属性
    }
    int id       = 解析 android:id 属性
    String tag   = 解析 android:tag 属性
    
    2、根据 id 或 tag 重用已经创建的 Fragment
    Fragment fragment = id != View.NO_ID ? findFragmentById(id) : null;
    if (fragment == null && tag != null) {
        fragment = findFragmentByTag(tag);
    }
    if (fragment == null && containerId != View.NO_ID) {
        fragment = findFragmentById(containerId);
    }

    3、新建 Fragment
    if (fragment == null) {
        3.1 反射创建 Fragment 实例
        fragment = getFragmentFactory().instantiate(context.getClassLoader(), fname);
        3.2 mFromLayout 设置为 true
        fragment.mFromLayout = true;
        fragment.mFragmentId = id != 0 ? id : containerId;
        fragment.mContainerId = containerId;
        fragment.mTag = tag;
        fragment.mInLayout = true;
        fragment.mFragmentManager = this;
        fragment.mHost = mHost;
        fragment.onInflate(mHost.getContext(), attrs, fragment.mSavedFragmentState);
        3.3 添加 Fragment，立即状态转移
        addFragment(fragment, true);
    } else if (fragment.mInLayout) {
        ...
    } else {
        ...
    }

    4.1 将 id 设置给 Fragment 根布局
    if (id != 0) {
        fragment.mView.setId(id);
    }
    4.2 将 tag 设置给 Fragment 根布局
    if (fragment.mView.getTag() == null) {
        fragment.mView.setTag(tag);
    }
    
    5、返回 Fragment 根布局
    return fragment.mView;
}

-> 3.3 添加 Fragment，立即状态转移
public void addFragment(Fragment fragment, boolean moveToStateNow) {
    ...
    if (moveToStateNow) {
        moveToState(fragment); // 状态转移
    }
}
-> 状态转移
void moveToState(Fragment f, int newState, ...) {
    ...
    if (f.mState <= newState ) {
        switch (f.mState) {
            case Fragment.INITIALIZING:
                if (nextState> Fragment.INITIALIZING) {
                    ...
                }
            // fall through
            case Fragment.CREATED:
                if (f.mFromLayout && !f.mPerformedCreateView) { // 如果来自布局，并且未执行过 onCreateView
                    最终调用：
                    mView = onCreateView(inflater, container, savedInstanceState);
                    f.onViewCreated(f.mView, f.mSavedFragmentState);
                    提示：最终在 LayoutInflater 中执行 viewGroup.addView(view, params);
                }
                if (!f.mFromLayout) { // 不是来自布局
                    最终调用：
                    mView = onCreateView(inflater, container, savedInstanceState);
                    f.onViewCreated(f.mView, f.mSavedFragmentState);
                    if (container != null) {
                        container.addView(f.mView);
                    }
                }
                ...
                // fall through
            case Fragment.ACTIVITY_CREATED:
                ...
                // fall through
            case Fragment.STARTED:
                ...
        }
    } else {
         ...
    }
}
```

以上代码已经非常简化了，代码虽然很长但是流程很清楚：

1. FragmentManager 根据布局中的 id 属性 或 tag 属性来重用 Fragment, 如果不存在则通过反射来创建 Fragment 实例。
2. 设置 mFromLayout 为 true，并立即执行状态转移。在 moveToState 的 CREATE 分支会根据 mFromLayout 判断：如果来自布局，并且未执行过 onCreateView，才会回调 Fragment#onCreateView 创建 view 实例。
3. 最终回溯到 LayoutInflater 中，执行 ViewGroup#addView(mView), 将 Fragment 根布局添加到父布局中（所以，不用在 Fragment 里创建的视图时调用 addView()).

> 5.3 Fragment 动态加载源码分析

Fragment 事务的源码在 4.4 讨论过了，我们知道通过事务添加 移除 的 Fragment 最终还是会走到 moveToState 来执行状态转移，在创建 view 实例后，mView 也会直接添加到 containerId 容器上。

> 5.4 动态加载和静态加载的区别体现在哪里？

静态加载和动态加载的主要区别：**执行加载操作的消息周期不同**：

- 静态加载和布局解析是在同一个 Handler  消息周期中的
- 动态加载和事务提交不一定在一个 Handler 消息中（取决于 调用 commit 还是 commitNow)

> 5.5 静态加载和动态加载的优缺点？

- 静态加载适合于界面初始化时就确定显示位置和时机的 Fragment，从布局文件中加载可以方便预览。相反的，动态加载适用于初始化时无法确定显示位置和时机的 Fragment，需要依赖代码中的判断条件动态判断。

```java
演示：使用事务操作从布局文件中静态加载的 Fragment
with(supportFragmentManager.beginTransaction()) {
    val fragmentA = supportFragmentManager.findFragmentById(R.id.FragmentA)
    if (null != fragmentA) {
        hide(fragmentA)
        commit()
    }
}
```

#### setRetainInstance() 到底做了什么？

> 6.1 概述

- 问题1：什么时候应该使用 setRetainInstance(true)?

  答：在配置变更时（例如屏幕旋转），整个 activity 需要销毁重建，顺带着 Activity 中的 Fragment 也需要销毁重建。而设置 setRetainInstance 的 Fragment 对象在 Activity 销毁重建的过程中不会被销毁。

- 问题2：setRetainInstance(true) 对 Fragment 生命周期的影响？

  答：在 activity 销毁时，Fragment 不会回调 onDestroy, 而是回调 onDestroyView + onDetach; 在Activity 重建的时候，Fragment 不会回调 onCreate，而是回调 onCreateView()

- 问题3：为什么废弃 setRetainInstance()?

  答：引入 viewModel 后，setRetainInstance API 开始变得鸡肋。ViewModel 已经提供了在 Activity 重建等场景下保持数据的能力，虽然 setRetainInstance 也具备相同功能，但需要利用 Fragment 来间接存储数据，使用起来不方便，存储粒度也过大

> 6.2 setRetainInstance 核心源码分析

`Fragment.java`

```java
@Deprecated
public void setRetainInstance(boolean retain) {
    mRetainInstance = retain;
    if (mFragmentManager != null) {
        if (retain) {
            mFragmentManager.addRetainedFragment(this);
        } else {
            mFragmentManager.removeRetainedFragment(this);
        }
    } else {
        mRetainInstanceChangedWhileDetached = true;
    }
}
```

`FragmentManager.java`

```java
void addRetainedFragment(@NonNull Fragment f) {
    mNonConfig.addRetainedFragment(f);
}
```

`FragmentManagerViewModel.java`

```java
void addRetainedFragment(@NonNull Fragment fragment) {
    if (mIsStateSaved) {
        if (FragmentManager.isLoggingEnabled(Log.VERBOSE)) {
            Log.v(TAG, "Ignoring addRetainedFragment as the state is already saved");
        }
        return;
    }
    if (mRetainedFragments.containsKey(fragment.mWho)) {
        return;
    }
    mRetainedFragments.put(fragment.mWho, fragment);
    if (FragmentManager.isLoggingEnabled(Log.VERBOSE)) {
        Log.v(TAG, "Updating retained Fragments: Added " + fragment);
    }
}
```

这段代码并不复杂，当我们调用 Fragment# setRetainInstance(true) 时，最终会将 Fragment 添加到一个 ViewModel 中。ViewModel 时具备在 Activity 重建时恢复数据的能力的，**现在的问题转换为 ViewModel 为什么可以恢复数据**？

简单来说，在 Activity 销毁时，最终会调用 Activity#retainNonConfigurationInstances() 保存 ActivityClientRecord，并托管给 ActivityManagerService。这个过程就相当于把 Fragment 保存到更长的生命周期了。关于 ViewModel 的具体分析，我后面会专门写一篇文章，期待吗？

