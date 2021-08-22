# Java Type 详解

#### 前言

错误可以分为两种：编译时错误和运行时错误。编译时错误在编译时可以发现并排除，而运行时错误具有很大的不确定性，在程序运行时才能发现，造成的后果可能是灾难性的。

泛型的引入使得一大部分错误可以提前到编译期间发现，极大的增强了代码的健壮性。但是我们知道 Java 泛型在运行时候是会进行泛型擦除的，那么怎么样才能得到在编译时期的泛型的信息呢？

Java 为我们提供了 Type 接口，使用它，我们可以得到这些信息。

**类型擦除是指泛型在运行的时候会去除泛型的类型信息**。Java 中，泛型主要是在编译层次中来实现的，在生成的字节码 即 class 文件是不包含泛型的 类型信息的。

即 List<String>，List<Object>,List<Integer> 虽然在编译时候是不同的，但是在编译完成后，在 class 文件中都只会把他们当做 List 来对待。

### 类 UML 图如下：

![](../picture/UML-04.png)

简单来说：Type 是所有类型的父接口，如原始类型（class）、参数化类型、数组类型、类型变量和基本原生类型 （class）。

子接口有  ParameterizedType, TypeVariable, GenericArrayType, WildcardType, 实现类有Class。

#### ParameterizedType --- 参数化类型

官方文档的说明是这样的

> ParameterizedType represents a parameterized type such as
> Collection<String>

**需要注意的是，并不只是 Collection<String> 才是 parameterized，任何类似于 ClassName 这样的类型都是 ParameterizedType ，比如下面的这些都是 parameterizedType.**

```
Map<String, Person> map;
Set<String> set1;
Class<?> clz;
Holder<String> holder;
List<String> list;
static class Holder<V>{	}
```

而类似于这样的 ClassName 不是 ParameterizedType。

```
Set set;
List aList;
```

### ParameterizedType 的几个主要方法

- Type[] getActualTypeArguments();
- Type getRawType();
- Type getOwnerType();

Type[] getActualTypeArguments();

返回这个 Type 类型的参数的实际类型数组。如 Map<String,Person> map 这个 Type 类型的参数的实际类型数组，如 Map<String,Person> map 这个 ParameterizedType 返回的是 String类，Person类的全限定类名的 Type Array。

Type getRawType();

返回的是当前这个 ParameterizedType 的类型。如 Map<String,Person> map 这个 ParameterizedType 返回的是 Map 类的全限定类名的 TypeArray。

Type getOwnerType()

> Returns a {@code Type} object representing the type that this type is a member of.

这个比较少用到。返回的是这个 ParameterizedType 所在的类的 Type （注意当前的 ParameterizedType 必须属于所在类的 member）。解释起来有点别扭，还是直接用代码说明吧。 比如 Map<String,Person> map 这个 ParameterizedType 的 getOwnerType() 为 null，而 Map.Entry<String, String>entry 的 getOwnerType() 为 Map 所属于的 Type
下面使用一个例子，来加深印象。

```
public class ParameterizedTypeBean {
	// 下面的 field 的 Type 属于 ParameterizedType
	Map<String, Person> map;
	Set<String> set1;
	Class<?> clz;
	Holder<String> holder;
	List<String> list;
	// Map<String,Person> map 这个 ParameterizedType 的 getOwnerType() 为 null，
	// 而 Map.Entry<String, String> entry 的 getOwnerType() 为 Map 所属于的 Type。
	Map.Entry<String, String> entry;
	// 下面的 field 的 Type 不属于ParameterizedType
	String str;
	Integer i;
	Set set;
	List aList;

	static class Holder<V> {

	}
}
```

```
public class TestHelper {

	public static void testParameterizedType() {
		Field f = null;
		try {
			Field[] fields = ParameterizedTypeBean.class.getDeclaredFields();
            // 打印出所有的 Field 的 TYpe 是否属于 ParameterizedType
			for (int i = 0; i < fields.length; i++) {
				f = fields[i];
				PrintUtils.print(f.getName()
						+ " getGenericType() instanceof ParameterizedType "
						+ (f.getGenericType() instanceof ParameterizedType));
			}
			getParameterizedTypeMes("map" );
			getParameterizedTypeMes("entry" );
			

		} catch (NoSuchFieldException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (SecurityException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}

	}

	private static void getParameterizedTypeMes(String fieldName) throws NoSuchFieldException {
		Field f;
		f = ParameterizedTypeBean.class.getDeclaredField(fieldName);
		f.setAccessible(true);
		PrintUtils.print(f.getGenericType());
		boolean b=f.getGenericType() instanceof ParameterizedType;
		PrintUtils.print(b);
		if(b){
			ParameterizedType pType = (ParameterizedType) f.getGenericType();
			PrintUtils.print(pType.getRawType());
			for (Type type : pType.getActualTypeArguments()) {
				PrintUtils.print(type);
			}
			PrintUtils.print(pType.getOwnerType()); // null
		}
	}
}
```

> print:map getGenericType() instanceof ParameterizedType true
>
> print:set1 getGenericType() instanceof ParameterizedType true
>
> print:clz getGenericType() instanceof ParameterizedType true
>
> print:holder getGenericType() instanceof ParameterizedType true
>
> print:list getGenericType() instanceof ParameterizedType true
>
> print:str getGenericType() instanceof ParameterizedType false
>
> print:i getGenericType() instanceof ParameterizedType false
>
> print:set getGenericType() instanceof ParameterizedType false
>
> print:aList getGenericType() instanceof ParameterizedType false
>
> print:entry getGenericType() instanceof ParameterizedType true
>
> print:java.util.Map<java.lang.String, com.xujun.gennericity.Person>
>
> print:true
>
> print:interface java.util.Map
>
> print:class java.lang.String
>
> print:class com.xujun.gennericity.Person
>
> print:null
>
> print:java.util.Map.java.util.Map$Entry<java.lang.String, java.lang.String>
>
> print:true
>
> print:interface java.util.Map$Entry
>
> print:class java.lang.String
>
> print:class java.lang.String
>
> print:interface java.util.Map

## **TypeVariable 变量**

比如 public class TypeVariableBean<K extends InputStream & Serializable, V> ，K ，V 都是属于类型变量。

### 主要方法

Type[] getBounds(); 得到上边界的 Type数组，如 K 的上边界数组是 InputStream 和 Serializable。 V 没有指定的话，上边界是 Object
D getGenericDeclaration(); 返回的是声明这个 Type 所在的类 的 Type
String getName(); 返回的是这个 type variable 的名称
