> PagerAdapter 简单介绍

> 使用场景

- 轮播图：ViewPager + 自定义 PagerAdapter
- fragment: TabLayout + ViewPager + FragmentPagerAdapter + Fragment

> PagerAdapter 抽象方法

#### 子类继承 PagerAdapter 抽象方法

- Object instantiateItem(ViewGroup container, int position)
  - 一句话：**要显示的页面或需要缓存的页面，会调用这个方法进行布局的初始化**
  - 这个方法是 ViewPager 需要加载某个页面的时候调用，container 就是 View Pager 自己，position 页面索引
  - 我们需要实现的是添加一个 View 到 container 中，然后返回一个跟这个 view 能够关联起来的对象，这个对象可以是 view 本身，也可以是其他对象（比如 FragmentPagerAdapter 返回的就是一个 Fragment），关键是在 isViewFromObject 能够将 view 和这个 object 关联起来
- void destroyItem(ViewGroup container, int position, Object object)
  - 一句话：**当 ViewPager 需要销毁一个页面时调用，我们需要将 position 对应的 view 从 container 中移除**
  - 这时参数除了 position 就只有 Object，其实就是上面 instantiateItem 方法返回的对象，这时要通过 object 找到对应的 View，然后将其移除掉，如果你的 instantiateItem 方法返回的就是 View，这里就直接转成 View 移除即可：container.removeView((View) object);如果不是，一般会自己创建一个 List 缓存 View 列表，然后根据 position 从 List 中找到相应的 View 移除
  - FragmentPagerAdapter 的实现是：mCurTransaction.detach((Fragment) object), 其实也就是将 fragment 的 view 从 container 中移除
- isViewFromObject(View view, Object object)
  - 一句话：**这个方法用于判断是否由对象生成界面，官方建议直接返回 return view == object**
  - 从名称理解起来像是判断 view 是否来自 object，进一步解释应该是上面 instantiateItem 方法中
  - 向 container 中添加的 view 和方法返回的对象两者之前一对一的关系；因为在 View Pager 内部有个方法叫 infoForChild
  - 这个方法是通过 view  去找到对应页面信息缓存类 ItemInfo（内部调用了 isViewFromObject),如果找不到，说明这个 view 是个野孩子，ViewPager 会认为不是 Adapter 提供的 View，所以这个 view 不会显示出来
- int getItemPosition(Object object)
  - 该方法是判断当前 object 对应的 view 是否需要更新，在调用 notifyDataSetChanged 时会间接触发该方法
  - 如果返回 POSITION_UNCHANGED 表明该页面不需要更新，如果返回 POSITION_NONE 则表示页面无效了，需要销毁并触发 destroyItem 方法（并且有可能调用 instantiateItem 重新初始化这个页面）

> PagerAdapter 原理介绍

- ViewPager + PagerAdapter 的合作关系
  - ViewPager 来控制一页界面构造和销毁的时机，使用回调来通知 PagerAdapter 具体做什么，PagerAdapter 只需要按照相应的步骤做。当然为了使用的更好、提供更多的功能，又建议了使用 View 的回收工作和管理工作，同时提供当数据改变时的界面刷新工作

  - instantiateItem(ViewGroup,int)

    - 构造指定位置的页面。Adapter 负责在这个方法中添加 view 到容器中。在 FragmentPagerAdapter 和 FragmentStatePagerAdapter 中，都是返回一个构造的 Fragment

  - destroyItem(ViewGroup,populate,Object)

    - 移除指定位置的页面。adapter 负责从容器中移除 view，最后是在 finishUpdate(ViewGroup)保证完成的。在FragmentPagerAdapter和FragmentStatePagerAdapter中，分别使用FragmentTransition.detach(Fragment)和FragmentTransition.remove(Fragment)来逻辑上销毁Fragment

  - finishUpdate(ViewGroup)

    - 当页面的显示变化完成时调用。在这里，你一定保证所有的页面从容器中合理的添加或移除掉

  - setPrimaryItem(ViewGroup,int,Object)

    - 被 View Pager 调用来通知 adapter 此时那个 item 应该被认为是主要的界面，这个页面将在当前页面被展示给用户。正是因为这个方法，才有在 view pager 中实现 fragment 懒加载的机制

  - getPageTitle(int): 返回每页的标题，多用于关联indicator

    getPageWidth(int): 返回指定的页面相对于ViewPager宽度的比例，范围(0.f-1.f]。默认值为1.f, 即占满整个屏幕。如果是0.5f, 那么在初始状态下，默认会出现前两个页面，而primary主页面是在ViewPager的起始位置（通常是屏幕左侧），直到最后一个页面在屏幕右侧，如果总共5个页面，返回值为0.2f, 那么将一次性出现所有的页面.

