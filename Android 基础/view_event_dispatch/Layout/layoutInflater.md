> Android | 带你探究 LayoutInflater 布局解析原理

#### 前言

- 在 Android UI 中，经常需要用到 LayoutInflater 类，它的基本作用是将 xml 布局文件解析成 View / View 树。除了基本的布局解析功能，LayoutInflater 还可以用于实现 **动态换肤 、视图转换、属性转换**等需求。

### 1、获取 LayoutInflater 对象

```java
@SystemService(Context.LAYOUT_INFLATER_SERVICE)
public abstract class LayoutInflater {
    ...
}
```

首先，你需要获得 LayoutInflater 的实例，由于 LayoutInflater 是抽象类，不能直接创建对象。获取该对象的方法如下：

`1、View.inflate() 方法`

```java
public static View inflate(Context context, @LayoutRes int resource, ViewGroup root) {
    LayoutInflater factory = LayoutInflater.from(context);
    return factory.inflate(resource, root);
}
```

`2、Activity#getLayoutInfalter()`

```
public LayoutInflater getLayoutInflater() {
    return getWindow().getLayoutInflater();

```

3、PhoneWindow # getLayoutInflater

```java
private LayoutInflater mLayoutInflater;

public PhoneWindow(Context context) {
    super(context);
    mLayoutInflater = LayoutInflater.from(context);
}

public LayoutInflater getLayoutInflater() {
    return mLayoutInflater;
}
```

`4、LayoutInflater#from(Context)`

```java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

现在，看下 `getSystemService()`内的逻辑：

`ContextImpl.java`

```
@Override
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```

`SystemServiceRegistry.java`

```java
private static final Map<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS = new ArrayMap<String, ServiceFetcher<?>>();

static {
    ...
    1. 注册 Context.LAYOUT_INFLATER_SERVICE 与服务获取器
    关注点：CachedServiceFetcher
    关注点：PhoneLayoutInflater
    registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class, new CachedServiceFetcher<LayoutInflater>() {
        @Override
        public LayoutInflater createService(ContextImpl ctx) {
            注意：getOuterContext()，参数使用的是 ContextImpl 的代理对象，一般是 Activity
            return new PhoneLayoutInflater(ctx.getOuterContext());
        }});
    ...
}

2. 根据 name 获取服务对象
public static Object getSystemService(ContextImpl ctx, String name) {
    ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    return fetcher != null ? fetcher.getService(ctx) : null;
}

注册服务与服务获取器
private static <T> void registerService(String serviceName, Class<T> serviceClass, ServiceFetcher<T> serviceFetcher) {
    SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
}

3. 服务获取器创建对象
static abstract interface ServiceFetcher<T> {
    T getService(ContextImpl ctx);
}
```

可以看到，ContextImpl 内部通过 SystemServiceRegistry 来获取服务对象，逻辑并不复杂：

1. 静态代码块注册了 name- ServiceFetcher 的映射
2. 根据 name 获得了 ServiceFetcher
3. ServiceFetcher 创建对象

ServiceFetcher 的子类有三种类型，它们的 `getSystemServcice`都是线程安全的，主要差别体现在 单例范围，具体如下：

| ServiceFetcher子类                     | 单例范围      | 描述                             | 举例                                      |
| -------------------------------------- | ------------- | -------------------------------- | ----------------------------------------- |
| CachedServiceFetcher                   | ContextImpl域 | /                                | LayoutInflater、LocationManager等（最多） |
| StaticServiceFetcher                   | 进程域        | /                                | InputManager、JobScheduler等              |
| StaticApplicationContextServiceFetcher | 进程域        | 使用 ApplicationContext 创建服务 | ConnectivityManager                       |

对于 LayoutInflater 来说，服务获取器是 CachedServiceFetcher 的子类，最终获取的服务对象为 PhoneLayoutInflater

![](/../picture/flater01.PNG)

这里有一个重点，这句代码非常隐蔽，要留意：

```
 return new PhoneLayoutInflater(ctx.getOuterContext());
