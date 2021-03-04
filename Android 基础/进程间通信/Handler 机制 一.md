# Android 全面解析之 Handler 机制1

## 前言

本文是系列文章的开始，只要介绍什么是 Handler、Handler 需要解决什么问题、以及如何使用 Handler。

## 正文

##### 什么是 Handler？

准确是 Handler 机制，Handler 是 Handler 机制中的一个角色。只是我们对 Handler 的接触比较多，所以经常以 Handler 来代称。

> Handler 机制是 Android 中基于单线消息队列模式的一套线程消息机制

他的本质是消息机制，负责消息的分发和处理。这样说可能很抽象。什么是“单线消息队列模式”？什么是“消息”？

通俗点来说，**每个线程**都有一个“流水线”，我们可以往这条流水线上面放“消息”，流水线的末端有人员去处理这些消息。因为流水线是单线的，所有消息都必须按照先来后到的形式依次处理（在 Handler 机制中有“加急线”：同步屏障）。如下图：

![](/picture/data-01.image)

**放什么消息以及怎么去处理消息，是需要我们去定义的**。Handler 机制相当于提供了这样的一套模式，我们只需要“放消息到流水线上”，“编写这些消息的处理逻辑”就可以了，流水线会源源不断的把消息运送到末端处理。最后注意重点“**每个线程只有一个”流水线“**，他的基本范围是线程，负责线程内的通信以及线程间的通信。每个线程可以看成一个厂房，每个厂房只有一个生产线。

## 两个关键问题

连接 Handler 的作用需要了解 Handler 背景下的两个关键问题：

> - 不能在非 UI 创建线程去操作 UI
> - 不能在主线程执行耗时任务

我们普遍的认知是：不能在非主线程 更新 UI。但这个是不准确的，如果我们在子线程更新了 UI，看看报错信息是什么：

![](/picture/data-02.image)

**只有创建视图层次结构的原始线程才能访问其视图**。但为什么我们一直都说是非主线程不能更新UI？这是因为我们的界面一般都是在主线程进行绘制的，所以界面的更新也就一般都限制在主线程内。这个异常是在 ViewRootImpl.checkThread()方法中抛出来的。那可以绕过吗？当然可以，在他还没创建出来的时候就可以偷偷更新 UI 了。阅读过 Activity 启动流程的同学知道，ViewRootImpl 是在 onCreate 方法之后被创建的，所以可以在 onCreate 方法中创建一个子线程偷偷的去更新 UI。

为什么不能在子线程去更新 UI？因为这会让界面产生不可预期的效果。例如主线程在绘制一个按钮的时候，绘制一半另一个线程突然把线程的大小改成两倍大，这个时候再回去主线程继续执行绘制逻辑。这个绘制的效果就会出现问题，所以 UI 的访问是绝对不能并发的。但是，子线程更新 UI想要加锁的话，这样又会造成另外的问题：**界面卡顿**。锁对于性能是有消耗的，是比较重量级别的操作，而 UI的要求是快准狠，加锁的会让操作性能大打折扣。Handler 就是解决这个问题的。

第二个问题，不能早主线程执行耗时操作。耗时操作包括网络请求、数据库操作等等，这些操作会导致 ANR（Application Not Responding)。这个是比较好理解的，没有什么问题，但是这个两个问题结合起来就有大问题了。数据请求一般是耗时操作，必须在子线程进行请求，而当请求完成之后又必须更新 UI，UI 只能在主线程进行更新，这就导致必须**切换线程执行代码**。那么 Handler 得到重要性就体现出来了。

## 为什么要有 Handler？

先说结论：

> - 切换代码执行的线程
> - 按顺序规则的处理消息，避免并发
> - 阻塞线程，避免让线程结束
> - 延迟处理消息

第一个作用是最明显的也是常用的，上一部分已经讲了 Handler 存在的必要性，Android 限制了**不能在非 UI 创建线程去操作 UI**，同时**不能在主线程执行耗时操作**，所以我们一般在子线程执行网络请求等耗时操作请求数据，然后再切换到主线程更新 UI。这个时候必须用到 Handler 去切换线程执行代码。

这里有一个误区：我们的 Activity 是执行在主线程的，我们在网络请求完成之后回调主线程的方法不就切换到了主线程了吗？这个错误是很低接的。代码并没有限制运行在哪个线程，代码执行的线程环境取决于你的执行逻辑是在哪个线程。举个例子：现在有一个方法test(),然后有两个不同的线程去调用他：

```java
new Thread(){
    // 第一个线程调用
    test();
}.start();

new Thread(){
    // 第二个线程调用
    test();
}
```