> PagerAdapter 销毁和缓存

在ViewPager三种Adapter的子view创建和销毁的方法添加相关的日志代码，如下

```
@Override
public void destroyItem(ViewGroup container, int position, Object object) {
    Log.d("yc", "destroyItem:" + position);
    //...省略部分代码
}

@Override
public Object instantiateItem(ViewGroup container, int position) {
    Log.d("yc", "instantiateItem:" + position);
    //...省略部分代码
}
```

- 滑动 ViewPager 翻页，观察控制台的输出，三种 Adapter 针对不同界面、不同滑动方法的翻页情况打印如下：

从图中我们可以看到，三种Adapter在相同的情况下，ViewPager的子页面销毁和创建时机是一样。通常所听到的都是FragmentPagerAdapter会缓存所有的Fragment子项，而上图中我们看到的是在滑动的过程中它的destroyItem方法被调用了，而在滑动回来时相对应的子项Fragment也确实调用instantiateItem方法。这样看来根本就没有缓存……

但是仔细对比了一下三个Adapter创建视图的过程，发现上面推论有所欠缺。 

- 因为在使用Fragment作为子视图时，我们是通过getItem方法返回Fragment的，单纯从这里打印instantiateItem的调用不代表Fragment真的完全被重新创建了（重新创建代表需要重新add，即从头走一遍生命周期，但是在这里不能证明），也可以通过两个FragmentAdapter中instantiateItem的实现证明（观察getItem方法的调用条件），所以又在Fragment对应的两种Adapter的getItem中添加相应的log代码，如下：

```
@Override
public Fragment getItem(int position) {
    Log.d("ccc", "getItem:" + position);
    return fragmentList.get(position);
}
```

通过上图我们可以看到，FragmentPagerAdapter在最后向右边划回来时并没有调用getItem方法（getItem是创建一个新的Fragment），这也就说明了他没有重新创建Fragment，证明了它会缓存所有Fragment，那么它到底在哪里做了缓存呢？具体看FragmentPagerAdapter分析……

> 自定义 PagerAdapter

- 比如，引导页使用 View Pager，这个时候动态管理的 Adapter，可以每次都会创建新 View，销毁旧 View。节省内存消耗性能。

```
public abstract class AbsDynamicPagerAdapter extends PagerAdapter {

	@Override
	public boolean isViewFromObject(@NonNull View arg0, @NonNull Object arg1) {
		return arg0==arg1;
	}

	@Override
	public void destroyItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
		container.removeView((View) object);
	}
	
	@Override
	public int getItemPosition(@NonNull Object object) {
		return super.getItemPosition(object);
	}

	@NonNull
	@Override
	public Object instantiateItem(@NonNull ViewGroup container, int position) {
		View itemView = getView(container,position);
		container.addView(itemView);
		return itemView;
	}

	/**
	 * 创建view
	 * @param container					container
	 * @param position					索引
	 * @return
	 */
	public abstract View getView(ViewGroup container, int position);
}
```

比如，常见有无限轮播图，可以自动轮播，大家应该用的特别多。这个时候可以优化自定义轮播图的PagerAdapter，创建集合用来存储view，再次用的时候先取集合，没有就创建。而不是频繁创建视图。

