# 深入 JVM -- 探索 Java 虚拟机的类加载机制

我们知道 Java 程序在编译的过程中需要先经过 javac 将 Java 文件编译成字节码文件才能被虚拟机执行；而类加载指的是将编译好的字节码（不仅仅指.class 文件中的字节码，任意的字节码流都可以被读取到 JVM）读取到JVM 的内存中的过程。虚拟机在加载 .class 文件时会对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型。这个过程称作虚拟机的类加载机制。类加载机制是虚拟机中很重要的一部分内容，在面试中出现的频率很高。因此，我们有必要进行学习。

为了更好的理解类加载的过程，我们先看一道类加载的面试题：

```java
// Person.java
public class Person{
    static{
        System.out.println("I'm a person");
    }
}

// Stuent.java
public class Student extends Person{
    public static String indentity="Student";
    static{
        System.out.println("I'm a student");
    }
}

public class Ryan extends Student{
    static{
        System.out.println("I'm Ryan");
    }
}
```

接下来写一个测试类：

```java
public class Test {
	public static void main(String[] args) {	
		System.out.println("Ryan.indentity=" + Ryan.indentity);
	}	
}
```

看一下输出：

```
I'm a person
I'm a student
Ryan.indentity=Student
```

> ### 一、类加载的过程

一个类从被加载到虚拟机内存中开始，到卸载出虚拟机内存为止，它的生命周期会经历加载（loading）、连接（Linking）、初始化（Initialization）、使用（Using）和卸载（Unloading）这几个阶段。而连接阶段又包括验证（Verification）、准备（Preparation）、解析（Resolution）三个阶段。如下图所示:

![](/picture/data-03.image)

其中，加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始。接下来，我们来详细的了解 Java 虚拟机中类加载的过程，即加载、验证、准备、解析和初始化这五个阶段。

##### 1、加载阶段

加载阶段是类加载过程的第一个阶段。这一阶段 JVM 会通过类的全限定名（可以来自.class 文件，也可能来自 ZIP 压缩包、网络等，甚至可以是运行时生成）来读取类的二进制字节流。并将读取到的二进制字节流转化为方法区的运行时数据结构，然后在内存中生成一个代表这个类的 java.lang.Class 对象。

简单的来说，这一阶段就是将类的字节码二进制流读取到 JVM 中，并生成代表这个类的 Class 对象。

##### 2、连接

连接阶段包含了验证、准备、解析三个过程。

##### 1）验证

这一阶段的目的是为了保证 Class 文件字节流符合当前虚拟机的要求。并且保证这些数据代码运行后不会危害虚拟机自身安全。验证阶段大致会完成四个阶段的检验：文件格式验证、元数据验证、字节码验证、符号引用验证。

###### 文件格式验证

这一阶段验证字节流是否符合 Class文件格式的规范，并且能被当前的虚拟机处理

###### 元数据验证

这一阶段是对字节码描述的信息进行语义分析，保证其描述的信息符合 Java 语言规范的要求

###### 字节码验证

这一阶段会通过数据流和控制流分析，确定程序语义是否合法，符合逻辑。这个阶段对类的方法体进行校验分析，保证被校验的类的方法在运行时不会做危害虚拟机安全的事情。

###### 符号引用验证

最后一个阶段的校验发生在虚拟机符号引用转换为值引用的时候，这个转化动作将在连接的第三阶段--解析阶段发生。符号引用校验可以看做是对类自身以外（常量池的各种符号引用）的信息进行校验

##### 2）准备

**准备阶段是类加载机制中一个很重要的一个阶段**。这一阶段是正式为类中定义的静态变量（被 static 修饰的变量）分配内存并设置类变量初始值的阶段。这些变量所使用的的内存都应该在方法区中进行分配。我们知道，在 JDK1.7 之前，HotSpot 虚拟机使用永生代带实现方法区。而在 JDK8 之后，方法区被放在 Java 堆中。因此，类变量也会随着 Class 对象放在 Java 堆中。

另外，关于准备阶段有两点需要注意：

**为变量分配内存**我们知道，Java 类中的变量可以分为 成员变量和类变量，类变量是指被 static 修饰的变量，其他类型的变量都属于成员变量。而准备阶段的内存分配仅包括类变量，不包括成员变量，成员变量只有在对象实例化的时候随着对象一起分配到 Java 堆中。

例如下面的代码在准备阶段只会为 Value 分配内存，不会为 str 分配内存。

```java
	public class Test {
		  public static int value = 123;
		  public  String str = "123";
	}
```

**为类变量赋初始值** 在准备阶段，JVM 会为类变量分配内存，并对其初始化。**而初始化的值并非我们在代码中赋予的值，而是数据类型的零值。**例如上述代码中经过准备阶段后 value 的值是0，而非123。但如果给 value 再加一个 final 修饰符，那么经过准备阶段，value 的值就是 123（因为此时的 value 相当于一个常量），这是因为再编译时 Javac 会为 value 生成 ConstantsValue 属性，在准备阶段虚拟机就会根据 ConstantValue 的设置将 value 赋值为 123.

