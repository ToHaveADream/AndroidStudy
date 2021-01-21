Path之完结篇

1、path方法详解

**rXxx**:方法

此类方法可以看到和前面的一些方法看起来很像，只不过在前面多了一个r，那么这个rXxx和前面的一些方法有什么区别呢？

> rXxx方法的坐标使用的是相对位移（基于当前点的位移），而之前方法的坐标是绝对位置（基于当前坐标系的坐标）

举个例子:kiwi_fruit:

```java
    Path path = new Path();

    path.moveTo(100,100);
    path.lineTo(100,200);

    canvas.drawPath(path,mDeafultPaint);
```

![](/picture/pathend-01.jpg)

这个例子中，先移动点到坐标（100，100）处，之后再连接点（100，100）到（100， 200）之间点直线，画出来是一条竖直的线。另外的一个例子:tomato:

```java
    Path path = new Path();

    path.moveTo(100,100);
    path.rLineTo(100,200);

    canvas.drawPath(path,mDeafultPaint);
```

![](/picture/pathend-02.jpg)

> 差别一看便知，相对位移和绝对位移的区别，相对位移就是再原来的基础上，加上相应的偏移得到的点

### 填充模式

我们再前面学习到，Paint有三种样式，“描边”“填充”以及”描边加填充“，我们这里所了解到就是在Paint设置为后两种样式时”不同的填充模式对图形渲染效果的影响。

我们要给一个图形内部填充颜色，首先要区分哪一部分是外部，哪一部分是内部，机器不像我们人那么聪明，机器是如何判断内外呢？

机器判断图形内外，一般有以下两种方法：

> 此处所有的图形都是封闭图形，不包括图形不封闭的情况

| 方法           | 判定条件                                 | 解释                                                         |
| -------------- | ---------------------------------------- | ------------------------------------------------------------ |
| 奇偶规则       | 奇数表示在图形内，偶数表示在图形外       | 从任意位置p做一条射线，若该射线相交的图形边的数目为奇数，则p是图形内部点，否则是外部点 |
| 非零环绕数规则 | 若环绕数为0表示在图形外，非0表示在图形内 | 首先使图形的边变为矢量。将环绕数初始化为零。再从任意位置p做一条射线。当从p点沿射线方向移动时，对在每个方向上穿过射线的边计数，每当图形的边从右到左穿过射线时，环绕数加1，从左到右时，环绕数减1.处理完图形的所有相关边后，若环绕数为非零，则p为内部点，否则，p是外部点 |

看一下两种判断方法是如何工作的。

**奇偶规则（Even-Odd RUle)**

这一个比较简单，也容易理解，直接用一个简单示例来说明。

![](/picture/pathend-03.jpg)

上图中有一个四边形，我们选取了三个点来判断这些点是否在图形内部。

> P1: 从P1发出一条射线，发现图形与该射线相交边数为0，偶数，故P1点在图形外部。
> P2: 从P2发出一条射线，发现图形与该射线相交边数为1，奇数，故P2点在图形内部。
> P3: 从P3发出一条射线，发现图形与该射线相交边数为2，偶数，故P3点在图形外部

**非零环绕数规则（Non-zero  Winding number Rule)**

Path中任何线段都是由方向性的，这也是使用非零环绕数规则的基础。

看例子：

![](/picture/pathend-04.jpg)

> P1: 从P1点发出一条射线，沿射线方向移动，并没有与边相交点部分，环绕数为0，故P1在图形外边。
> P2: 从P2点发出一条射线，沿射线方向移动，与图形点左侧边相交，该边从左到右穿过射线，环绕数－1，最终环绕数为－1，故P2在图形内部。
> P3: 从P3点发出一条射线，沿射线方向移动，在第一个交点处，底边从右到左穿过射线，环绕数＋1，在第二个交点处，右侧边从左到右穿过射线，环绕数－1，最终环绕数为0，故P3在图形外部

**自相交图形**

**自相交图形定义：多边形在平面内除顶点外还有其他公共点**

简单的提一下自相交图形，了解概念即可，下面就是一个简单的自相交图形：

![](/picture/pathend-05.jpg)

**Android中的填充模式**
Android中的填充模式有四种，是封装在Path中的一个枚举

| 模式             | 简介             |
| ---------------- | ---------------- |
| EVEN_ODD         | 奇偶规则         |
| INVERSE_EVEN_ODD | 反奇偶规则       |
| WINDING          | 非零环绕数规则   |
| INVERSE_WINDING  | 反非零环绕数规则 |

