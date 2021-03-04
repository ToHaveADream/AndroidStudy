# Android 全面解析之 Handler 机制（三）：内部关键类

### 前言

本文内容主要介绍是Handler 的内部关键类：Message、MessageQueue、Looper。从本文开始介绍 Handler 的底层原理了。

### 正文

## Message

##### 概述

Message 是承载消息的类，主要关注他的内部属性

```java
// 用户自定义，主要用于辨别Message的类型
public int what;
// 用于存储一些整型数据
public int arg1;
public int arg2;
// 可放入一个可序列化对象
public Object obj;
// Bundle数据
Bundle data;
// Message处理的时间。相对于1970.1.1而言的时间
// 对用户不可见
public long when;
// 处理这个Message的Handler
// 对用户不可见
Handler target;
// 当我们使用Handler的post方法时候就是把runnable对象封装成Message
// 对用户不可见
Runnable callback;
// MessageQueue是一个链表，next表示下一个
// 对用户不可见
Message next;
```

##### 循环利用 Message

当我们获取 Message 的时候，官方建议是通过 Message.obtain() 方法进行获取，当使用完之后使用 Recycle()方法来回收循环利用，而不是直接 new 一个新的对象。

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; 
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

Message 维护了一个静态链表，链表头是 sPool，Message 有一个next 属性，本身就是一个链表结构。sPoolSync 是一个 object 对象，仅作为解决并发访问色痕迹。当我们调用obtain 来获取一个新的 Message 的时候，首先会检查链表中是否有空闲的 Message，如果没有则新建一个返回。

当我们使用完成之后，需要使用 recycle 进行回收，如果这个 Message 正在使用则会抛出异常，否则调用 recycleUnchecked 进行回收，把Message 中的内容进行清空，然后判断链表是否达到最大值（50），然后插入到链表中。

##### Message

Message 的作用就是承载消息，它的内部有很多的属性用户给用户赋值，同时 Message 本身也是一个链表结构，无论是在 MessageQueue 还是在 Message 内部的回收机制，都是使用这个结构来形成链表。一般来说我们不需要去调用 recycle 进行回收，在 Looper 中会自动把 Message 进行回收，后面会看到。

## MessageQueue

#### 概述

每个线程都只有一个MessageQueue，他是一个用于承载消息的队列，内部使用链表作为数据结构，所有等待处理的消息都会在这里排队。MessageQueue 是一个“修改版的 LinkQueue"。它有两个关键的方法：入队和出队（enqueueMessage 和 next）。这也是 MessageQueue 的重点所在。

Message 还有一个关键概念：线程休眠。当 MessageQueue 中没有消息或者都在等待中，则会让线程休眠，让出 CPU 资源，提高 CPU的利用率。进入休眠之后，如果需要继续执行代码则需要将线程唤醒。当方法暂时无法直接返回需要等待的时候，则可以将线程阻塞，即休眠，等待被唤醒的时候继续执行逻辑。

#### 关键方法

- 出队　-- next()

  next 方法主要是做消息出队工作

  ```java
  Message next() {
      // 如果looper已经退出了，这里就返回null
      final long ptr = mPtr;
      if (ptr == 0) {
          return null;
      }
      ...
      // 阻塞时间
      int nextPollTimeoutMillis = 0;
      for (;;) {
          if (nextPollTimeoutMillis != 0) {
              Binder.flushPendingCommands();
          }
          // 阻塞对应时间  
          nativePollOnce(ptr, nextPollTimeoutMillis);
      // 对MessageQueue进行加锁，保证线程安全
          synchronized (this) {
              final long now = SystemClock.uptimeMillis();
              Message prevMsg = null;
              Message msg = mMessages;
              ...
              if (msg != null) {
                  if (now < msg.when) {
                      // 下一个消息还没开始，等待两者的时间差
                      nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                  } else {
                      // 获得消息且现在要执行，标记MessageQueue为非阻塞
                      mBlocked = false;
                      // 链表操作
                      if (prevMsg != null) {
                          prevMsg.next = msg.next;
                      } else {
                          mMessages = msg.next;
                      }
                      msg.next = null;
                      msg.markInUse();
                      return msg;
                  }
              } else {
                  // 没有消息，进入阻塞状态
                  nextPollTimeoutMillis = -1;
              }
             ...
      }
  }
  ```

  代码逻辑很长，其中还涉及了同步屏障和 IdleHandler。next 方法的目的是获取 MessageQueue 中的一个 Message，如果队列中没有消息的话，就会把方法阻塞住，等待新的消息来唤醒，主要步骤如下：

  1、如果 Looper 已经退出了，直接返回 null

  2、进入死循环，知道获取到 Message或者退出

  3、循环中先判断是否需要阻塞，阻塞结束后，对 MessageQueue 进行加锁，获取 Message

  4、如果 MessageQueue 中没有消息，则直接把线程无限阻塞 直到被唤醒

  5、如果 MessageQueue 中有消息，则判断是否需要等待，否则直接返回对应的 Message。

  nextPollTimeoutMillis 表示需要阻塞的时间，-1表示无限时间，只有通过唤醒才能打破阻塞

