# Android 全面解析 Context 机制



## 引言

#### Handler 机制

> Handler 机制可以让我们避免手写 死循环和输入阻塞 来不断地获取用户输入以及避免线程直接结束。而是采用事务驱动型设计，使用 Handler 消息机制，让 AMS 可以控制整个程序地运行逻辑

我们知道 android 的 程序是通过main 方法跑起来的，然后通过 handler 机制来控制程序的运行，那么四大组件和普通的Java类有什么区别呢？为什么同样是 Java类，而ActivityThread、Acitvity等等这些类就显得那么特俗呢？我们的代码、写的布局是通过什么路径使用系统资源展示再屏幕上的呢？这一切就设计到我们的今天的主角：Context

## 什么是 Context

回想到最初学习 Android 开发的时候，第一用到Context 是什么时候？应该是Toast，其常规用法是：

```java
Toast.makeText(this, "我是toast", Toast.LENGTH_SHORT).show()
```

当初也不知道什么是 Context，只知道它需要一个 Context 类型，把 Activity 对象传进去即可。从此 Context 贯穿在开发过程中的方方面面，但是始终不了解这个 Context 到底是什么东西？为什么要用到这个对象？首先看一下官方对 Context 的一个解释：

```java
/**
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
public abstract class Context {...}
```

> 这是关于应用程序环境的全局信息的接口。这是一个抽象类，由Android系统提供。它允许访问程序特定的资源和类文件，以及向上调用应用程序级别的操作，比如启动活动、广播和接收intent。

可以看到Context最重要的作用是获取全局信息、访问系统资源、调用应用程序级别的操作。可能对于这些作用没有什么印象，想一下，如果没有context，我们如何做到以下操作：

- 弹出一个toast
- 启动一个 Activity
- 获取程序布局文件、drawable 文件等
- 访问数据库

这些平时看似简单的操作，一旦失去了 context 将无法执行。这些行为都有一个共同点：**需要与系统交互**。四大组件为什么配为组件，而我们写的就只能叫做一个普通的 Java 类，正是因为 context 的这些功能让四大组件有了不一样的能力。简单来说，context 就是：

> 应用程序和系统之间的桥梁，应用程序访问系统各种资源的接口

我们一般使用 context 最多的是两种场景：直接调用 context 的方法和调用接口时需要 context 参数，这些行为都意味着我们需要访问系统相关的资源。

那 context 时从哪里来的？AMS！ AMS 是系统级进程，拥有访问系统级别操作的权利，应用程序的启动收到 AMS的控制，在程序启动的过程中，AMS会把一个凭证通过跨进程通信给到我们的应用程序，我们的程序会把这个凭证封装成  context，并提供一系列的接口，这样我们的程序就也就可以很方便的访问系统资源了。这样的好处是：

> 系统可以对应用程序级别的操作进行调控，限制各种场景下的权限，同时也可以防止恶意攻击

如 Application 类的 context 和 Activity 的context 的权利是不一样的，生命周期也不一样。对于想要操作 系统攻击用户的程序也进行了阻止，没有获得允许的 Java 类没有任何权利，而 Activity 开放给用户也只有部分有限的权利。而我们开发者获取 context 的途径，也只有从 activity、Application等组件中获取。

因而，什么是 context？Context 是应用程序与系统之间沟通的桥梁，是应用程序访问系统资源的接口，同时也是系统给应用程序的一张权限凭证，有了context，一个 Java类才可以被称之为组件。

## Context 家族

上一部分我们了解到了什么是 context以及 context的重要性，这一部分就来了解一下context在源码中的子类继承情况。先看一个图：

![](/picture/data-01.image)

最顶层是 `Context抽象类`,它定义了一套与系统交互的接口，ContextWrapper 继承自 Context，但是并没有真正的实现 Context 中的接口，而是把接口的实现都交给了 ContextImpl，ContextImpl 是 context 接口的真正实现者，从 AMS拿来的凭证也是封装到了 ContextImpl中，然后赋值给了 ContextWrapper，这里运用到了一种模式：装饰者模式。Application 和 Service 都继承自 ContextWrapper,那么它们也就拥有Context 的接口方法且自身就是 context，方便开发者的使用。Activity 比较特俗，因为它是有界面的，所以它需要一个主题：Theme，ContextThemeWrapper 在 ContextWrapper 的基础上增加与主题相关的操作。

这样设计有这样设计的优点：