我们可以看到上面的四种模式，分别成对，例如“奇偶规则”与“反奇偶规则”是一对。

Inverse的含义是“相反，对立”，说明反奇偶规则刚好与奇偶规则相反，例如对于一个矩形而言，使用奇偶规则会填充矩形内部，而使用反奇偶规则会填充矩形外部，这个在后面示例中会展示两者的区别。

**Android与填充模式相关的方法**

> 这些都是Path中的方法

| 方法                  | 作用                                             |
| --------------------- | ------------------------------------------------ |
| setFillType           | 设置填充规则                                     |
| getFillType           | 获取当前填充规则                                 |
| isInverseFillType     | 获取是否是反向规则                               |
| toggleInverseFillType | 切换填充规则（即原有规则与反向规则之间相互切换） |

**演示**

**奇偶规则与反奇偶规则**

```java
    mDeafultPaint.setStyle(Paint.Style.FILL);                   // 设置画布模式为填充

    canvas.translate(mViewWidth / 2, mViewHeight / 2);          // 移动画布(坐标系)

    Path path = new Path();                                     // 创建Path

    //path.setFillType(Path.FillType.EVEN_ODD);                   // 设置Path填充模式为 奇偶规则
    path.setFillType(Path.FillType.INVERSE_EVEN_ODD);            // 反奇偶规则

    path.addRect(-200,-200,200,200, Path.Direction.CW);         // 给Path中添加一个矩形
```

下面两张图片分别是在奇偶规则与反奇偶规则的情况下绘制的结果，可以看出其填充的区域刚好相反：

> 白色为背景色，黑色为填充色

![](/picture/pathend-07.jpg)![](/picture/pathend-06.jpg) 



**图形边的方向对非零奇偶环绕数规则填充结果的影响**

前面我们了解了在图形边的方向会如何影响到填充效果，我们这里验证一下：

```java
 mDeafultPaint.setStyle(Paint.Style.FILL);                   // 设置画笔模式为填充

    canvas.translate(mViewWidth / 2, mViewHeight / 2);          // 移动画布(坐系)

    Path path = new Path();                                     // 创建Path

    // 添加小正方形 (通过这两行代码来控制小正方形边的方向,从而演示不同的效果)
    // path.addRect(-200, -200, 200, 200, Path.Direction.CW);
    path.addRect(-200, -200, 200, 200, Path.Direction.CCW);

    // 添加大正方形
    path.addRect(-400, -400, 400, 400, Path.Direction.CCW);

    path.setFillType(Path.FillType.WINDING);                    // 设置Path填充模式为非零环绕规则

    canvas.drawPath(path, mDeafultPaint);                       // 绘制Path
```

![](/picture/pathend-08.jpg) ![](/picture/pathend-09.jpg)

#### 布尔操作

布尔操作与我们学生时代学习的集合操作比较像，只要知道集合操作中的交集，并集，差集等操作，那么理解布尔操作也是很容易的。

**布尔操作是两个Path之间的运算，主要作用是用一些简单的图形通过一些规则合成相对比较复杂，或难以直接得到的图形**

如太极中的阴阳鱼，如果使用贝塞尔曲线制作，可能需要六段贝塞尔曲线才行，而在这里我们可以用四个Path通过布尔运算得到，而且会相对来说更容易一点。

![](/picture/pathend-10.jpg)

```java
    canvas.translate(mViewWidth / 2, mViewHeight / 2);

    Path path1 = new Path();
    Path path2 = new Path();
    Path path3 = new Path();
    Path path4 = new Path();

    path1.addCircle(0, 0, 200, Path.Direction.CW);
    path2.addRect(0, -200, 200, 200, Path.Direction.CW);
    path3.addCircle(0, -100, 100, Path.Direction.CW);
    path4.addCircle(0, 100, 100, Path.Direction.CCW);


    path1.op(path2, Path.Op.DIFFERENCE);
    path1.op(path3, Path.Op.UNION);
    path1.op(path4, Path.Op.DIFFERENCE);

    canvas.drawPath(path1, mDeafultPaint);
```

前面演示了布尔运算的作用，接下来了解一下布尔运算的核心逻辑：布尔逻辑。

Path的布尔运算有五种逻辑，如下：