```

`LayoutInflater.java`

```java
public Context getContext() {
    return mContext; 
}

protected LayoutInflater(Context context) {
    mContext = context;
    initPrecompiledViews();
}
```

可以看到，实例化 PhoneLayoutInflater 时使用了 getOuterContext, 也就是参数使用的时 ContextImpl 的代理对象，一般就是 Activity 了，也就是说，在 Activity Fragment View Dialog 中，获取 LayoutInflater # getContext ，返回的就是 Activity。

> 小结

- 获取 LayoutFlater 对象只有通过 `LayoutInflater.from(context)`, 内部委派给 `Context#getSystemService(...)`, 线程安全；
- 使用同一个 Context 对象，获得的 LayoutInflater 是单例
- LayoutInflater 的实现类是 PhoneLayoutInflater

### 2、inflate 主流程源码分析

前面我们获取了 LayoutInflater 对象的过程，现在我们可以调用 `inflate` 进行布局解析了。`LayoutInflater#inflate(...)` 有多个重载方法，最终都会调用到：

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    1. 解析预编译的布局
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }
    2. 构造 XmlPull 解析器 
    XmlResourceParser parser = res.getLayout(resource);
    try {
    3. 执行解析
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

- `tryInflatePrecompiled(...)` 是解析预编译的布局
- 构造 XmlPull 解析器 XmlResourceParser
- 执行解析，是解析的主流程

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    1. 结果变量
    View result = root;
    2. 最外层的标签
    final String name = parser.getName();
    3. <merge>
    if (TAG_MERGE.equals(name)) {
        3.1 异常
        if (root == null || !attachToRoot) {
            throw new InflateException("<merge /> can be used only with a valid "
                + "ViewGroup root and attachToRoot=true");
        }
        3.2 递归执行解析
        rInflate(parser, root, inflaterContext, attrs, false);
    } else {
        4.1 创建最外层 View
        final View temp = createViewFromTag(root, name, inflaterContext, attrs);
        
        ViewGroup.LayoutParams params = null;

        if (root != null) {
            4.2 创建匹配的 LayoutParams
            params = root.generateLayoutParams(attrs);
            if (!attachToRoot) {
                4.3 如果 attachToRoot 为 false，设置LayoutParams
                temp.setLayoutParams(params);
            }
        }

        5. 以 temp 为 root，递归执行解析
        rInflateChildren(parser, temp, attrs, true);
        
        6. attachToRoot 为 true，addView()
        if (root != null && attachToRoot) {
            root.addView(temp, params);
        }

        7. root 为空 或者 attachToRoot 为 false，返回 temp
        if (root == null || !attachToRoot) {
            result = temp;
        }
    }
    return result;
}

-> 3.2
void rInflate(XmlPullParser parser, View parent, Context context, AttributeSet attrs, boolean finishInflate) {
    while(parser 未结束) {
        if (TAG_INCLUDE.equals(name)) {
            1) <include>
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
            2) <merge>
            throw new InflateException("<merge /> must be the root element");
        } else {
            3) 创建 View 
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            4) 递归
            rInflateChildren(parser, view, attrs, true);
            5) 添加到视图树
            viewGroup.addView(view, params);
        }
    }
}

-> 5. 递归执行解析
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

关于 `<include> & <merge>`, 后面再说。对于参数 `root & attachToRoot` 的不同情况，对应得到的输出不同，总结如下：

![](/picture/layoutInflater02.png)

### 3、createViewFromTag()

在 **第2节** 主流程代码中，用到了 `createViewFromTag()`, 它负责创建 View 对象：

