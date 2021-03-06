### Canvas之画布操作

#### Canvas基本操作

##### 1.画布操作

<font color = red> 为什么要有画布操作</font>?

画布操作可以帮助我们用更加容易理解的方式制作图形

例如：从坐标原点为起点，绘制一个长度20dp，与水平线夹角为30度的线段怎么做？

按照我们通常的想法，就是先使用三角函数计算出线段结束点的坐标，然后调用drawLine即可。但是是否被禁锢了？

假设我们先绘制一长度为20dp的水平线，然后将这条水平线旋转30度，则最终看起来效果是相同的，而且不用进行三角函数计算，这样是否更加简单一点了呢？

<font size = 4>**合理的使用画布操作可以帮助我们用给更加容易理解的方式创作想要的效果，这也是画布操作存在的原因 **</font>

> 所有的画布操作都只影响后续的绘制，对之前已经绘制过的内容没有影响

------

1）位移（translate）

translate是坐标系的移动，可以为图形绘制选择一个<font color= red size = 5>合适的坐标系</font>。请注意，**位移是基于当前位置移动，而不是基于屏幕左上角的（0，0）点移动**，如下：

> 画笔默认是从（0，0）开始

```java
// 省略了创建画笔的代码

// 在坐标原点绘制一个黑色圆形
mPaint.setColor(Color.BLACK);
canvas.translate(200,200);
canvas.drawCircle(0,0,100,mPaint);

// 在坐标原点绘制一个蓝色圆形
mPaint.setColor(Color.BLUE);
canvas.translate(200,200);
canvas.drawCircle(0,0,100,mPaint);
```

![](canvas-translate-01.PNG)

我们首先将坐标系移动一段距离绘制一个圆形，之后再移动一段距离绘制一个圆形，两次移动是可叠加的

------

2）缩放（scale)

缩放提供了两个方法，如下：

```java
public void scale (float sx, float sy)

public final void scale (float sx, float sy, float px, float py)
```

这两个方法中前两个参数是相同的：分别为x轴和y轴的缩放比例。第二种方法比前一种多了两个参数，用来控制缩放中心位置的

缩放比例（sx,sy)取值范围详解：

| 取值范围       | 说明                                           |
| -------------- | ---------------------------------------------- |
| (负无穷，-1)   | 先根据缩放中心点放大n倍，再根据中心轴进行翻转  |
| -1             | 根据缩放中心进行翻转                           |
| （-1，0）      | 先根据缩放中心缩小到n，再根据中心轴进行翻转    |
| 0              | 不会显示，若sx为0，则宽度为0，不会显示，sy同理 |
| （0，1）       | 根据缩放中心缩小到n                            |
| 1              | 没有变化                                       |
| （1， 正无穷） | 根据缩放中心放大n倍                            |

如果在缩放时稍微注意一下就会发现**缩放的中心默认为坐标原点，而缩放中心轴就是坐标轴 ，如下所示：

```java
// 将坐标系原点移动到画布正中心
canvas.translate(mWidth / 2, mHeight / 2);

RectF rect = new RectF(0,-400,400,0);   // 矩形区域

mPaint.setColor(Color.BLACK);           // 绘制黑色矩形
canvas.drawRect(rect,mPaint);

canvas.scale(0.5f,0.5f);                // 画布缩放

mPaint.setColor(Color.BLUE);            // 绘制蓝色矩形
canvas.drawRect(rect,mPaint);
```

![](canvas-scale-01.PNG)

> 缩放中心为原点

接下来根据第二种的方法让**缩放中心**位置稍微的进行变化，如下所示：

```java
// 将坐标系原点移动到画布正中心
canvas.translate(mWidth / 2, mHeight / 2);

RectF rect = new RectF(0,-400,400,0);   // 矩形区域

mPaint.setColor(Color.BLACK);           // 绘制黑色矩形
canvas.drawRect(rect,mPaint);

canvas.scale(0.5f,0.5f,200,0);          // 画布缩放  <-- 缩放中心向右偏移了200个单位

mPaint.setColor(Color.BLUE);            // 绘制蓝色矩形
canvas.drawRect(rect,mPaint);
```

![](canvas-scale-02.PNG)

> 图中箭头所指的就是缩放中心

按照表格中的说明，**当缩放比例为负数的时候会根据缩放中心轴进行翻转**，如下所示：

```java
// 将坐标系原点移动到画布正中心
canvas.translate(mWidth / 2, mHeight / 2);

RectF rect = new RectF(0,-400,400,0);   // 矩形区域

mPaint.setColor(Color.BLACK);           // 绘制黑色矩形
canvas.drawRect(rect,mPaint);


canvas.scale(-0.5f,-0.5f);          // 画布缩放

mPaint.setColor(Color.BLUE);            // 绘制蓝色矩形
canvas.drawRect(rect,mPaint);
```

![](canvas-scale-03.PNG)

> 为了显示效果，添加了坐标系，并且对矩形中几个重要的点进行了标注

由于未对缩放中心进行偏移，所有的默认的缩放中心就是原点，中心轴就是x轴和y轴