| 逻辑名称           | 类比 | 说明                                   | 示意图                        |
| ------------------ | ---- | -------------------------------------- | ----------------------------- |
| DIFFERENCE         | 差集 | Path1中减去Path2后剩下的部分           | ![](/picture/pathbool-01.jpg) |
| REVERSE_DIFFERENCE | 差集 | Path2中减去Path1后剩下的部分           | ![](/picture/pathbool-02.jpg) |
| INTERSECT          | 交集 | Path1与Path2相交的部分                 | ![](/picture/pathbool-03.jpg) |
| UNION              | 并集 | 包含全部Path1和Path2                   | ![](/picture/pathbool-04.jpg) |
| XOR                | 异或 | 包含Path1与Path2但不包括两者相交的部分 | ![](/picture/pathbool-05.jpg) |

**布尔运算方法**

在Path中的布尔运算有两个方法

```java
    boolean op (Path path, Path.Op op)
    boolean op (Path path1, Path path2, Path.Op op)
```

两个方法的返回值用于判断布尔运算是否成功，他们使用方法如下：

```java
    // 对 path1 和 path2 执行布尔运算，运算方式由第二个参数指定，运算结果存入到path1中。
    path1.op(path2, Path.Op.DIFFERENCE);
    
    // 对 path1 和 path2 执行布尔运算，运算方式由第三个参数指定，运算结果存入到path3中。
    path3.op(path1, path2, Path.Op.DIFFERENCE)
```

**布尔运算示例**

![](/picture/pathbool-06.jpg)

代码：

```java
    int x = 80;
    int r = 100;

    canvas.translate(250,0);

    Path path1 = new Path();
    Path path2 = new Path();
    Path pathOpResult = new Path();

    path1.addCircle(-x, 0, r, Path.Direction.CW);
    path2.addCircle(x, 0, r, Path.Direction.CW);

    pathOpResult.op(path1,path2, Path.Op.DIFFERENCE);
    canvas.translate(0, 200);
    canvas.drawText("DIFFERENCE", 240,0,mDeafultPaint);
    canvas.drawPath(pathOpResult,mDeafultPaint);

    pathOpResult.op(path1,path2, Path.Op.REVERSE_DIFFERENCE);
    canvas.translate(0, 300);
    canvas.drawText("REVERSE_DIFFERENCE", 240,0,mDeafultPaint);
    canvas.drawPath(pathOpResult,mDeafultPaint);

    pathOpResult.op(path1,path2, Path.Op.INTERSECT);
    canvas.translate(0, 300);
    canvas.drawText("INTERSECT", 240,0,mDeafultPaint);
    canvas.drawPath(pathOpResult,mDeafultPaint);

    pathOpResult.op(path1,path2, Path.Op.UNION);
    canvas.translate(0, 300);
    canvas.drawText("UNION", 240,0,mDeafultPaint);
    canvas.drawPath(pathOpResult,mDeafultPaint);

    pathOpResult.op(path1,path2, Path.Op.XOR);
    canvas.translate(0, 300);
    canvas.drawText("XOR", 240,0,mDeafultPaint);
    canvas.drawPath(pathOpResult,mDeafultPaint);
```

**计算边界**

这个方法主要作用是计算path所占用的空间以及所在位置，方法如下：

```
 void computeBounds (RectF bounds, boolean exact)
```

它有两个参数：

| 参数   | 作用                                                     |
| ------ | -------------------------------------------------------- |
| bounds | 测量结果会放入这个矩形                                   |
| exact  | 是否精确测量，目前这一个参数作用已经废弃，一般写true即可 |

**计算边界示例**

计算path边界的一个简单示例：

![](/picture/pathend-11.jpg)

代码如下：

```java
    // 移动canvas,mViewWidth与mViewHeight在 onSizeChanged 方法中获得
    canvas.translate(mViewWidth/2,mViewHeight/2);

    RectF rect1 = new RectF();              // 存放测量结果的矩形

    Path path = new Path();                 // 创建Path并添加一些内容
    path.lineTo(100,-50);
    path.lineTo(100,50);
    path.close();
    path.addCircle(-100,0,100, Path.Direction.CW);

    path.computeBounds(rect1,true);         // 测量Path

    canvas.drawPath(path,mDeafultPaint);    // 绘制Path

    mDeafultPaint.setStyle(Paint.Style.STROKE);
    mDeafultPaint.setColor(Color.RED);
    canvas.drawRect(rect1,mDeafultPaint);   // 绘制边界
```

**重置路径**

重置path有两个方法，分别是reset和rewind，两者区别主要有以下两点：

| 方法   | 是否保留FillType设置 | 是否保留原有数据结构 |
| ------ | -------------------- | -------------------- |
| reset  | 是                   | 否                   |
| rewind | 否                   | 是                   |

**这两个方法应该何时选择呢？**

选择权重：FillType > 数据结构

因为”FillType"影响的是显示效果，而“数据结构”影响的是重建速度。