```
public abstract class AbsLoopPagerAdapter extends PagerAdapter {
    private BannerView mViewPager;
    /**
     * 用来存放View的集合
     */
    private ArrayList<View> mViewList = new ArrayList<>();
    /**
     * 刷新全部
     */
    @Override
    public void notifyDataSetChanged() {
        mViewList.clear();
        initPosition();
        super.notifyDataSetChanged();
    }

    /**
     * 获取item索引
     *
     * POSITION_UNCHANGED表示位置没有变化，即在添加或移除一页或多页之后该位置的页面保持不变，
     * 可以用于一个ViewPager中最后几页的添加或移除时，保持前几页仍然不变；
     *
     * POSITION_NONE，表示当前页不再作为ViewPager的一页数据，将被销毁，可以用于无视View缓存的刷新；
     * 根据传过来的参数Object来判断这个key所指定的新的位置
     * @param object                        objcet
     * @return
     */
    @Override
    public int getItemPosition(@NonNull Object object) {
        return POSITION_NONE;
    }

    /**
     * 注册数据观察者监听
     * @param observer                      observer
     */
    @Override
    public void registerDataSetObserver(@NonNull DataSetObserver observer) {
        super.registerDataSetObserver(observer);
        initPosition();
    }

    private void initPosition(){
        if (getRealCount()>1){
            if (mViewPager.getViewPager().getCurrentItem() == 0&&getRealCount()>0){
                int half = Integer.MAX_VALUE/2;
                int start = half - half%getRealCount();
                setCurrent(start);
            }
        }
    }

    /**
     * 设置位置，利用反射实现
     * @param index                         索引
     */
    @TargetApi(Build.VERSION_CODES.KITKAT)
    private void setCurrent(int index){
        try {
            Field field = ViewPager.class.getDeclaredField("mCurItem");
            field.setAccessible(true);
            field.set(mViewPager.getViewPager(),index);
        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    public AbsLoopPagerAdapter(BannerView viewPager){
        this.mViewPager = viewPager;
    }

    @Override
    public boolean isViewFromObject(@NonNull View arg0, @NonNull Object arg1) {
        return arg0==arg1;
    }

    /**
     * 如果页面不是当前显示的页面也不是要缓存的页面，会调用这个方法，将页面销毁。
     * @param container                     container
     * @param position                      索引
     * @param object                        object
     */
    @Override
    public void destroyItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
        container.removeView((View) object);
        Log.d("PagerAdapter","销毁的方法");
    }

    /**
     *  要显示的页面或需要缓存的页面，会调用这个方法进行布局的初始化。
     * @param container                     container
     * @param position                      索引
     * @return
     */
    @NonNull
    @Override
    public Object instantiateItem(@NonNull ViewGroup container, int position) {
        int realPosition = position%getRealCount();
        View itemView = findViewByPosition(container,realPosition);
        container.addView(itemView);
        Log.d("PagerAdapter","创建的方法");
        return itemView;
    }

    /**
     * 这个是避免重复创建，如果集合中有，则取集合中的
     * @param container                     container
     * @param position                      索引
     * @return
     */
    private View findViewByPosition(ViewGroup container, int position){
        for (View view : mViewList) {
            if (((int)view.getTag()) == position&&view.getParent()==null){
                return view;
            }
        }
        View view = getView(container,position);
        view.setTag(position);
        mViewList.add(view);
        return view;
    }


    @Deprecated
    @Override
    public final int getCount() {
        //设置最大轮播图数量 ，如果是1那么就是1，不轮播；如果大于1则设置一个最大值，可以轮播
        //return getRealCount();
        return getRealCount()<=1?getRealCount(): Integer.MAX_VALUE;
    }

    /**
     * 获取轮播图数量
     * @return                          数量
     */
    public abstract int getRealCount();

    /**
     * 创建view
     * @param container                 viewGroup
     * @param position                  索引
     * @return
     */
    public abstract View getView(ViewGroup container, int position);
}

```

