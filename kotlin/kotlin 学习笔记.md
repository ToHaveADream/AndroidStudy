1、变量类型分为 val(不可变) 和 var（可变）

var:可以不设置类型

val:如果不赋予初值，则必须指定类型（在所有的路径下）。如果赋予初值，则可以不指定，类型可以自动推导。在声明的时候可以不赋予初值，但是在赋予值之后，不可再更改。

2、基本数据类型

- kotlin 中一切皆对象，没有Java 中的原始基本类型，但 byte char integer float boolean 都有，但是都作为对象存在

- 对于数字没有隐式拓宽转换，但是在 Java 中 int 可以隐式转换为 long。需要强转。
- 所有未超过 int 的最大值的整型在初始化的变量都会自动推断为 int 类型，如果初值超过了最大值，则自动推断为 long，如需显示指定则在值后面加 L 后缀
- 字符不能视为数字
- 不支持八进制

```
val intIndex：Int = 100

val doubleIndex: Double = intIndex.toDouble()
// 以下代碼会报错
val doubleIndex: Double = intIndex
```

kotlin 的可空类型不能用 Java 的基本数据类型表示，因为 null 只能被存储在 Java 的引用类型的变量中，这意味着只要使用了基本数据类型的可空版本，它就会被编译成对应的包装类型。

```
// 基本数据类型
val intValue_1: Int = 200
// 包装类 intValue_2 和 intValue_3 是可空类型
val intValue_2: Int ?= intValue_1
val intValue_3: Int ?= intValue_1
// == 表示的是数值相等性
// === 比较的是引用是否相等 
如果 intValue_1 的值为 100，则 === 为 true，因为 Java 对包装类对象的重复使用
kotlin 可空类型被包装为基本类型
```

