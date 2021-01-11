## 弧度与角度

#### 相关定义

| 名称 | 定义                                                         |
| ---- | ------------------------------------------------------------ |
| 角度 | 两条射线从圆心向圆周射出，形成一条夹角和夹角正对的一段弧。当这段弧长正好等于圆周长的360分之一时，两条射线的夹角的大小为1度 |
| 弧度 | 两条射线从圆心向圆周射出，形成一条夹角和夹角正对的一段弧。当这段弧长正好等于圆的半径时，两条射线的夹角的大小为1弧度 |

### 换算公式

圆一周对应的角度为360度（角度），对应的弧度为2Π弧度   公式：(rad 是弧度， deg是角度)
$$
rad = deg * \pi /180
$$

### 角度区别

在常见的数学坐标系中角度增大的方向为逆时针，在默认的屏幕坐标系中角度增加的方式为顺时针



# 颜色

### 颜色模式

| 颜色模式 |         备注         |
| :------: | :------------------: |
| ARGB8888 | 四通道高精度（32位） |
| ARGB4444 | 四通道低精度（16位） |
|  RGB565  | 屏幕默认模式（16位） |
|  Alpha8  |     仅有透明通道     |

以ARGB8888为例简单介绍颜色定义：

|   类型   | 解释   | 0（0x00) | 255(0xff) |
| :------: | ------ | -------- | --------- |
| A(alpha) | 透明度 | 透明     | 不透明    |
|  R(Red)  | 红色   | 无色     | 红色      |
| G(Green) | 绿色   | 无色     | 绿色      |
| B(Blue)  | 蓝色   | 无色     | 蓝色      |

其中 A R G B的取值范围均为0 ~ 255(即16进制的0x00 ~ 0xFF)

A从0x00到0xFF表示从透明到不透明

RGB从0x00到0xFF表示颜色从浅到深

当RGB全部为最小值（0x00)时颜色为黑色，全取最大值(255)时颜色为白色

### 创建颜色的方法

#### Color给出几种颜色

```java
val color = Color.BLACK            //黑色
val color = Color.DKGRAY           //深灰色
val color = Color.GRAY             //灰色
val color = Color.LTGRAY           //亮灰色
val color = Color.WHITE            //白色
val color = Color.RED              //红色
val color = Color.GREEN            //绿色
val color = Color.BLUE             //蓝色
val color = Color.YELLOW           //黄色
val color = Color.CYAN             //青色
val color = Color.MAGENTA          //品红色
val color = Color.TRANSPARENT      //透明
```

#### 代码中定义

```java
val color = Color.argb(127, 255, 0, 0)   //半透明红色
val color = 0xaaff0000          //带有透明度的红色
```



#### /res/values/color.xml中定义

```java
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="red">#ff0000</color>
    <color name="green">#00ff00</color>
</resources>
```

几种不同的定义方式

```java
#f00            //低精度 - 不带透明通道红色
#af00           //低精度 - 带透明通道红色

#ff0000         //高精度 - 不带透明通道红色
#aaff0000       //高精度 - 带透明通道红色
```

在代码中引用定义的颜色的方法

```java
public static int getColor(@NonNull Context context, @ColorRes int id) {
    if (Build.VERSION.SDK_INT >= 23) {
        return context.getColor(id);
    } else {
        return context.getResources().getColor(id);
    }
}
```

Android中给出了一个兼容方法来兼容不同版本下的颜色获取，可以根据对应SDK版本来使用不同的方式。



在xml中引用或创建颜色：

```xml
<!--在style文件中引用-->
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <item name="colorPrimary">@color/red</item>
</style>

android:background="@color/red"     //引用在/res/values/color.xml 中定义的颜色
android:background="#ff0000"        //创建并使用颜色
```

### 颜色混合模式

因为我们的显示屏是没法透明的，因此最终显示在屏幕上的颜色里可以认为没有Alpha通道的。Alpha通道主要在两个图像混合的时候生效。

默认的情况下，当一个颜色绘制到Canvas上时的混合模式是这样进行计算的：

（RGB通道)最终颜色 = 绘制的颜色 + （1 - 绘制颜色的透明度） * Canvas上的原有颜色