- 入队 -- enqueueMessage()

```java
MessageQueue.class

boolean enqueueMessage(Message msg, long when) {
    // Hanlder不允许为空
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    // 对MessageQueue进行加锁
    synchronized (this) {
        // 判断目标thread是否已经死亡
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
        // 标记Message正在被执行，以及需要被执行的时间，这里的when是距离1970.1.1的时间
        msg.markInUse();
        msg.when = when;
        // p是MessageQueue的链表头
        Message p = mMessages;
        boolean needWake;
        // 判断是否需要唤醒MessageQueue
        // 如果有新的队头，同时MessageQueue处于阻塞状态则需要唤醒队列
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            ...
            // 根据时间找到插入的位置
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                ...
            }
            msg.next = p; 
            prev.next = msg;
        }

        // 如果需要则唤醒队列
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

这部分代码比较多：主要操作就是链表操作以及判断是否需要唤醒 MessageQueue，下面再总结一下：

- 首先判断 message 的目标 Handler 不能为空且不能正在使用中
- 对 MessageQueue 进行加锁
- 判断目标线程是否已经死亡，死亡则直接返回 false
- 初始化 Message 的执行时间以及标记正在使用中
- 然后根据 Message 的执行时间，找到在链表中的插入位置进行插入
- 同样判断是否需要唤醒MessageQueue。有两种情况需要唤醒：当新插入的Message 在链表头时，如果 MessageQueue 是空的或者正在等待下一个任务的延迟时间执行，这个时候需要唤醒 MessageQueue。

#### MessageQueue 总结

Message 两大重点：阻塞休眠和队列操作。源码中还涉及到了同步屏障和 IdleHandler。

### Looper

##### 概述

Looper 可以说是 Handler 机制中的一个重要的核心。Looper 相当于 线程消息机制的引擎，驱动整个消息机制的运行。Looper 负责从线程中取出消息，然后交给对应的 Handler 去处理。如果队列中没有消息，则 MessageQueue 的 next 方法会阻塞线程，等待消息的到来。每个线程有且只有有一个“引擎”，也就是 Looper，如果没有 Looper，那么消息机制就运行不起来，而如果有多个 Looper，就会违背单线操作的概念，造成并发操作。

每个线程只有一个 Looper，由不同的 Looper 分发的 Message 运行在不同的线程中。Looper 的内部维护者一个 MessageQueue，当初始化 Looper 的时候会顺带着初始化 MessageQueue。

Looper 使用 ThreadLocal 来保证每一个线程只有一个相同的副本

#### 关键方法

- prepare:初始化 Looper

```java
Looper.class

static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

public static void prepare() {
    prepare(true);
}

// 最终调用到了这个方法
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

每个线程使用 Handler 之前，都必须调用Looper.prepare() 方法来初始化当前线程的 Looper。参数 quitAllowed 表示该Looper 是否可以退出。主线程的 Looper 是不能退出的，不然程序就终止了。我们在主线程使用 Handler 是不用初始化的，因为 Activity 在启动的时候就已经帮我们初始化主线程 Looper 了。所以在主线程可以使用 Looper.myLooper() 获得当前线程的 Looper了。

