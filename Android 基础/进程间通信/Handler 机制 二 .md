# Android 全面解析之 Handler 机制 二：ThreadLocal

### 前言

本文内容主要介绍的是 **Handler 的内部模式结构和 ThreadLocal 详解**。

### 正文

## Handler 内部模式结构

下图为 Handler 机制的内部模式图：

![](/picture/data-03.image)

Handler 机制内部有三大关键角色：Handler、Looper、MessageQueue。其中 MessageQueue 是 Looper 内部的一个对象，MessageQueue 和 Looper 每一个线程只有一个，而 Handler 是可以有很多歌的。他们的工作流程是：

- 用户使用线程的 Looper 构造 handler  之后，通过 Handler 的 send 和 post 方法发送消息
- 消息会加入到 MessageQueue 中，等待 Looper 获取
- Looper 会不断的从 MessageQueue 中获取 Message 然后交付给 对应的 Handler 处理

## ThreadLocal

### 概述

> ThreadLocal 是 Java 中用于存储线程内部数据的工具类

ThreadLocal 是用来存储数据的，但是每个线程只能访问到各自线程的数据。我们一般的用法是：

```java
ThreadLocal<String> stringLocal = new ThreadLocal<>();
stringLocal.set("java");
String s = stringLocal.get();
```

不同的线程之间访问到的数据是不一样的：

```java
public static void main(String[] args){
    ThreadLocal<String> stringLocal = new ThreadLocal<>();
 stringLocal.set("java");
    
    System.out.println(stringLocal.get());
    new Thread(){
        System.out.println(stringLocal.get());
    }
}

结果：
java
null
```

线程只能访问到自己线程存储的数据。

#### Thread Local 的作用

threadLocal 的特性适用于**同样的数据类型，不同的线程有不同的备份**情况。如我们一直讲的 Looper。每个线程都有一个对象，但是不同的线程的 Looper 对象是不同的，这个时候就特别适合使用 ThreadLocal 来存储数据，这也是为什么要讲 ThreadLocal 的原因。

#### ThreadLocal 内部结构

ThreadLocal 的内部机制结构如下：

![](/picture/data-04.image)

每个线程，也就是每个线程内部维护着一个 ThreadLocalMap，ThreadLocalMap 内部存储着多个 Entry。Entry 可以理解为键值对，它的本质是一个弱引用，内部有一个 object 类型的内部变量，如下：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k); //由于Entry 继承了 WeakReference,所以这里以一个弱引用指向 ThreadLocal 对象
        value = v;
    }
}
```

这里为什么需要使用弱引用呢？看下面这种场景：

```java
public void func1() {
        ThreadLocal tl = new ThreadLocal<Integer>(); //line1
         tl.set(100);   //line2
         tl.get();       //line3
}
```

line1 创建了一个 ThreadLocal 对象，t1 是指向这个对象的强引用；line2 调用 set方法之后，就会新建一个 Entry,通过源码可知 Entry 对象的 K 是指向 ThreadLocal 的弱引用，如图：

![](/picture/01.PNG)

当 func 方法执行完毕，栈帧被销毁，强引用 t1 也没有了，但是此时线程的 ThreadLocalMap 里面的某个 Entry 的k引用还是指向这个对象，若这个k是强引用，就会导致 k 指向的 ThreadLocal 对象及V 指向的对象不能被 GC 回收，造成内存泄漏，但是弱引用就不会有这种问题，而且在entry的 k 引用为 null 之后，再调用 get，set， remove 方法的时候，会尝试删除 key 为null 的entry,可以释放 value 对象占用的内存。

虽然弱引用，保证了 K 指向的 ThreadLocal 对象可以及时被 gc 回收，但是 v 指向的 value 对象是需要ThreadLocalMap 调用 get set时发现k 为null 时才会回收整个entry、value，因此弱引用不能完全保证内存完全不泄露。**我们要在不使用某个ThreadLocal对象后，手动调用remove方法来删除它，尤其是在线程池中，不仅仅是内存泄露的问题，因为线程池中的线程是重复使用的，意味着这个线程的ThreadLocalMap对象也是重复使用的，如果我们不手动调用remove方法，那么后面的线程就有可能获取到上个线程遗留下来的value值，造成bug**。

​       web容器使用了线程池，当一个请求使用完某个线程，该线程会放回线程池被其它请求使用，这就导致一个问题，不同的请求还是有可能会使用到同一个线程（只要请求数量大于线程数量），而ThreadLocal是属于线程的，如果我们使用完ThreadLocal对象而没有手动删掉，那么后面的请求就有机会使用到被使用过的ThreadLocal对象，如果一个请求在使用ThreadLocal的时候，是先get()来判断然后再set()，那就会有问题，因为get到的是别的请求set的内容，如果一个请求每次使用ThreadLocal，都是先set再get，那就不会有问题，因为一个线程同一时刻只被一个请求使用，只要我们每次使用之前，都设置成自己想要的内容，那就不会在使用的过程中被覆盖。使用ThreadLocal最好是每次使用完就调用remove方法，将其删掉，避免先get后set的情况导致业务的错误。

Entry 是 ThreadLocalMap 的一个静态类，这样每个 Entry里面就维护了一个 ThreadLocal 和 ThreadLocal 泛型对象。每个线程的内部维护着一个 Entry 数组，并通过 hash 算法使得读取数据的速度可以达到 O(1)。由于不同的线程对应的 Thread 对象不同，所以对应的 ThreadLocalMap 肯定也不同。这样只有获取到 Thread  对象才能获取到内部数据，数据就被隔离再不同的线程内部。

#### ThreadLocal 工作流程

那ThreadLocal 是怎么实现把数据存储在不同的线程呢？先从set 方法入手：

```java
TheadLocal.class
    
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