此时虽然都是test 这个方法，但是他的执行逻辑是由不同的线程调用的，所以他是执行在两个不同的线程环境下。而我们想要把逻辑切换到另外一个进程去执行的时候，就需要用到 Hanler 来切换线程。

第二个作用可能会看起来有点蒙。但其实他解决了另一个问题：并发操作。虽然切换线程解决了，如果主线程正在绘制一个按钮，刚好测量好按钮的长宽，突然一个子线程来一个请求打断了，先停下来这边的绘制工作，把按钮改成了两倍大，然后逻辑回来继续绘制，这个时候之前的测量的长度已经是不准确的了，绘制的结果肯定也是不准确的，怎么解决？**单线消息队列**。在讲什么是 Handler 那部分简单讲过，相当于一个流水线一样的模型。子线程的请求会变成一个一个的消息，然后主线程依次进行处理，这样就不会出现执行一般被打断的情况了。

同样这种模型也不止解决 UI 并发问题，在 ActivityThread 中有一个 H 类，他其实就是一个 Handler。在 ActivityThread 中定义了一百种消息类型及对应的处理逻辑。这样，当需要让 ActivityThread 处理某一个逻辑的时候，只需要发送对应的消息给他即可，而且可以保证消息按顺序执行，例如先调用 onCreate 在调用 onResume。如果没有 Handler 的话，就需要让 ActivtyThread 开放接口，同时还需要不断的执行回调保证任务按顺序执行。

我们执行一个Java 程序的时候，从 main 方法入口，执行完之后，就退出了，但是我们Android 程序肯定是不行的，他需要一直等待用户的操作，而 Handler 机制就解决了这个问题，但消息队列中没有任务的时候，就会把线程阻塞，等待到有新的任务的时候，再重新启动消息处理。

第四个作用让延迟处理消息得到了最佳的解决方案。假如你想让应用启动5秒之后界面弹出一个对话框，没有 Handler得情况下，会如何处理？开一个 Thread 睡眠 5秒后，但是若有多个延时任务呢？并且创建线程是一个很重量级别的操作并且创建线程的数目也是有限的。而直接给 Handler 发送延迟对应时间的消息，他会在对应时间之后准时处理消息（当然有特殊情况，如单间消息处理时间过长或者同步屏障），而且无论发送多少延迟消息都不会对性能有影响，同时，ANR 的时间也是通过该功能实现的。

## 如何使用Handler

我们平常使用 Handler 有两种不同的创建方式，但是总体流程是相同的：

- ```java
  创建lopper
  使用 looper 创建 Handler
  启动 looper
  使用 Handler 发送消息
  ```

Looper 可以理解为循环器，就像“流水线”上的滚带。每个线程只有一个 Looper，通常主线程已经创建好了，追溯应用程序启动流程可以知道启动过程调用了 Looper.prepareMainLooper，而在子线程必须使用如下方法来初始化 Looper：

```java
Looper.prepare()
```

第二部就是创建 Handler，也是很熟悉的一步，我们通常有两种方法来创建 Handler：创建 callback 对象和继承。如下：

```java
public class MainActivity extends AppComposeActivity{
    ...;
    // 第一种方法：使用callBack创建handler
    public void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        Handler handler = Handler(Looper.myLooper(),new CallBack(){
            public Boolean handleMessage(Message msg) {
                TODO("Not yet implemented")
            }
        });
    }
    
    // 第二种方法：继承Handler并重写handlerMessage方法
    static MyHandler extends Hanlder{
        public MyHandler(Looper looper){
            super(looper);
        }
        @Override
        public void handleMessage(Message msg){
            super.handleMessage(msg);
            // TODO(重写这个方法)
        }
    }
}
```

注意第二种方法，需要使用静态内部类，不然可能会造成内存泄漏。原因是非静态内部类会持有外部类的引用，而 Handler 发出的 Message 会持有 Handler 的引用。如果这个 message 是一个延迟消息的话，此时 Activity被退出了，但是 Message 依然在流水线上，Message- Handler - Activity，那么 Activity就无法被回收，导致内存泄漏。

两个方法各有千秋，继承可以实现比较复杂的逻辑，callBack比较适合简单的逻辑。

然后再调用 Looper 的 loop 方法来启动 Looper:

```java
Looper.loop()
```

最后就是使用 handler 来发送消息了。当我们获得 handler 的实例之后，就可以通过它的 sendMessage 方法和 post 相关的方法来发送消息，如下：

```java
handler.sendMessage(msg);
handler.sendMessageDelayed(msg,delayTime);
handler.post(runnable);
handler.postDelayed(runnable,delayTime);
```

然后一般情况下是哪个 Handler 发出的消息，最终由哪个Handler 来处理。这样只要我们拿到 Handler 对象，就可以向对应的线程发送消息了。









