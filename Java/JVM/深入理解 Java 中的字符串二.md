# 深入理解 Java 中的字符串二

这一篇来看一下 StringBuilder 和 StringBuffer。这两个类和 String 有什么关系呢，看一下：

![](/picture/data-26.image)

从图中可以看出StringBuilder和StringBuffer都继承了AbstractStringBuilder，而AbstractStringBuilder与String实现了共同的接口CharSequence。

我们知道，字符串是由一系列字符组成的，String的内部就是基于char数组（jdk9之后基于byte数组）实现的，而数组通常是一块连续的内存区域，在数组初始化的时候就需要指定数组的大小。上一篇文章中我们已经知道String是不可变的，因为它内部的数组被声明为了final，同时，String的字符拼接、插入、删除等操作均是通过实例化新的对象实现的。而今天要认识的StringBuilder和StringBuffer与String相比就具有了动态性。接下来就让我们一起来认识下这两个类。

## 一、StringBuilder

在StringBuilder的父类AbstractStringBuilder 中可以看到如下代码：

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    /**
     * The value is used for character storage.
     */
    char[] value;

    /**
     * The count is the number of characters used.
     */
    int count;
}
```

StringBuilder与String一样都是基于char数组实现的，不同的是StringBuilder没有final修饰，这就意味着StringBuilder是可以被动态改变的。接下来看下StringBuilder无参构造方法，代码如下：

```java
 /**
     * Constructs a string builder with no characters in it and an
     * initial capacity of 16 characters.
     */
    public StringBuilder() {
        super(16);
    }
复制代码
```

在这个方法中调用了父类的构造方法，到AbstractStringBuilder 中看到其构造方法如下：

```java
    /**
     * Creates an AbstractStringBuilder of the specified capacity.
     */
    AbstractStringBuilder(int capacity) {
        value = new char[capacity];
    }
复制代码
```

AbstractStringBuilder的构造方法内部初始化了一个容量为capacity的数组。也就是说StringBuilder默认初始化了一个容量为16的char[]数组。StringBuilder中除了无参构造外还提供了多个构造方法，源码如下：

```java
 /**
     * Constructs a string builder with no characters in it and an
     * initial capacity specified by the {@code capacity} argument.
     *
     * @param      capacity  the initial capacity.
     * @throws     NegativeArraySizeException  if the {@code capacity}
     *               argument is less than {@code 0}.
     */
    public StringBuilder(int capacity) {
        super(capacity);
    }

    /**
     * Constructs a string builder initialized to the contents of the
     * specified string. The initial capacity of the string builder is
     * {@code 16} plus the length of the string argument.
     *
     * @param   str   the initial contents of the buffer.
     */
    public StringBuilder(String str) {
        super(str.length() + 16);
        append(str);
    }

    /**
     * Constructs a string builder that contains the same characters
     * as the specified {@code CharSequence}. The initial capacity of
     * the string builder is {@code 16} plus the length of the
     * {@code CharSequence} argument.
     *
     * @param      seq   the sequence to copy.
     */
    public StringBuilder(CharSequence seq) {
        this(seq.length() + 16);
        append(seq);
    }
复制代码
```

这段代码的第一个方法初始化一个指定容量大小的StringBuilder。另外两个构造方法分别可以传入String和CharSequence来初始化StringBuilder，这两个构造方法的容量均会在传入字符串长度的基础上在加上16。

### 1.StringBuilder的append操作与扩容

上篇文章已经知道通过StringBuilder的append方法可以进行高效的字符串拼接，append方法是如何实现的呢？这里以append（String）为例，可以看到StringBuilder的append调用了父类的append方法，其实不止append，StringBuilder类中操作字符串的方法几乎都是通过父类来实现的。append方法源码如下：

```java
	// StringBuilder
 	@Override
    public StringBuilder append(String str) {
        super.append(str);
        return this;
    }
    
  // AbstractStringBuilder
  public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);
        count += len;
        return this;
    }
```

在append方法的第一行首先进行了null检查，等于null的时候调用了appendNull方法。其源码如下：

```java
private AbstractStringBuilder appendNull() {
        int c = count;
        ensureCapacityInternal(c + 4);
        final char[] value = this.value;
        value[c++] = 'n';
        value[c++] = 'u';
        value[c++] = 'l';
        value[c++] = 'l';
        count = c;
        return this;
    }