##### 3）解析

解析阶段是虚拟机将常量池的符号引用替换为直接引用的过程。这一阶段不太重要，了解即可。

#### 3、初始化

初始化是类加载的最后一个阶段，也是类加载过程中最重要的一个阶段。在这一阶段用户定义的 Java 程序代码（字节码）才真正开始执行。什么意思呢？刚才提到在准备阶段 JVM 会为类变量赋值默认的初始值。**而初始化阶段类变量才会被赋予我们在代码中声明的值。JVM 会根据语句执行顺序对类对象进行初始化。**

在 Java 虚拟机规范中并没有强制约束在什么情况下开始执行类加载的第一个“加载”阶段，但是对于初始化阶段却有着严格的约束。一般来说当 JVM 遇到下面 6 种情况的时候会触发类加载的初始化（在执行初始化阶段之前需要先执行加载、验证和准备阶段）：

- 遇到 new、getStatic、putStatic、invokeStatic 这四条字节码指令的时候，如果类没有初始化，则需要先进行初始化。生成这四条指令的最常见的 Java  代码的场景是：使用 new 关键字实例化对象的时候、读取或者设置一个类的静态字段（被 final 修饰、已在编译器把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。
- 使用 Java.lang.reflect 包的方法对类进行反射调用的时候，如果类没有进行初始化，则需要先出发其初始化
- 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先出发其父类的初始化
- 当虚拟机启动时，用户需要指定一个要执行的主类（包含main() 方法的那个类），虚拟机会先初始化这个主类。
- 当使用 JDK 1.7 动态语言支持时，如果一个 java.lang.invoke.MethodHandle 实例最后的解析结果 REF_getstatic，REF_putstatic，REF_invokestatic 的方法句柄，并且这个方法句柄所对应的类没有进行初始化，则需要先触发其初始化。
- 当一个接口中定义了 JDK8 新加入的默认方法（被 default 关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

> ### 二、类加载例题分析

在了解类加载的过程之后，通过几个例题分析来深入的理解类加载。

##### 1、开篇例题分析

开篇的面试题中在 main 方法中调用了 Ryan.indentity，而 indentity 是位于 Ryan 父类 Student 中的类变量。根据初始化阶段中的第1点我们知道：

> 读取或设置一个类的静态字段（被 final 修饰、已在编译器把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候会先触发类的初始化。

因此，此时会首先去加载并初始化 Student 类（因为 identity 是位于 Student 类中的静态变量，因此 Ryan 类不会被加载），从而初始化 3中可以知道：

> 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先出发其父类的初始化

因此，最先被加载并初始化的类应该是 Person 类，Person 类的初始化导致了首先输出 I‘m a person 语句，接着 Student 类被加载，所以第二行输出了 I'm a student 。在完成上述类加载后输出 Ryan.indentity = Student

##### 2、例题二

给出 Singleton 类如下代码所示，请分析程序的输出结果。

```java
public class Singleton {
    private static Singleton singleton = new Singleton();
    public static int x;
    public static int y = 0;

    private Singleton() {
        ++x;
        ++y;
        System.out.println("Singleton构造方法执行，x = " + x +",y = " + y);
    }

    public static void main(String[] args) {
        System.out.println("singleton.x = " + singleton.x);
        System.out.println("singleton.x = " + singleton.y);
    }
}
```

输出结果：

```java
Singleton构造方法执行，x = 1,y = 1
singleton.x = 1
singleton.x = 0
```

如果不了解类加载的过程，会觉得这是一个很奇怪的输出结果。x y的初始值都是0，在构造方法中经过了同样的 ++ 操作，而最后的输出结果为什么不一样呢？我们还是先从类初始化的几个条件入手。

- 从触发类初始化的条件4可知，虚拟机启动时会先加载包含 main 方法的类。因此，SingleTon 首先会出发类加载流程。

- 而经过加载、验证流程，进入类加载的准备阶段，这一阶段虚拟机会为类变量分配内存和并对其进行初始化赋值。注意，准备阶段只会给类变量赋默认值，经过准备阶段后结果如下：

  ```java
  public class Singleton {
      private static Singleton singleton = null;
      public static int x = 0;
      public static int y = 0;
  }
  ```

- 初始化阶段会根据代码顺序为类变量赋代码中声明的值。因此，首先会实例化 Singleton，并将实例化后的值赋给 Singleton。而此时，由于 x、y还没有被赋值。因此 x、y 还没有被赋值。因此 x、y 均为0.所以，在经过 ++ 操作后输出 x、y 的值均为1

- 接下来为 x、y 赋代码中声明的值，而我们的代码中 x 没有赋初始值，y 则被赋值为0。因此，此时 x 仍然为1，而 y 则被赋值为0.