prepare 方法重点在 sThreadLocal.set(new Looper(quitAllowed));可以看出来使用了 ThreadLocal 来创建当前线程的 Looper 对象副本。如果当前线程已经有 Looper了，则会抛出异常，sThreadLocal 是 Looper 的一个静态变量，这里每个线程调用一次prepare 方法就可以初始化当前线程的 Looper了。

接下来看 Looper 的构造方法：

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

逻辑很简单，初始化了一个 MessageQueue,再把当前线程的 Thread 赋值给 mThread。

- myLooper():获取当前线程的 Looper 对象

- loop() :循环获取消息

  当Looper 完成初始化方法之后，他是不会自己启动的，需要我们启动Looper，调用 Looper 的 loop 方法即可：

  ```java
  public static void loop() {
      // 获取当前线程的Looper
      final Looper me = myLooper();
      if (me == null) {
          throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
      }
      final MessageQueue queue = me.mQueue;
      ...
      for (;;) {
          // 获取消息队列中的消息
          Message msg = queue.next(); // might block
          if (msg == null) {
              // 返回null说明MessageQueue退出了
              return;
          }
          ...
          try {
              // 调用Message对应的Handler处理消息
              msg.target.dispatchMessage(msg);
              if (observer != null) {
                  observer.messageDispatched(token, msg);
              }
              dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
          }
          ...
      // 回收Message
          msg.recycleUnchecked();
      }
  }
  ```

Loop()方法是这个Looper 引擎的核心所在。首先获取当前线程的 Looper 对象，没有则抛出异常，然后进入一个死循环；不断调用 MessageQueue 的 next 方法来获取消息，然后调用 message 的Handler 的 disPathMessage 放大来处理 Message。

前面知道：MessageQueue 的 next 方法可能会阻塞：当 MessageQueue 为空或者目前没有任何消息需要处理。所以 Looper 会一直阻塞在这里，线程也不会结束。当我们退出 Looper的时候，next 会返回 null，那么 Looper 也会跟着结束了。

同时，因为 Looper 是运行在不同的线程的逻辑，其调用 的 dispatchMessage 方法也是运行在不同的线程的，就达到了切换线程的目的。

- quit/quitSafely：退出 Looper

  quit 是直接将 Looper 退出，quitSafely 是将 MessageQueue 中的不需要等待的消息处理完成之后再退出，看一下代码：

  ```java
  public void quit() {
      mQueue.quit(false);
  }
  // 最终都是调用到了这个方法
  void quit(boolean safe) {
      // 如果不能退出则抛出异常。这个值在初始化Looper的时候被赋值
      if (!mQuitAllowed) {
          throw new IllegalStateException("Main thread not allowed to quit.");
      }
  
      synchronized (this) {
          // 退出一次之后就无法再次运行了
          if (mQuitting) {
              return;
          }
          mQuitting = true;
      // 执行不同的方法
          if (safe) {
              removeAllFutureMessagesLocked();
          } else {
              removeAllMessagesLocked();
          }
          // 唤醒MessageQueue
          nativeWake(mPtr);
      }
  }
  ```

  最后都调用了quitSafely，这个方法先判断是否能退出，然后再执行退出逻辑。如果 mQuitting 为 true，那么会直接返回，我们回达县 mQuitting 这个变量只有在这里被执行了赋值，所以一旦 Looper 退出，则无法再次执行了。之后执行不同的退出逻辑，removeAllMessagesLocked是直接把所有的 Message 移除，而后者先把执行时间为当前时间或者更早的Message 先执行后再移除剩下需要延迟执行的 Message。之后的 MessageQueue 的 next 方法会退出，Looper() 放大也跟着退出了，那么线程也停止了。

### Looper 总结

Looper 作为Handler 消息机制的 动力引擎，不断的从 MessageQueue 中获取消息，然后交给 Handler 去执行，Looper 使用前需要初始化该线程的 Looper 对象，再调用 Looper 方法启动他。

同时 Handler 也是实现切换的核心，因为不同的 Looper 是运行在不同的线程，他所调用的 dispatchMessage 方法也是运行在不同的线程，所以 Message 的处理就被切换到 Looper 所在的线程了。当 Looper不再使用的时候，可调用不同的放大来退出，注意 Looper一旦退出，**线程**直接结束。