> ### PagerAdapter两个子类

PagerAdapter 的两个直接子类 FragmentPagerAdapter 和 FragmentStatePagerAdapter 。而我们常常会在 ViewPager 和 Fragment 结合使用的时候来使用这两个适配器

#### FragmentPagerAdapter

- FragmentPagerAdapter 它将每一个页面表示为一个 Fragment，并且每一个 Fragment 都将会保存到 FragmentManager 当中。而且，当用户没可能再次回到页面的时候，FragmentManager 才会将这个 Fragment 销毁。 
  - FragmentPagerAdapter：对于不再需要的 fragment，选择调用 onDetach() 方法，仅销毁视图，并不会销毁 fragment 实例。
- 使用 FragmentPagerAdapter 需要实现两个方法： 
  - public Fragment getItem(int position) 返回的是对应的 Fragment 实例，一般我们在使用时，会通过构造传入一个要显示的 Fragment 的集合，我们只要在这里把对应的 Fragment 返回就行了。
  - public int getCount() 这个上面介绍过了返回的是页面的个数，我们只要返回传入集合的长度就行了。
  - 使用起来是非常简单的，FragmentStatePagerAdapter 的使用也和上面一样，那两者到底有什么区别呢？
- 错误说法
  - 超出范围的Fragment会被销毁。所以之前，我一直认为的是，FragmentPagerAdapter中通常最多会保留3个Fragment, 超出左右两侧的Fragment将被销毁，滑动到时又会被重新构造。

PagerAdapter的实现类，使用将一直保留在FragmentManager中的Fragment来代表每一页，直到用户返回上一页。 

- 当用于典型地使用多静态化的Fragment时，FragmentPagerAdapter无疑是最好使用的，例如一组tabs. 每个用户访问过的页面的Fragment都将会保留在内存中，即使它的视图层在不可见时已经被销毁。这可能导致使用比较大数量的内存，因为Fragment实例持有任意数量的状态。如果使用大数据的页面，考虑使用FragmentStatePagerAdapter.
- 从上面可以看出，即使是超出可视范围和缓存范围之外的Fragment，它的视图将会被销毁，但是它的实例将会保留在内存中，所以每一页的Fragment至始至终都只需要构造一次而已。通常是在主页中使用FragmentPagerAdapter, 但是超出范围的Fragment的视图会被销毁，我们也可以在Fragment中缓存View来避免状态的丢失，也可以使用另外的机制，如缓存View的状态。

```
@Override
public Object instantiateItem(ViewGroup container, int position) {
    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }

    final long itemId = getItemId(position);

    // Do we already have this fragment?
    String name = makeFragmentName(container.getId(), itemId);
    Fragment fragment = mFragmentManager.findFragmentByTag(name);
    if (fragment != null) {
        if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
        mCurTransaction.attach(fragment);
    } else {
        fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
        mCurTransaction.add(container.getId(), fragment,
                makeFragmentName(container.getId(), itemId));
    }
    if (fragment != mCurrentPrimaryItem) {
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
    }

    return fragment;
}

@Override
public void destroyItem(ViewGroup container, int position, Object object) {
    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }
    if (DEBUG) Log.v(TAG, "Detaching item #" + getItemId(position) + ": f=" + object
            + " v=" + ((Fragment)object).getView());
    mCurTransaction.detach((Fragment)object);
}
```

从上面源码可以得出结论 

- 当被销毁时，Fragment并没有从FragmentTransition中移除，而是调用了FragmentTransition.detach(Fragment)方法，这样销毁了Fragment的视图，但是没有移除Fragment本身。
- detach：对应执行的是Fragment生命周期中onPause()-onDestroyView()的方法，此时并没有执行onDestroy和onDetach方法。所以在恢复时只需要attach方法即可（可以在FragmentPagerAdapter的instantiateItem方法中看到调用，对应源码下面给出），attach方法对应的是执行Fragment生命周期中onCreateView()-onResume()