- 类加载完成后打印 x、y 的值

经过以上两个例题的分析，相信大家对 JVM 的类加载机制有了一个更清楚的认识。而在类加载机制中除了类加载的过程，还有一个很重要的知识点，那就是类加载器，我们接着往下看。

> ### 三、类加载器

类加载的过程是由类加载器来完成的。类加载器在 Java 程序中起到的作用可以说是远超类加载阶段。我们在程序中使用到的任意一个类都需要类加载器将其加载到虚拟机，并且由类加载器保证被加载类的唯一性。我们这里需要明白一点：**两个类是否相等的前提条件是这两个类由一个类加载器加载的。如果两个类来自同一个 Class 文件，但是被同一个虚拟机下面的两个类加载器加载，那么这两个类也是不相等**。

那么问题来了，虚拟机是如何保证同一个 Class文件只能被同一个类加载器加载呢？要解答这个问题首先要了解类加载的区分。

#### 1、类加载器的分类

在 Java 中类加载器分为启动类加载器（Bootstrap Class Loader)、扩展类加载器（Extension Class Loader)、应用类加载器（Application Class Loader） 以及自定义类加载器（User Class Loader)。接下来我们就分别来认识这几种类加载器。

##### 1）启动类加载器（Bootstrap  Class Loader)

这个类加载器是虚拟机的一部分，使用 C++ 语言实现。这个类加载器只负责加载存放在 <JAVA_HOME>\lib目录中，或者被 -Xbootclasspath 参数所指定的路径中存放的 Java虚拟机能够识别的（按照文件名识别，如 rt.jar、tool.jar。名字不符合的类库即使放在lib目录下也不会被加载）类库加载到虚拟机中。

2）扩展类加载器（Extention Class Loader)

这个类加载器位于类 sun.miss.Launcher$ExtClassLoader 中，并且是由 Java 代码所实现的。他负责加载<Java_HOME>\lib\ext目录中，或被 java.ext.dirs 系统变量所指定的路径中所有的类库。开发者可以直接在程序中使用扩展类加载器来加载 Class 文件。

3）应用类加载器 （Application Class Loader）

这个类加载器位于 sun.misc.Launcher$AppClassLoader 中，同样是由 Java 语言实现。他负责加载用户类路径（ClassPath）上所有的类库。开发者同样可以直接在代码中使用这个类加载器。如果程序中没有自定义的类加载器，一般情况下这个就是程序中默认的类加载器。

4）自定义类加载器（User Class Loader）

除了上述三种 Java 系统中的类加载器外，很多情况下用户还会通过自定义类加载器加载所需要的类。诸如增加除了磁盘之外的 Class 文件来源，或者通过类加载器实现类的隔离、重载等功能。

### 2、双亲委派模型

在文章开头我们已经提到两个类相等的前提条件应该是这两个类是由同一个类加载器加载的。既然 Java 中存在这么多的类加载器，那么 Java 是如何保证同一个类都是由同一个类加载器加载的呢？这主要得益于类加载器的“双亲委派模型”。接下来我们就来认识一下什么是“双亲委派模型”。

如下图所示，展示了各个类加载器之间的层次关系就是本节要讲的“双亲委派模型”

![](/picture/data-04.image)

双亲委派模型要求除了顶层启动类加载器外，其余的类加载器都应该有自己的父类加载器。而这里类加载之间的父子关系不是通过继承来实现的，而是通过组合的关系来复用类加载器的代码。

双亲委派模型的工作过程如下：

> 如果一个类加载器收到了类加载的请求，首先他不会自己尝试加载这个类，而是把这个请求委派给父类加载器完成，每个层次的类加载器都是如此。因此，所有的类加载请求最终都会被传送到最顶层的启动类加载器中，只有当父加载器无法找到这个加载请求的类时，子类加载器才会尝试去完成加载。

双亲委派模型的代码实现非常简单，如下：

```java
protected synchronized Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException {
        // 首先检查该类型是否已经被加载过了
        Class c = findLoadedClass(name);
        if (c == null) { //如果这个类还没有被加载，则尝试加载该类
            try {
                if (parent != null) { // 如果存在父类加载器，就委派给父类加载器加载
                    c = parent.loadClass(name, false);
                } else { // 如果不存在父类加载器，就尝试使用启动类加载器加载
                    c = findBootstrapClass0(name);
                }
            } catch (ClassNotFoundException e) {// 父类加载器找不到要加载的类，则抛出ClassNotFoundException 
                // 尝试调用自身的findClass方法进行类加载
                c = findClass(name);
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
```

这段代码的逻辑非常清晰易懂，代码中已经做了详细的注释说明。正式因为双亲委派模型具备一种带有优先级的层次关系，使得无论哪个类加载最终都会委派给处于最顶层的启动类加载器进行下载，因此程序中各个类加载器环境中都能够保证是同一类。



