逻辑不是很复杂，首先获取当前线程的 Thread 对象，然后再获取 Thread 的 ThreadLocalMap 对象，如果该 map 对象不存在则创建一个并调用它的 set 方法把数据存起来。我们继续看 ThreadLocalMap 的 set 方法：

```java
ThreadLocalMap.class

private void set(ThreadLocal<?> key, Object value) {
    // 每个ThreadLocalMap内部都有一个Entry数组
    Entry[] tab = table;
    int len = tab.length;
    // 获取新的ThreadLocal在Entry数组中的下标
    int i = key.threadLocalHashCode & (len-1);
    // 判断当前位置是否发生了Hash冲突
    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        // 如果数据存在且相同则直接返回
        if (k == key) {
            e.value = value;
            return;
        }
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 若当前位置没有其他元素则直接把新的Entry对象放入
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 判断是否需要对数组进行扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

这里的逻辑和 HashMap 是很像的，我们可以直接使用 HashMap 的思维来理解 ThreadLocalMap：ThreadLocalMap 的 key 是 ThreadLocal,value 是 ThreadLocal 对应的泛型。他的存储步骤如下：

- 根据自身的 threadLocalHashMap 与数组的长度进行相与得到下标
- 如果此下标为空，则直接插入
- 如果此下标已经有元素，则判断两者的 ThreadLocal 是否相同，相同则更新value 返回，否则找下一个下标
- 直到找到合适的位置把 entry 对象插入
- 最后判断是否需要对 entry 数组进行扩容

是不是和 HashMap 非常像？和 HashMap 的不同的时：hash 算法不一样，以及这里使用的时开放地址法，而HashMap 使用的是链地址法。ThreadLocal 牺牲了一定的空间来换取更快的速度。

然后继续看 ThreadLocal  的 get 方法：

```java
ThreadLocal.class

public T get() {
    // 获取当前线程的ThreadLocalMap
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 根据ThreadLocal获取Entry对象
        ThreadLocalMap.Entry e = map.getEntry(this);
        // 如果没找到也会执行初始化工作
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 把获取到的对象进行返回
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

前面讲到 ThreadLocalMap  其实就像是一个 HashMap,它的 get 方法也是一样的，使用 ThreadLocal 作为 key 获取到对应的 entry，再把 value 返回即可。如果 Map 尚未初始化即会执行初始化操作。下面继续看下 ThreadLocalMap 的 get 方法：

```java
ThreadLocalMap.class

private Entry getEntry(ThreadLocal<?> key) {
    // 根据hash算法找到下标
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 找到数据则返回，否则通过开发地址法寻找下一个下标
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

### 内存泄露问题

我们会发现 Entry 中，ThreadLocal 是一个 弱引用，而 Value 却是一个强引用。如果外部没有对 ThreadLocal 的任何引用，那么 ThreadLocal 就会被回收，此时其对应的 value 也就变的没有意义了，但是无法被回收，这就造成了内存泄露，怎么解决？再 ThreadLocal  回收的时候记得调用其 remove 方法把 entry 移除，防止内存泄露。

### ThreadLocal 总结

ThreadLocal 适用于在不同线程作用域的数据备份

ThreadLocal 机制通过在每一个线程维护一个 ThreadLocalMap，其key 为 ThreadLocal,value 为 ThreadLocal 对应的泛型对象，这样每个 ThreadLocal 就可以作为 key 将不同的 value 存储到不同的 Thread 的 Map 中，当获取数据的时候，同一个 ThreadLocal 就可以在不同的线程获取到不同的数据，如下图：

![](/picture/data-05.image)

ThreadLocalMap 类似于一个改版的 HashMap，内部也是使用数组和 hash 算法来存储数据，使得存储和读取的速度非常快。

同时使用ThreadLocal需要注意内存泄露问题，当ThreadLocal不再使用的时候，需要通过remove方法把value移除。

