```

appendNull方法中首先调用了ensureCapacityInternal来确保字符串数组容量充值，关于ensureCapacityInternal这个方法下边再详细分析。接下来可以看到把"null"的字符添加到了char[]数组value中。

上文我们提到，StringBuilder内部数组的默认容量是16，因此，在进行字符串拼接的时候需要先确保char[]数组有足够的容量。因此，在appendNull方法以及append方法中都调用了ensureCapacityInternal方法来检查char[]数组是否有足够的容量，如果容量不足则会对数组进行扩容，ensureCapacityInternal源码如下：

```java
private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        if (minimumCapacity - value.length > 0)
            expandCapacity(minimumCapacity);
    }
```

这里判读如果拼接后的字符串长度大于字符串数组的长度则会调用expandCapacity进行扩容。

```java
void expandCapacity(int minimumCapacity) {
        int newCapacity = value.length * 2 + 2;
        if (newCapacity - minimumCapacity < 0)
            newCapacity = minimumCapacity;
        if (newCapacity < 0) {
            if (minimumCapacity < 0) // overflow
                throw new OutOfMemoryError();
            newCapacity = Integer.MAX_VALUE;
        }
        value = Arrays.copyOf(value, newCapacity);
    }
```

expandCapacity的逻辑也很简单，首先通过原数组的长度乘2并加2后计算得到扩容后的数组长度。接下来判断了newCapacity如果小于minimumCapacity，则将minimumCapacity值赋值给了newCapacity。这里因为调用expandCapacity方法的不止一个地方，所以加这句代码确保安全。

而接下来的一句代码就很有趣了，newCapacity 和minimumCapacity 还有可能小于0吗？当minimumCapacity小于0的时候竟然还抛出了一个OutOfMemoryError异常。其实，这里小于0是因为越界了。我们知道在计算机中存储的都是二进制，乘2相当于向左移了一位。以byte为例，一个byte有8bit,在有符号数中最左边的一个bit位是符号位，正数的符号位为0，负数为1。那么一个byte可以表示的大小范围为[-128~127]，而如果一个数字大于127时则会出现越界，即最左边的符号位会被左边第二位的1顶替，就出现了负数的情况。当然，并不是byte而是int,但是原理是一样的。

另外在这个方法的最后一句通过Arrays.copyOf进行了一个数组拷贝，其实Arrays.copyOf在上篇文章中就有见到过，在这里不妨来分析一下这个方法，看源码：

```java
 public static char[] copyOf(char[] original, int newLength) {
        char[] copy = new char[newLength];
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```

咦？copyOf方法中竟然也去实例化了一个对象！！那不会影响性能吗？莫慌，看一下这里仅仅是实例化了一个newLength长度的空数组，对于数组的初始化其实仅仅是指针的移动而已，浪费的性能可谓微乎其微。接着这里通过System.arraycopy的native方法将原数组复制到了新的数组中。

### 2.StringBuilder的subString()方法toString()方法

StringBuilder中其实没有subString方法，subString的实现是在StringBuilder的父类AbstractStringBuilder中的。它的代码非常简单，源码如下：

```java
public String substring(int start, int end) {
        if (start < 0)
            throw new StringIndexOutOfBoundsException(start);
        if (end > count)
            throw new StringIndexOutOfBoundsException(end);
        if (start > end)
            throw new StringIndexOutOfBoundsException(end - start);
        return new String(value, start, end - start);
    }
复制代码
```

在进行了合法判断之后，substring直接实例化了一个String对象并返回。这里和String的subString实现其实并没有多大差别。 而StringBuilder的toString方法的实现其实更简单，源码如下：

```java
 @Override
    public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
复制代码
```

这里直接实例化了一个String对象并将StringBuilder中的value传入，我们来看下String(value, 0, count)这个构造方法：

```java
 	public String(char value[], int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > value.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        this.value = Arrays.copyOfRange(value, offset, offset+count);
    }
复制代码
```

可以看到，在String的这个构造方法中又通过Arrays.copyOfRange方法进行了数组拷贝，Arrays.copyOfRange的源码如下：

```java
   public static char[] copyOfRange(char[] original, int from, int to) {
        int newLength = to - from;
        if (newLength < 0)
            throw new IllegalArgumentException(from + " > " + to);
        char[] copy = new char[newLength];
        System.arraycopy(original, from, copy, 0,
                         Math.min(original.length - from, newLength));
        return copy;
    }
复制代码
```

Arrays.copyOfRange与Arrays.copyOf类似，内部都是重新实例化了一个char[]数组，所以String构造方法中的this.value与传入进来的value不是同一个对象。**意味着StringBuilder在每次调用toString的时候生成的String对象内部的char[]数组并不是同一个！这里立一个Falg**！

### 3.StringBuilder的其它方法

StringBuilder除了提供了append方法、subString方法以及toString方法外还提供了还提供了插入（insert）、删除（delete、deleteCharAt）、替换（replace）、查找（indexOf）以及反转（reverse）等一些列的字符串操作的方法。但由于实现都非常简单，这里就不再赘述了。

## 二、StringBuffer

在第一节已经知道，StringBuilder的方法几乎都是在它的父类AbstractStringBuilder中实现的。而StringBuffer同样继承了AbstractStringBuilder，这就意味着StringBuffer的功能其实跟StringBuilder并无太大差别。我们通过StringBuffer几个方法来看

```java
	 /**
     * A cache of the last value returned by toString. Cleared
     * whenever the StringBuffer is modified.
     */
    private transient char[] toStringCache;

  	@Override
    public synchronized StringBuffer append(String str) {
        toStringCache = null;
        super.append(str);
        return this;
    }

    /**
     * @throws StringIndexOutOfBoundsException {@inheritDoc}
     * @since      1.2
     */
    @Override
    public synchronized StringBuffer delete(int start, int end) {
        toStringCache = null;
        super.delete(start, end);
        return this;
    }

  /**
     * @throws StringIndexOutOfBoundsException {@inheritDoc}
     * @since      1.2
     */
    @Override
    public synchronized StringBuffer insert(int index, char[] str, int offset,
                                            int len)
    {
        toStringCache = null;
        super.insert(index, str, offset, len);
        return this;
    }

@Override
    public synchronized String substring(int start) {
        return substring(start, count);
    }
    
// ...
```

可以看到在StringBuffer的方法上都加上了synchronized关键字,也就是说StringBuffer的所有操作都是线程安全的。所以，在多线程操作字符串的情况下应该首选StringBuffer。 另外，我们注意到在StringBuffer的方法中比StringBuilder多了一个toStringCache的成员变量 ，从源码中看到toStringCache是一个char[]数组。它的注释是这样描述的：

> toString返回的最后一个值的缓存，当StringBuffer被修改的时候该值都会被清除。

我们再观察一下StringBuffer中的方法，发现只要是操作过操作过StringBuffer中char[]数组的方法，toStringCache都被置空了！而没有操作过字符数组的方法则没有对其做置空操作。另外，注释中还提到了 toString方法，那我们不妨来看一看StringBuffer中的 toString，源码如下：

```java
   @Override
    public synchronized String toString() {
        if (toStringCache == null) {
            toStringCache = Arrays.copyOfRange(value, 0, count);
        }
        return new String(toStringCache, true);
    }
```

这个方法中首先判断当toStringCache 为null时会通过 Arrays.copyOfRange方法对其进行赋值，Arrays.copyOfRange方法上边已经分析过了，他会重新实例化一个char[]数组，并将原数组赋值到新数组中。这样做有什么影响呢？细细思考一下不难发现在不修改StringBuffer的前提下，多次调用StringBuffer的toString方法，生成的String对象都共用了同一个字符数组--toStringCache。这里是StringBuffer和StringBuilder的一点区别。至于StringBuffer中为什么这么做其实并没有很明确的原因，可以参考StackOverRun [《Why StringBuffer has a toStringCache while StringBuilder not?》](https://stackoverrun.com/cn/q/12697236)中的一个回答:

> 1.因为StringBuffer已经保证了线程安全，所以更容易实现缓存（StringBuilder线程不安全的情况下需要不断同步toStringCache） 2.可能是历史原因





























