####  FragmentStatePagerAdapter

- FragmentStatePagerAdapter:会销毁不再需要的 fragment，当当前事务提交以后，会彻底的将 Fragment 从当前 activity 的 fragmentManager 中移除，销毁时，会将其 onSaveInstanceState(Bundle state) 中的 bundle 信息保存下载，当用户切换回来，可以通过该 bundle 恢复生成新的 fragment，也就是说，你可以在 onSaveInstanceState(Bundle) 方法中保存一些数据，在 onCreate 中进行恢复创建

  - 使用 FragmentStatePagerAdapter 更省内存，但是销毁后重建也是需要时间的。一般情况下，如果你是制作主页面，就3、 4个 tab，那么可以选择使用 FragmentPagerAdapter，如果你是用于 ViewPager 展示数目特别多的条目时，那么建议使用 FragmentStatePagerAdapter

- PagerAdapter的实现类，使用Fragment来管理每一页。这个类也会管理保存和恢复Fragment的状态。 

  - 当使用一个大数量页面时，FragmentStatePagerAdapter将更加有用，工作机制类似于ListView. 当每页不再可见时，整个Fragment将会被销毁，只保留Fragment的状态。相对于FragmentPagerAdapter, 这个将允许页面持有更少的内存。

  ```
  @Override
  public Object instantiateItem(ViewGroup container, int position) {
      // If we already have this item instantiated, there is nothing
      // to do.  This can happen when we are restoring the entire pager
      // from its saved state, where the fragment manager has already
      // taken care of restoring the fragments we previously had instantiated.
      if (mFragments.size() > position) {
          Fragment f = mFragments.get(position);
          if (f != null) {
              return f;
          }
      }
  
      if (mCurTransaction == null) {
          mCurTransaction = mFragmentManager.beginTransaction();
      }
  
      Fragment fragment = getItem(position);
      if (DEBUG) Log.v(TAG, "Adding item #" + position + ": f=" + fragment);
      if (mSavedState.size() > position) {
          Fragment.SavedState fss = mSavedState.get(position);
          if (fss != null) {
              fragment.setInitialSavedState(fss);
          }
      }
      while (mFragments.size() <= position) {
          mFragments.add(null);
      }
      fragment.setMenuVisibility(false);
      fragment.setUserVisibleHint(false);
      mFragments.set(position, fragment);
      mCurTransaction.add(container.getId(), fragment);
  
      return fragment;
  }
  
  @Override
  public void destroyItem(ViewGroup container, int position, Object object) {
      Fragment fragment = (Fragment) object;
  
      if (mCurTransaction == null) {
          mCurTransaction = mFragmentManager.beginTransaction();
      }
      if (DEBUG) Log.v(TAG, "Removing item #" + position + ": f=" + object
              + " v=" + ((Fragment)object).getView());
      while (mSavedState.size() <= position) {
          mSavedState.add(null);
      }
      mSavedState.set(position, fragment.isAdded()
              ? mFragmentManager.saveFragmentInstanceState(fragment) : null);
      mFragments.set(position, null);
  
      mCurTransaction.remove(fragment);
  }
  ```

  ### 三种Adapter的总结

  - 三种Adapter的缓存策略 
    - PagerAdapter：缓存三个，通过重写instantiateItem和destroyItem达到创建和销毁view的目的。
    - FragmentPagerAdapter：内部通过FragmentManager来持久化每一个Fragment，在destroyItem方法调用时只是detach对应的Fragment，并没有真正移除！
    - FragmentPagerStateAdapter：内部通过FragmentManager来管理每一个Fragment，在destroyItem方法，调用时移除对应的Fragment。
  - 三个Adapter使用场景分析 
    - PagerAdapter：当所要展示的视图比较简单时适用
    - FragmentPagerAdapter：当所要展示的视图是Fragment，并且数量比较少时适用
    - FragmentStatePagerAdapter：当所要展示的视图是Fragment，并且数量比较多时适用。