下表是各个PorterDuff模式的混合计算公式：（D指原本在Canvas上的内容dst，S指绘制输入的内容src，a指alpha通道，c指RGB各个通道）

| PorterDuff.Mode     | 算法                                                     | 作用                                                         |
| ------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| CLEAR               | 【0，0】                                                 | 图像的alpha和rgb均为0                                        |
| SRC                 | [Sa, Sc]                                                 | 取原图像的值                                                 |
| DST                 | [Da, Dc]                                                 | 取目标图像的值                                               |
| SRC_OVER            | [Sa + (1 - Sa)Da, Rc = Sc + (1 - Sa)Dc]                  | 结果是Src覆盖在了Dst上。注意alpha值的影响，不一定是这个结果  |
| DST_OVER            | [Sa + (1 - Sa)Da, Rc = Dc + (1 - Da)Sc]                  | 结果是Dst盖在了Src上。注意alpha值的影响，不一定是这个结果    |
| SRC_IN              | [Sa * Da, Sc * Da]                                       | 结果是在Src色值不为0的地方，且Dst透明值不为0的地方能看到合成图像 |
| DST_IN              | [Sa * Da, Sa * Dc]                                       | 结果是在Dst色值不为0的地方，且Src透明值不为0的地方能看到合成图像 |
| SRC_OUT             | [Sa * (1 - Da), Sc * (1 - Da)]                           | 结果是在Src色值不为0，且Dst透明值不为1的地方能看到合成图像   |
| DST_OUT             | [Da * (1 - Sa), Dc * (1 - Sa)]                           | 结果是Dst色值不为0， 且src透明值不为1的地方可以看到合成图像  |
| SRC_ATOP            | [Da, Sc * Da + (1 - Sa) * Dc]                            | 结果是在Src和Dst色值不同时为0，且Dst的透明值不为0，且当Src色值为0但Src透明值不为1的地方能看到合成图像 |
| DST_ATOP            | [Sa, Sa * Dc + (1 - Da) * Sc]                            | 结果是在Src和Dst色值不同时为0，且Src的透明度不为0，且当Dst色值为0但Dst的透明度不为1的地方能看到图像 |
| XOR                 | [Sa + Da - 2 * Sa * Da, Sc * (1 - Da) + (1 - Sa) * Dc]   | 在不相交的地方按原样绘制源图像和目标图像，相交的地方受到对应alpha和色值影响 |
| DARKEN              | [Sa + Da - SaDa, Sc(1 - Da) + Dc*(1 - Sa) + min(Sc, Dc)] | 取较暗的透明值，色值计算相对复杂                             |
| LIGHTEN             | [Sa + Da - SaDa, Sc(1 - Da) + Dc*(1 - Sa) + max(Sc, Dc)] | 取较亮的透明值，色值计算相对复杂                             |
| MUTIPLY             | [Sa * Da, Sc * Dc]                                       | 结果是在Src和Dst透明值均不为0，且色值均不为0的地方能看到合成图像 |
| SCREEN              | [Sa + Da - Sa * Da, Sc + Dc - Sc * Dc]                   | 可以看到，它可能有多种情况，需要实际计算                     |
| ADD Saturate(S + D) |                                                          |                                                              |
| OVERLAY             |                                                          |                                                              |

#### Android颜色渲染PorterDuff以及Xfermode详解

看到PorterDuff单词觉得好奇怪，这是两个任命的组合：Tomas Porter 和Tom Duff. 他们是最早提出图形混合概念的大神级人物。

使用PorterDuff.Mode可以完成任意2D图像操作，比如涂鸦画板应用中的橡皮檫效果，绘制自定义的进度,etc

XferMode一共有三个子类：

AvoidXferMode指定了一个颜色和容差，强制Paint避免在它上面绘图（或者只在它上面绘图）

**PixelXorXferMode**当覆盖已有的颜色时，应用一个简单的像素异或操作

**PorterDuffXferMode**这是一个非常强大的转换模式，使用它，可以使用图像合成的16条Porter-Duff规则的任意一条来控制Paint如何与已有的Canvas图像进行交互。要应用转换模式，可以使用**setXferMode**方法，如下所示：