```java
已简化
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs, boolean ignoreThemeAttr) {

    1. 应用 ContextThemeWrapper 以支持 android:theme
    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }

    2. 先使用 Factory2 / Factory 实例化 View，相当于拦截
    View view;
    if (mFactory2 != null) {
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {
        view = mFactory.onCreateView(name, context, attrs);
    } else {
        view = null;
    }

    3. 使用 mPrivateFactory 实例化 View，相当于拦截
    if (view == null && mPrivateFactory != null) {
        view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }

    4. 调用自身逻辑
    if (view == null) {
        if (-1 == name.indexOf('.')) {
            4.1 <tag> 中没有.
            view = onCreateView(parent, name, attrs);
        } else {
            4.2 <tag> 中有.
            view = createView(name, null, attrs);
        }
    }
    return view;     
}

-> 4.2 <tag> 中有.

构造器方法签名
static final Class<?>[] mConstructorSignature = new Class[] {
            Context.class, AttributeSet.class};

缓存 View 构造器的 Map
private static final HashMap<String, Constructor<? extends View>> sConstructorMap =
            new HashMap<String, Constructor<? extends View>>();

public final View createView(String name, String prefix, AttributeSet attrs) {
    1) 缓存的构造器
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }
    Class<? extends View> clazz = null;
    2) 新建构造器
    if (constructor == null) {
        2.1) 拼接 prefix + name 得到类全限定名
        clazz = mContext.getClassLoader().loadClass(prefix != null ? (prefix + name) : name).asSubclass(View.class);
        2.2) 创建构造器对象
        constructor = clazz.getConstructor(mConstructorSignature);
        constructor.setAccessible(true);
        2.3) 缓存到 Map
        sConstructorMap.put(name, constructor);
    }
    
    3) 实例化 View 对象
    final View view = constructor.newInstance(args);

    4) ViewStub 特殊处理
    if (view instanceof ViewStub) {
        // Use the same context when inflating ViewStub later.
        final ViewStub viewStub = (ViewStub) view;
        viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
    }
    return view;
}

----------------------------------------------------

-> 4.1 <tag> 中没有.

PhoneLayoutInflater.java

private static final String[] sClassPrefixList = {
    "android.widget.",
    "android.webkit.",
    "android.app."
};

已简化
protected View onCreateView(String name, AttributeSet attrs) {
    for (String prefix : sClassPrefixList) {
        View view = createView(name, prefix, attrs);
            if (view != null) {
                return view;
            }
    }
    return super.onCreateView(name, attrs);
}
```

- 应用 ContextThemeWrapper 以支持 `android:theme`, 这是处理针对特定 View 设置主题；
- 使用 Factory2 / Factory 实例化 View，相当于拦截
- 使用 `mPrivateFactory` 实例化 View，相当于拦截
- 使用 LayoutInflater 自身逻辑，分为：

小结：

1. 使用 Factory2 接口可以拦截实例化 View 对象的步骤
2. 实例化 View 的优先顺序为：Factory2 / Factory -> mPrivateFactory -> PhoneLayoutInflater;
3. 使用反射实例化 View 对象，同时构造器对象做了缓存

![](/picture/layoutInflater03.png)

### 4、Factory2 接口

现在我们来讨论 `Factory2` 接口，上一节提到，`Factory2` 可以拦截实例化 View 的步骤，在 LayoutInfalter 中有两个方法可以设置：

`LayoutInflater.java`

```java
方法1：
public void setFactory2(Factory2 factory) {
    if (mFactorySet) {
        关注点：禁止重复设置
        throw new IllegalStateException("A factory has already been set on this LayoutInflater");
    }
    if (factory == null) {
        throw new NullPointerException("Given factory can not be null");
    }
    mFactorySet = true;
    if (mFactory == null) {
        mFactory = mFactory2 = factory;
    } else {
        mFactory = mFactory2 = new FactoryMerger(factory, factory, mFactory, mFactory2);
    }
}

方法2 @hide
public void setPrivateFactory(Factory2 factory) {
    if (mPrivateFactory == null) {
        mPrivateFactory = factory;
    } else {
        mPrivateFactory = new FactoryMerger(factory, factory, mPrivateFactory, mPrivateFactory);
    }
}
```

现在，我们来看源码中哪里调用这两个方法：

