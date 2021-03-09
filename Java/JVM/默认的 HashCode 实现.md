# Java 默认的 hashcode 方法到底得到的是什么？

hashcode 方法影响 jvm 性能？听上去是天方夜谭，实际上蕴含着一些微小的原理，接下来让我们走进 hashcode方法，一探native 方法源头。

#### 默认实现是什么？

调用 hashcode 方法默认返回的值被称为 identity hash code（标识哈希码）接下来我们会用标识哈希码来区分重写 hashcode 方法。如果一个类重写了 hashcode 方法，可以通过 System.identityhashcode 方法获取的标识哈希吗。

在 hashcode 的方法注释中，说 hashcode 一般是通过对象内存地址映射过来的。

但是了解 JVM 的同学肯定知道，不管是标记复制算法还是标记整理算法，都会改变对象的内存地址。鉴于 JVM 重定位对象地址，但该 hashcode 又不能变化。

先看一下源码：

```java
public native int hashCode();
```

hashcode 是一个本地方法。

#### 真正的 hashcode 方法

hashcode 方法的实现依赖于 jvm，不同的 jvm 有不同的是实现，我们目前可以看到的 jvm 源码就是 OpenJDK 的源码 OpenJDK 的源码。OpenJDK 的源码大部分和 Oracle 的 JVM 源码一致。

#### 真正的 identity hash code 生成

生成 hash 的最终函数 get_next_hash，这个函数提供了六种生成 hash 值的方法。

```xml
0. A randomly generated number.
1. A function of memory address of the object.
2. A hardcoded 1 (used for sensitivity testing.)
3. A sequence.
4. The memory address of the object, cast to int.
5. Thread state combined with xorshift (https://en.wikipedia.org/wiki/Xorshift)
```

OpenJDK 8默认使用的是第五种算法，6 7使用的是第一种算法。

#### 对象头格式

对上一节，我们知道了 hash 值是放在对象头里面的，那就了解以下 对象头的结构吧。

```
30 // The markOop describes the header of an object.
31 //
32 // Note that the mark is not a real oop but just a word.
33 // It is placed in the oop hierarchy for historical reasons.
34 //
35 // Bit-format of an object header (most significant first, big endian layout below):
36 //
37 //  32 bits:
38 //  --------
39 //             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
40 //             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
41 //             size:32 ------------------------------------------>| (CMS free block)
42 //             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
43 //
44 //  64 bits:
45 //  --------
46 //  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
47 //  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
48 //  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
49 //  size:64 ----------------------------------------------------->| (CMS free block)
50 //
51 //  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
52 //  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
53 //  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
54 //  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block
```

它的格式在32位和64位上略有不同，64位有两种变体，具体取决于是否启用了压缩对象指针。

#### 对象头中偏向锁和 Hashcode 的冲突

在上一节我们看到，normal object 和 biased object 分别存放的是 hashcode 和线程的id。因此也就是说如果调用了本地方法 hashcode，就会占用偏向锁对象使用的位置，偏向锁将会失效，晋升为轻量级锁。

这个过程看下这个图。

![](/picture/642.png)

这里来简单的解答以下，首先在 JVM 启动的时候，可以使用 UseBiasedLocking 参数开启偏向锁。

接下来，如果偏向锁可用，那分配到的对象Mark Word 格式为包含线程 ID，当未标记的时候，线程 ID 为0，第一次获取锁时，线程会把自己的线程 ID 写到 ThreadID 字段，这样，下一次获取锁的时候直接检查标记字中的线程 ID 和自身 ID是否相同，如果一直就认为获取了锁，因此不需要再次获取锁。

假设这个时候有别的线程需要竞争锁了，此时线程会通知持有偏向锁的线程持有锁，假设持有偏向锁的线程已经销毁，则将对象头设为无锁状态，如果线程或者，则尝试切换，如果不成功，那么锁会升级为轻量级锁。

这时有个问题来了，如果需要获取对象的 identity hashcode，偏向锁就会被禁用，然后给原先设置线程 ID 的地方写入了 hash 值。

如果 hash 有值，或者偏向锁无法撤销，则会进入轻量级锁，轻量级锁竞争时，每个线程会先将 hashcode 值保存到自己的栈内存中，然后通过 CAS 尝试将自己新建的记录空间地址写入到对象头中，谁先写入成功谁拥有了对象。

轻量级锁竞争失败的线程会自旋尝试获得锁一段时间，一段时间还是获取不到，则升级为重量级锁，没获取锁的线程会被阻塞。







