- Activity 等可以更加方便地使用 Context，可以把自身当作context 来使用，遇到需要 context  接口直接把自身传进去即可
- 运用装饰者模式，向外屏蔽 ContextImpl 地内部逻辑，同时当需要修改 ContextImpl 地逻辑实现，ContextWrapper 地逻辑几乎不需要修改
- 更方便地扩展不同情景下地逻辑，如 service 和 activity，情景不同，需要地接口方法也不同，但是与系统交互地接口是相同的。使用装饰者模式可以扩展出很多的功能，同时只需要把 ContextImpl 对象传进去赋值即可。

## Context 的分类

前面讲到 Context 的家族体系时，了解到他的最终实现类有：Application、Activity、Service，ContextImpl 被前三者持有，是Context 接口的真正实现，那么这里讨论一下这三者有什么不同，和使用时需要注意的问题。

#### Application

Application  是全局 Context，整个应用程序只有一个，它可以访问到应用程序的包信息等资源信息，获取 Application 的方法有两个：

```java
context.getApplicationContext()
activity.getApplication()
```

通过 context 和 activity 都可以获取到 Application，那这两个方法有什么区别？没有区别，我们打印来看一下：

```java
override fun onCreate(savedInstanceState: Bundle?) {
    ...
    Log.d("strong", "application：$application")
    Log.d("strong", "applicationContext：$applicationContext")
}
```

打印可以看到确实是同一个对象，但为什么需要提供两个一样的作用的方法？ `getApplication`方法更加直观，但是只能在 Activity 中调用。`getApplicationContext()`适用范围更广，任意一个 context 对象都可以调用这个方法。

Application 类的 Context 的特点是生命周期长，在整个应用运行期间它都会存在。同时我们可以自定义 Application，并在里面做一些全局的初始化操作，或者写一个静态的 Context 供给全局获取，不需要在方法中传入 Context，如：

```java
class MyApplication : Application(){
    // 全局context
    companion object{
        lateinit var context: Context
    }
    override fun onCreate() {
        super.onCreate()
        // 做全局初始化操作
        RetrofitManager.init(this)
        context = this
    }
}
```

这样我们就可以在应用启动的时候对一些组件进行初始化，同时可以通过 MyApplication.context 来获取 Application 对象。

但是 ！！！！ 请不要把 Application当作工具类来使用。用于 Application 获取的便利性，有开发者会在 Application 中编写一些工具方法，全局获取使用，这样是不行的，自定义 Application 的目的是在程序启动的时候做全局初始化的工作，而不能拿来代替工具类，这严重的违背谷歌设计 Application 的原则，也违背了 Java 代码规范的单一职责原则。

### 四大组件

Activity 继承自 ContextThemeWrapper，是一个拥有主题的 context  对象。Activity 常用于与UI有关操作，如添加 window 等，常规使用可以直接使用 activity.this。

Service继承自 ContextWrapper，也可以和 Activity 一样直接使用 service.this 来使用 context。和 activity 不同的是，Service 没有界面，所以也不需要主题。

ContentProvider 使用的是 Application 的 context，Broadcast 使用的是 activity 的 context，这两点在后面会进行源码分析。

### BaseContext

嗯？baseContext  是什么？把这个拿出来单独讲，细心的读者可能会发现 activity 中有一个方法： `getBaseContext`。这个是 ContextWrapper 中的 mBase 对象，也就是 ContextImpl，也就是 context 接口的真正逻辑实现。

### Context 的使用问题

使用 context 的最重要的问题之一是 **注意内存泄露**。不同的 context的生命周期不同，Application 是在应用存在的期间会一直存在，而Activity 是会随着界面的销毁而销毁，如果当我们的代码长时间持有了 activity 的 context，如静态引用或者 单例类，那么会导致 activity 无法被释放，如下面的代码：

```java
object MyClass {
    lateinit var mContext : Context
    fun showToast(context : Context){
        mContext = context
    }
}
```

单例类在应用持续的时间都会一直存在，这样 context 也就会一直被持有，activity 无法被回收，导致内存泄露。

那，我们就都换成 Application 不就可以了，如下：

```java
object MyClass {
    lateinit var mContext : Context
    fun showToast(context : Context){
        mContext = context.applicationContext
    }
}
```

答案是：不可以。什么时候可以使用 Application？**不涉及 UI 以及启动Activity操作**。Activity 的 context 是拥有主题属性的，如果使用 Application 来操作 UI，那么会丢失自定义的主题，采用系统默认的主题。同时，有些UI 操作只有 Activity 可以执行，如弹出 dialog，这涉及到 window 的 token 问题。这也是官方对 context 的不同权限色痕迹，**没有界面的 context，就不应该有操作界面的权利**。使用 Application 启动的 Activity 必须指定 task 以及标记 为 singleTask，因为 Application 是没有任务栈的，需要重新开一个新的任务栈。因此，**我们需要根据不同的 Context 的不同职责来执行不同的任务**。