> 4.1 setFactory2

在 AppCompatActivity & AppCompatDialog 中，相关源码简化如下：

`AppCompatDialog.java`

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    设置 Factory2
    getDelegate().installViewFactory();
    super.onCreate(savedInstanceState);
    getDelegate().onCreate(savedInstanceState);
}
```

`AppCompatActivity.java`

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    final AppCompatDelegate delegate = getDelegate();
    设置 Factory2
    delegate.installViewFactory();
    delegate.onCreate(savedInstanceState);
    夜间主题相关
    if (delegate.applyDayNight() && mThemeId != 0) {
        if (Build.VERSION.SDK_INT >= 23) {
            onApplyThemeResource(getTheme(), mThemeId, false);
        } else {
            setTheme(mThemeId);
        }
    }
    super.onCreate(savedInstanceState);
}
```

`AppCompatDelegateImpl.java`

```java
public void installViewFactory() {
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    if (layoutInflater.getFactory() == null) {
        关注点：设置 Factory2 = this（AppCompatDelegateImpl）
        LayoutInflaterCompat.setFactory2(layoutInflater, this);
    } else {
        if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImpl)) {
            Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                        + " so we can not install AppCompat's");
        }
    }
}
```

`LayoutInflaterCompat.java`

```java
public static void setFactory2(@NonNull LayoutInflater inflater, @NonNull LayoutInflater.Factory2 factory) {
    inflater.setFactory2(factory);

    if (Build.VERSION.SDK_INT < 21) {
        final LayoutInflater.Factory f = inflater.getFactory();
        if (f instanceof LayoutInflater.Factory2) {
            forceSetFactory2(inflater, (LayoutInflater.Factory2) f);
        } else {
            forceSetFactory2(inflater, factory);
        }
    }
}
```

可以看到，在 AppCompatDialog & AppCompatActivity 初始化时，都通过 `setFactory2` 设置了拦截器，设置的对象是

**AppCompatDelegateImpl**:

```java
已简化
class AppCompatDelegateImpl extends AppCompatDelegate
        implements MenuBuilder.Callback, LayoutInflater.Factory2 {

    @Override
    public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        return createView(parent, name, context, attrs);
    }

    @Override
    public View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs) {
        if (mAppCompatViewInflater == null) {
            mAppCompatViewInflater = new AppCompatViewInflater();
        }
    }
    委托给 AppCompatViewInflater 处理
    return mAppCompatViewInflater.createView(...)
}
```

AppCompatViewInflater 与 LayoutInflater 的核心流程差不多，主要差别是前者会将 `<TextView>` 等标签解析为 `AppCompatTextView` 对象：

`AppCompatViewInflater.java`

```java
final View createView(...) {
    ...

    switch (name) {
        case "TextView":
            view = createTextView(context, attrs);
            break;
        ...
        default:
            view = createView(context, name, attrs);
    }
    return view;
}

@NonNull
protected AppCompatTextView createTextView(Context context, AttributeSet attrs) {
    return new AppCompatTextView(context, attrs);
}
```

> ### 4.2 setPrivateFactory()

`setPrivateFactory()`是 hide 方法，在 Activity 中调用，相关源码简化如下：

`Activity.java`

```java
final FragmentController mFragments = FragmentController.createController(new HostCallbacks());

final void attach(Context context, ActivityThread aThread,...) {
    attachBaseContext(context);
    mFragments.attachHost(null /*parent*/);

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    关注点：设置 Factory2
    mWindow.getLayoutInflater().setPrivateFactory(this);
    ...
}
```

可以看到，这里设置的 Factory2 其实就是 Activity 本身（this), 这说明 Activity 也实现了 Factory2：

```java
public class Activity extends ContextThemeWrapper implements LayoutInflater.Factory2,...{

    public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        if (!"fragment".equals(name)) {
            return onCreateView(name, context, attrs);
        }

        return mFragments.onCreateView(parent, name, context, attrs);
    }
}
```