本次缩放可以看作是先根据缩放中心缩放到原理的0.5倍，然后分别按照x轴和y轴进行翻转

```java
// 将坐标系原点移动到画布正中心
canvas.translate(mWidth / 2, mHeight / 2);

RectF rect = new RectF(0,-400,400,0);   // 矩形区域

mPaint.setColor(Color.BLACK);           // 绘制黑色矩形
canvas.drawRect(rect,mPaint);

canvas.scale(-0.5f,-0.5f,200,0);          // 画布缩放  <-- 缩放中心向右偏移了200个单位

mPaint.setColor(Color.BLUE);            // 绘制蓝色矩形
canvas.drawRect(rect,mPaint);
```

![](canvas-scale-04.PNG)

本次对缩放中心Y轴坐标进行了偏移，故中心轴也向右偏移了

> 和位移（translate）是一样的，缩放也是可以叠加的

```
canvas.scale(0.5f,0.5f);
canvas.scale(0.5f,0.1f);
```

调用两次缩放，则X轴实际缩放0.5 * 0.5=0.25，y轴实际缩放0.5 * 0.1 = 0.05

下面制作一个有趣的图形：

```java
// 将坐标系原点移动到画布正中心  填充模式为描边
canvas.translate(mWidth / 2, mHeight / 2);

RectF rect = new RectF(-400,-400,400,400);   // 矩形区域

for (int i=0; i<=20; i++)
{
	canvas.scale(0.9f,0.9f);
	canvas.drawRect(rect,mPaint);
}
```

![](canvas-scale-05.PNG)

------

3)旋转（rotate)

旋转提供了两种方法：

```java
public void rotate (float degrees)

public final void rotate (float degrees, float px, float py)
```

和缩放一样，第二种方法多出来的两个参数依旧是控制旋转中心点的。

默认的旋转中心依旧是坐标原点：

```java
// 将坐标系原点移动到画布正中心
canvas.translate(mWidth / 2, mHeight / 2);

RectF rect = new RectF(0,-400,400,0);   // 矩形区域

mPaint.setColor(Color.BLACK);           // 绘制黑色矩形
canvas.drawRect(rect,mPaint);

canvas.rotate(180);                     // 旋转180度 <-- 默认旋转中心为原点

mPaint.setColor(Color.BLUE);            // 绘制蓝色矩形
canvas.drawRect(rect,mPaint);
```

![](canvas-scale-06.PNG)

改变旋转中心位置：

```java
// 将坐标系原点移动到画布正中心
canvas.translate(mWidth / 2, mHeight / 2);

RectF rect = new RectF(0,-400,400,0);   // 矩形区域

mPaint.setColor(Color.BLACK);           // 绘制黑色矩形
canvas.drawRect(rect,mPaint);

canvas.rotate(180,200,0);               // 旋转180度 <-- 旋转中心向右偏移200个单位

mPaint.setColor(Color.BLUE);            // 绘制蓝色矩形
canvas.drawRect(rect,mPaint);  //旋转的为canvas画布的坐标系
```

![](canvas-scale-07.PNG)

旋转也可以是叠加的：

```java
canvas.rotate(180);
canvas.rotate(20);
```

调整两次旋转，实际的旋转角度为180 + 20 = 200度

为了演示这个效果，如下所示：

```javascript
// 将坐标系原点移动到画布正中心
canvas.translate(mWidth / 2, mHeight / 2);

canvas.drawCircle(0,0,400,mPaint);          // 绘制两个圆形
canvas.drawCircle(0,0,380,mPaint);

for (int i=0; i<=360; i+=10){               // 绘制圆形之间的连接线
   canvas.drawLine(0,380,0,400,mPaint);
   canvas.rotate(10);
}
```

![](canvas-scale-08.PNG)

4）错切（skew)

skew这里翻译为错切，错切是特殊类型的线性变换

错切只提供了一种方法：

```java
public void skew (float sx, float sy)
```

参数含义：

float sx:将画布在x方向上倾斜相应的角度，sx倾斜角度的tan值

float sy:将画布在y轴方向上倾斜相应的角度，sy为倾斜角度的tan值

变换后：

```java
X = x +sx * y
Y = y + sy * x
```

如下所示：

```java
// 将坐标系原点移动到画布正中心
canvas.translate(mWidth / 2, mHeight / 2);

RectF rect = new RectF(0,0,200,200);   // 矩形区域

mPaint.setColor(Color.BLACK);           // 绘制黑色矩形
canvas.drawRect(rect,mPaint);

canvas.skew(1,0);                       // 水平错切 <- 45度

mPaint.setColor(Color.BLUE);            // 绘制蓝色矩形
canvas.drawRect(rect,mPaint);
```

![](canvas-scale-09.PNG)

如你所想，错切也是可叠加的，不过请注意，调用次序不同绘制结果也不同

```java
// 将坐标系原点移动到画布正中心
canvas.translate(mWidth / 2, mHeight / 2);

RectF rect = new RectF(0,0,200,200);   // 矩形区域

mPaint.setColor(Color.BLACK);           // 绘制黑色矩形
canvas.drawRect(rect,mPaint);

canvas.skew(1,0);                       // 水平错切
canvas.skew(0,1);                       // 垂直错切

mPaint.setColor(Color.BLUE);            // 绘制蓝色矩形
canvas.drawRect(rect,mPaint);
```

![](canvas-scale-10.PNG)

5)快照（save)和回滚（restore)

Q:为什么存在快照和回滚

A:画布的操作是不可逆的，而且很多画布操作会影响后续的步骤，例如第一个例子，两个圆形都是在坐标原点绘制的，而因为坐标系的移动绘制出来的实际位置不同。所以会对画布的状态进行保存和回滚

**与之相关的API**

| 相关API        | 简介                                                         |
| -------------- | ------------------------------------------------------------ |
| save           | 把当前画布的状态进行保存，然后放入特定的栈中                 |
| saveLayerxxx   | 新建一个图层，并放入特定的栈中                               |
| restore        | 把栈中最顶层的画布状态取出来，并按照这个状态恢复当前的画布   |
| restoreToCount | 弹出指定位置及其以上的所有的状态，并按照指定位置的状态进行恢复 |
| getSaveCount   | 获取栈中内容的数量（即保存的次数）                           |

下面对其中的一些概念和方法进行分析：

**状态栈**：

![](canvas-state-01.PNG)

这个栈可以存储画布状态和图层状态

**Q:什么是画布状态和图层状态**

A:实际上我们看到的画布是由多个图层构成的，如下图：

![](canvas-state-02.PNG)

实际上我们之前讲解的绘制操作和画布操作都是在默认图层上面进行的。在通常情况下，使用默认图层就可以满足需求，但是如果需要绘制比较复杂的内容；如地图（地图可以由多个图层叠加而成，比如：道路图、政区层），则分图层绘制比较好一点：

你可以把这个图层看作是一层一层的玻璃板，你在每层玻璃板上绘制内容、然后把这些玻璃板叠起来看就是最终效果。

### SaveFlags:

| 名称                       | 简介                                            |
| -------------------------- | ----------------------------------------------- |
| ALL_SAVE_FLAG              | 默认，保存全部状态                              |
| CLIP_SAVE_FLAG             | 保存剪辑区                                      |
| CLIP_TO_LAYER_SAVE_FLAG    | 剪辑区作为图层保存                              |
| FULL_COLOR_LAYER_SAVE_FLAG | 保存图层的全部色彩通道                          |
| HAS_ALPHA_LAYER_SAVE_FLAG  | 保存图层的alpha(不透明度)通道                   |
| MATRIX_SAVE_FLAG           | 保存Matrix信息( translate, rotate, scale, skew) |

save

save由两种方法：

```java
// 保存全部状态
public int save ()

// 根据saveFlags参数保存一部分状态
public int save (int saveFlags)
```

可以看到第二种方法比第一个方法多了一个saveFlags参数，使用这个参数可以只保存一部分状态，更加灵活，这个saveFlags参数具体可以参考上面表格中的内容。

每调用一次save方法，都会在栈顶添加一条状态信息，以上面状态栈图片为例，再调用一次save则会在第5次上面再添加一条状态。

saveLayerXXX

saveLayerxxx有比较多的方法

```java
// 无图层alpha(不透明度)通道
public int saveLayer (RectF bounds, Paint paint)
public int saveLayer (RectF bounds, Paint paint, int saveFlags)
public int saveLayer (float left, float top, float right, float bottom, Paint paint)
public int saveLayer (float left, float top, float right, float bottom, Paint paint, int saveFlags)

// 有图层alpha(不透明度)通道
public int saveLayerAlpha (RectF bounds, int alpha)
public int saveLayerAlpha (RectF bounds, int alpha, int saveFlags)
public int saveLayerAlpha (float left, float top, float right, float bottom, int alpha)
public int saveLayerAlpha (float left, float top, float right, float bottom, int alpha, int saveFlags)
```

> saveLayerxxx方法会让你花费更多的时间去渲染图像（图层多了相互之间叠加会导致计算量成倍增长），使用前请谨慎，如果可能，尽量避免使用

使用saveLayerxxx方法，也会将图层状态放入状态栈中，同样使用restore方法进行恢复

**restore**

状态回滚，就是从栈顶取出一个状态然后根据内容进行恢复

同样以上面状态栈图片为例，调用一次restore方法则将恢复状态栈中第5此取出，根据里面保存的状态进行恢复

**restoreToCount**

弹出指定位置以及以上所有的状态，并根据指定位置状态进行恢复

以上图进行示例，如果调用restoreToCount（2）则会弹出2 3 4 5的状态，并根据第2次保存的状态进行恢复

**getSaveCount **

获取保存的次数，即状态栈中保存状态的数量，以上面状态图片为例，使用该函数的返回值为5

不过请注意，该函数的最小返回值为1，即使弹出了所有的状态，返回值依旧为1，代表默认状态

