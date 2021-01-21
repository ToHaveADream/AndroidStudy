# Path之玩出花样（PathMeasure)

![](/picture/pathmesaure-01.gif)

## Path & PathMeasure

故名思义，PathMeasure是一个用来测量Path的类，主要有以下方法：

#### 构造方法

| 方法名                                      | 释义                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| PathMeasure()                               | 创建一个空的PathMeasure                                      |
| PathMeasure(Path path, boolean forceClosed) | 创建 PathMeasure 并关联一个指定的Path(Path需要已经创建完成)。 |

#### 公共方法

| 返回值  | 方法名                                                       | 释义                               |
| ------- | ------------------------------------------------------------ | ---------------------------------- |
| void    | setPath(Path path, boolean forceClosed)                      | 关联一个Path                       |
| boolean | isClosed()                                                   | 是否闭合                           |
| float   | getLength()                                                  | 获取Path的长度                     |
| boolean | nextContour()                                                | 跳转到下一个轮廓                   |
| boolean | getSegment(float startD, float stopD, Path dst, boolean startWithMoveTo) | 截取片段                           |
| boolean | getPosTan(float distance, float[] pos, float[] tan)          | 获取指定长度的位置坐标及该点切线值 |
| boolean | getMatrix(float distance, Matrix matrix, int flags)          | 获取指定长度的位置坐标及该点Matrix |

### 1.构造函数

构造函数有两个。

**无参构造函数：**

```
  PathMeasure ()
```

用这个构造函数可创建一个空的 PathMeasure，但是使用之前需要先调用 setPath 方法来与 Path 进行关联。被关联的 Path 必须是已经创建好的，如果关联之后 Path 内容进行了更改，则需要使用 setPath 方法重新关联。

**有参构造函数：**

```
  PathMeasure (Path path, boolean forceClosed)
```

用这个构造函数是创建一个 PathMeasure 并关联一个 Path， 其实和创建一个空的 PathMeasure 后调用 setPath 进行关联效果是一样的，同样，被关联的 Path 也必须是已经创建好的，如果关联之后 Path 内容进行了更改，则需要使用 setPath 方法重新关联。

该方法有两个参数，第一个参数自然就是被关联的 Path 了，第二个参数是用来确保 Path 闭合，如果设置为 true， 则不论之前Path是否闭合，都会自动闭合该 Path(如果Path可以闭合的话)。

**在这里有两点需要明确:**

- 1. 不论 forceClosed 设置为何种状态(true 或者 false)， 都不会影响原有Path的状态，**即 Path 与 PathMeasure 关联之后，之前的的 Path 不会有任何改变。**
- 1. forceClosed 的设置状态可能会影响测量结果，**如果 Path 未闭合但在与 PathMeasure 关联的时候设置 forceClosed 为 true 时，测量结果可能会比 Path 实际长度稍长一点，获取到到是该 Path 闭合时的状态。**

验证一下：

```
canvas.translate(mViewWidth/2,mViewHeight/2);

Path path = new Path();

path.lineTo(0,200);
path.lineTo(200,200);
path.lineTo(200,0);

PathMeasure measure1 = new PathMeasure(path,false);
PathMeasure measure2 = new PathMeasure(path,true);

Log.e("TAG", "forceClosed=false---->"+measure1.getLength());
Log.e("TAG", "forceClosed=true----->"+measure2.getLength());

canvas.drawPath(path,mDeafultPaint);
```

log如下：

```
com.gcssloop.canvas E/TAG: forceClosed=false---->600.0
com.gcssloop.canvas E/TAG: forceClosed=true----->800.0
```

我们所创建的 Path 实际上是一个边长为 200 的正方形的三条边，通过上面的示例就能验证以上两个问题。

- 1.我们将 Path 与两个的 PathMeasure 进行关联，并给 forceClosed 设置了不同的状态，之后绘制再绘制出来的 Path 没有任何变化，所以与 Path 与 PathMeasure进行关联并不会影响 Path 状态。
- 2.我们可以看到，设置 forceClosed 为 true 的方法比设置为 false 的方法测量出来的长度要长一点，这是由于 Path 没有闭合的缘故，多出来的距离正是 Path 最后一个点与最开始一个点之间点距离。**forceClosed 为 false 测量的是当前 Path 状态的长度， forceClosed 为 true，则不论Path是否闭合测量的都是 Path 的闭合长度。**

### 2.setPath、 isClosed 和 getLength

这三个方法都如字面意思一样，非常简单，这里就简单是叙述一下，不再过多讲解。

setPath 是 PathMeasure 与 Path 关联的重要方法，效果和 构造函数 中两个参数的作用是一样的。

isClosed 用于判断 Path 是否闭合，但是如果你在关联 Path 的时候设置 forceClosed 为 true 的话，这个方法的返回值则一定为true。

getLength 用于获取 Path 的总长度，在之前的测试中已经用过了。

### 3.getSegment

getSegment 用于获取Path的一个片段，方法如下：

```
boolean getSegment (float startD, float stopD, Path dst, boolean startWithMoveTo)
```

方法各个参数释义：

| 参数            | 作用                             | 备注                                                         |
| --------------- | -------------------------------- | ------------------------------------------------------------ |
| 返回值(boolean) | 判断截取是否成功                 | true 表示截取成功，结果存入dst中，false 截取失败，不会改变dst中内容 |
| startD          | 开始截取位置距离 Path 起点的长度 | 取值范围: 0 <= startD < stopD <= Path总长度                  |
| stopD           | 结束截取位置距离 Path 起点的长度 | 取值范围: 0 <= startD < stopD <= Path总长度                  |
| dst             | 截取的 Path 将会添加到 dst 中    | 注意: 是添加，而不是替换                                     |
| startWithMoveTo | 起始点是否使用 moveTo            | 用于保证截取的 Path 第一个点位置不变                         |

- 如果 startD、stopD 的数值不在取值范围 [0, getLength] 内，或者 startD == stopD 则返回值为 false，不会改变 dst 内容。
- 如果在安卓4.4或者之前的版本，在默认开启硬件加速的情况下，更改 dst 的内容后可能绘制会出现问题，请关闭硬件加速或者给 dst 添加一个单个操作，例如: dst.rLineTo(0, 0)

先看看如何使用：

创建一个Path，并在其中添加了一个矩形，我们想截取矩形中的一部分，就是下图中的红色部分

> 矩形边长400dp，起始点在左上角，顺时针

![](/picture/pathmeasure-02.jpg)

代码如下：

```java
canvas.translate(mViewWidth / 2, mViewHeight / 2);          // 平移坐标系

Path path = new Path();                                     // 创建Path并添加了一个矩形
path.addRect(-200, -200, 200, 200, Path.Direction.CW);

Path dst = new Path();                                      // 创建用于存储截取后内容的 Path

PathMeasure measure = new PathMeasure(path, false);         // 将 Path 与 PathMeasure 关联

// 截取一部分存入dst中，并使用 moveTo 保持截取得到的 Path 第一个点的位置不变
measure.getSegment(200, 600, dst, true);                    

canvas.drawPath(dst, mDeafultPaint);                        // 绘制 dst
```

结果如下：

![](/picture/pathmeasure-03.jpg)

然而当dst中有内容的时候会怎样？

```java
canvas.translate(mViewWidth / 2, mViewHeight / 2);          // 平移坐标系

Path path = new Path();                                     // 创建Path并添加了一个矩形
path.addRect(-200, -200, 200, 200, Path.Direction.CW);

Path dst = new Path();                                      // 创建用于存储截取后内容的 Path
dst.lineTo(-300, -300);                                     // <--- 在 dst 中添加一条线段

PathMeasure measure = new PathMeasure(path, false);         // 将 Path 与 PathMeasure 关联

measure.getSegment(200, 600, dst, true);                   // 截取一部分 并使用 moveTo 保持截取得到的 Path 第一个点的位置不变

canvas.drawPath(dst, mDeafultPaint);                        // 绘制 Path
```

![](/picture/pathmesaure-04.jpg)

结论如下：被截取到Path片段会被添加到dst中，而不是替换dst中的内容。

如果startWithMoveTo都是false？

```java
canvas.translate(mViewWidth / 2, mViewHeight / 2);          // 平移坐标系

Path path = new Path();                                     // 创建Path并添加了一个矩形
path.addRect(-200, -200, 200, 200, Path.Direction.CW);

Path dst = new Path();                                      // 创建用于存储截取后内容的 Path
dst.lineTo(-300, -300);                                     // 在 dst 中添加一条线段

PathMeasure measure = new PathMeasure(path, false);         // 将 Path 与 PathMeasure 关联

measure.getSegment(200, 600, dst, false);                   // <--- 截取一部分 不使用 startMoveTo, 保持 dst 的连续性

canvas.drawPath(dst, mDeafultPaint);                        // 绘制 Path
```

结果如下：

![](/picture/pathmeasure-05.jpg)

从该示例我们又可以得到一条结论：**如果 startWithMoveTo 为 true, 则被截取出来到Path片段保持原状，如果 startWithMoveTo 为 false，则会将截取出来的 Path 片段的起始点移动到 dst 的最后一个点，以保证 dst 的连续性。**

从而我们可以用以下规则来判断 startWithMoveTo 的取值：

| 取值  | 主要功用                              |
| ----- | ------------------------------------- |
| true  | 保证截取得到的 Path 片段不会发生形变  |
| false | 保证存储截取片段的 Path(dst) 的连续性 |

### 4.nextContour

我们知道 Path 可以由多条曲线构成，但不论是 getLength , getSegment 或者是其它方法，都只会在其中第一条线段上运行，而这个 `nextContour` 就是用于跳转到下一条曲线到方法，*如果跳转成功，则返回 true， 如果跳转失败，则返回 false。*

如下，我们创建了一个 Path 并使其中包含了两个闭合的曲线，内部的边长是200，外面的边长是400，现在我们使用 PathMeasure 分别测量两条曲线的总长度。

![](/picture/pathmeasure-06.jpg)

代码如下：

```java
canvas.translate(mViewWidth / 2, mViewHeight / 2);      // 平移坐标系

Path path = new Path();

path.addRect(-100, -100, 100, 100, Path.Direction.CW);  // 添加小矩形
path.addRect(-200, -200, 200, 200, Path.Direction.CW);  // 添加大矩形

canvas.drawPath(path,mDeafultPaint);                    // 绘制 Path

PathMeasure measure = new PathMeasure(path, false);     // 将Path与PathMeasure关联

float len1 = measure.getLength();                       // 获得第一条路径的长度

measure.nextContour();                                  // 跳转到下一条路径

float len2 = measure.getLength();                       // 获得第二条路径的长度

Log.i("LEN","len1="+len1);                              // 输出两条路径的长度
Log.i("LEN","len2="+len2);

log输出结果:
com.gcssloop.canvas I/LEN: len1=800.0
com.gcssloop.canvas I/LEN: len2=1600.0
```

通过测试，我们可以得到以下内容：

- 1.曲线的顺序与 Path 中添加的顺序有关。
- 2.getLength 获取到到是当前一条曲线分长度，而不是整个 Path 的长度。
- 3.getLength 等方法是针对当前的曲线(其它方法请自行验证)。

#### 5.getPosTan

这个方法是用于得到路径上某一长度的位置以及该位置的正切值：

```
boolean getPosTan (float distance, float[] pos, float[] tan)
```

方法各个参数释义：

| 参数            | 作用                 | 备注                                                         |
| --------------- | -------------------- | ------------------------------------------------------------ |
| 返回值(boolean) | 判断获取是否成功     | true表示成功，数据会存入 pos 和 tan 中， false 表示失败，pos 和 tan 不会改变 |
| distance        | 距离 Path 起点的长度 | 取值范围: 0 <= distance <= getLength                         |
| pos             | 该点的坐标值         | 当前点在画布上的位置，有两个数值，分别为x，y坐标。           |
| tan             | 该点的正切值         | 当前点在曲线上的方向，使用 Math.atan2(tan[1], tan[0]) 获取到正切角的弧度值。 |

这个方法也不难理解，除了其中 `tan` 这个东东，这个东西是干什么的呢？

`tan` 是用来判断 Path 上趋势的，即在这个位置上曲线的走向，请看下图示例，注意箭头的方向:

![](/picture/pathmeasure-07.gif)

可以看到上图中箭头在沿着Path运动时，方向始终与Path走向保持一致，保持方向主要就是依靠`tan`  。

```java
private float currentValue = 0;     // 用于纪录当前的位置,取值范围[0,1]映射Path的整个长度

private float[] pos;                // 当前点的实际位置
private float[] tan;                // 当前点的tangent值,用于计算图片所需旋转的角度
private Bitmap mBitmap;             // 箭头图片
private Matrix mMatrix;             // 矩阵,用于对图片进行一些操作
```

```java
private void init(Context context) {
    pos = new float[2];
    tan = new float[2];
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inSampleSize = 2;       // 缩放图片
    mBitmap = BitmapFactory.decodeResource(context.getResources(), R.drawable.arrow, options);
    mMatrix = new Matrix();
}
```

具体绘制：

```java
canvas.translate(mViewWidth / 2, mViewHeight / 2);      // 平移坐标系

Path path = new Path();                                 // 创建 Path

path.addCircle(0, 0, 200, Path.Direction.CW);           // 添加一个圆形

PathMeasure measure = new PathMeasure(path, false);     // 创建 PathMeasure

currentValue += 0.005;                                  // 计算当前的位置在总长度上的比例[0,1]
if (currentValue >= 1) {
  currentValue = 0;
}

measure.getPosTan(measure.getLength() * currentValue, pos, tan);        // 获取当前位置的坐标以及趋势

mMatrix.reset();                                                        // 重置Matrix
float degrees = (float) (Math.atan2(tan[1], tan[0]) * 180.0 / Math.PI); // 计算图片旋转角度

mMatrix.postRotate(degrees, mBitmap.getWidth() / 2, mBitmap.getHeight() / 2);   // 旋转图片
mMatrix.postTranslate(pos[0] - mBitmap.getWidth() / 2, pos[1] - mBitmap.getHeight() / 2);   // 将图片绘制中心调整到与当前点重合

canvas.drawPath(path, mDeafultPaint);                                   // 绘制 Path
canvas.drawBitmap(mBitmap, mMatrix, mDeafultPaint);                     // 绘制箭头

invalidate();                
```

**核心要点:**

> - 1.**通过 tan 得值计算出图片旋转的角度**，tan 是 tangent 的缩写，即中学中常见的正切， 其中tan[0]是邻边边长，tan[1]是对边边长，而Math中 `atan2` 方法是根据正切是数值计算出该角度的大小,得到的单位是弧度(取值范围是 -pi 到 pi)，所以上面又将弧度转为了角度。
> - 2.**通过 Matrix 来设置图片对旋转角度和位移**，这里使用的方法与前面讲解过对 canvas操作 有些类似，对于 `Matrix` 会在后面专一进行讲解，敬请期待。
> - 3.**页面刷新**，页面刷新此处是在 onDraw 里面调用了 invalidate 方法来保持界面不断刷新，但并不提倡这么做，正确对做法应该是使用 线程 或者 ValueAnimator 来控制界面的刷新，关于控制页面刷新这一部分会在后续的 动画部分 详细讲解，同样敬请期待。

关于`tan`这个参数有很多魔法师不理解，特此拉出来详述一下，`tan` 在数学中被称为正切，在直角三角形中，一个锐角的**正切**定义为它的对边(Opposite side)与邻边(Adjacent side)的比值(来自维基百科)：

![](/picture/pathmeasure-08.jpg)

我们此处用 `tan` 来描述 Path 上某一点的切线方向，**主要用了两个数值 tan[0] 和 tan[1] 来描述这个切线的方向(切线方向与x轴夹角)** ，看上面公式可知 `tan` 既可以用 `对边／邻边` 来表述，也可以用 `sin／cos` 来表述，此处用两种理解方式均可以(**注意下面等价关系**):

> **tan[0] = cos = 邻边(单位圆x坐标)**
> **tan[1] = sin = 对边(单位圆y坐标)**

**以 sin／cos理解:**

![](/picture/pathmeasure-09.jpg)

在圆上最右侧点的切线方向向下(动图中小飞机朝向和切线朝向一致)，切线角度为90度.
sin90 = 1，cos90 = 0
tan[0] = cos = 0
tan[1] = sin = 1

**以 对边／邻边 理解(单位圆上坐标):**

按照这种理解方式需要借助一个单位圆，单位圆上任意一点到圆心到距离均为 1，以下图30度为例：

![](/picture/pathmeasure-10.jpg)

tan30 = 对边／邻边 = AB／OA = B点y坐标／B点x坐标

> **另外根据单位圆性质同样可以证得:**
> sin30 = 对边／斜边 = AB／OB = AB = B点y坐标 (单位圆边上任意一点距离圆心距离均为1，故OB = 1)
> cos30 = 邻边／斜边 = OA／OB = OA = B点x坐标
>
> **化为通用公式即为:**
> sin = 该角度在单位圆上对应点的y坐标
> cos = 该角度在单位圆上对应点的x坐标
>
> 即 tan = sin／cos = y／x
> tan[0] = x
> tan[1] = y
>
> 另外注意，这个单位圆与小飞机路径没有半毛钱关系，例如上一个例子中的90度切线，不要在单位圆上找对应位置，**要找对应角度的位置，90度对应的位置是(0，1)**，所以:
> tan[0] = x = 0
> tan[1] = y = 1
>
> 其实绕来绕去全是等价的 (╯°Д°)╯︵ ┻━┻

**PS: 使用 Math.atan2(tan[1], tan[0]) 将 tan 转化为角(单位为弧度)的时候要注意参数顺序。**

### 6.getMatrix

这个方法是用于得到路径上某一长度的位置以及该位置的正切值的矩阵：

```
boolean getMatrix (float distance, Matrix matrix, int flags)
```

方法各个参数释义：

| 参数            | 作用                         | 备注                                                         |
| --------------- | ---------------------------- | ------------------------------------------------------------ |
| 返回值(boolean) | 判断获取是否成功             | true表示成功，数据会存入matrix中，false 失败，matrix内容不会改变 |
| distance        | 距离 Path 起点的长度         | 取值范围: 0 <= distance <= getLength                         |
| matrix          | 根据 falgs 封装好的matrix    | 会根据 flags 的设置而存入不同的内容                          |
| flags           | 规定哪些内容会存入到matrix中 | 可选择 POSITION_MATRIX_FLAG(位置) ANGENT_MATRIX_FLAG(正切)   |

其实这个方法就相当于我们在前一个例子中封装 `matrix` 的过程由 `getMatrix` 替我们做了，我们可以直接得到一个封装好到 `matrix`，岂不快哉。

但是我们看到最后到 `flags` 选项可以选择 `位置` 或者 `正切` ,如果我们两个选项都想选择怎么办？

如果两个选项都想选择，可以将两个选项之间用 `|` 连接起来，如下：

```
measure.getMatrix(distance, matrix, PathMeasure.TANGENT_MATRIX_FLAG | PathMeasure.POSITION_MATRIX_FLAG);
```

我们可以将上面都例子中 `getPosTan` 替换为 `getMatrix`， 看看是不是会显得简单很多:

具体绘制:

```
Path path = new Path();                                 // 创建 Path

path.addCircle(0, 0, 200, Path.Direction.CW);           // 添加一个圆形

PathMeasure measure = new PathMeasure(path, false);     // 创建 PathMeasure

currentValue += 0.005;                                  // 计算当前的位置在总长度上的比例[0,1]
if (currentValue >= 1) {
    currentValue = 0;
}

// 获取当前位置的坐标以及趋势的矩阵
measure.getMatrix(measure.getLength() * currentValue, mMatrix, PathMeasure.TANGENT_MATRIX_FLAG | PathMeasure.POSITION_MATRIX_FLAG);

mMatrix.preTranslate(-mBitmap.getWidth() / 2, -mBitmap.getHeight() / 2);   // <-- 将图片绘制中心调整到与当前点重合(注意:此处是前乘pre)

canvas.drawPath(path, mDeafultPaint);                                   // 绘制 Path
canvas.drawBitmap(mBitmap, mMatrix, mDeafultPaint);                     // 绘制箭头

invalidate();                                                           // 重绘页面
```

> 由于此处代码运行结果与上面一样，便不再贴图片了，请参照上面一个示例的效果图。

可以看到使用 getMatrix 方法的确可以节省一些代码，不过这里依旧需要注意一些内容:

- 1.对 `matrix` 的操作必须要在 `getMatrix` 之后进行，否则会被 `getMatrix` 重置而导致无效。
- 2.矩阵对旋转角度默认为图片的左上角，我们此处需要使用 `preTranslate` 调整为图片中心。
- 3.pre(矩阵前乘) 与 post(矩阵后乘) 的区别，此处请等待后续的文章或者自行搜索。

# Path & SVG

我们知道，用Path可以创建出各种个样的图形，但如果图形过于复杂时，用代码写就不现实了，不仅麻烦，而且容易出错，所以在绘制复杂的图形时我们一般是将 SVG 图像转换为 Path。

**你说什么时SVG？**

SVG是一种矢量图，内部用的是XML格式化存储方式存储操作和数据，你完全可以将SVG看作是Path的各项操作简化后书写的存储格式。

Path和SVG结合通常能诞生出一些奇妙的东西，如下：

![](/picture/pathmeasure-10.gif)![](/picture/pathmeasure-11.gif)

> 图片来自这个开源库：[PathView](https://github.com/geftimov/android-pathview)
>
> SVG转Path的解析可以用这个 [AndroidSVG](https://bigbadaboom.github.io/androidsvg/)

# Path使用技巧

先看一个效果图，先分析一下实现过程

![](/picture/pathmesaure-01.gif)

这个是一个搜索的动效图，通过分析可以得到其应该有四种状态，

| 状态     | 概述                                         |
| -------- | -------------------------------------------- |
| 初始状态 | 初始状态，没有任何动效，只显示一个搜索标志 🔍 |
| 准备搜索 | 放大镜图标逐渐变化为一个点                   |
| 正在搜索 | 围绕这一个圆环运动，并且线段长度会周期性变化 |
| 准备结束 | 从一个点逐渐变化成为放大镜图标               |

这些状态是有序转换的，转换流程以及转换条件如下：

> 其中 `正在搜索` 这个状态持续时间长度是不确定的，在没有搜索`完成`之前，应该一直处于搜索状态。

![](/picture/pathend-12.jpg)

**Path划分**

为了制作方便，为此整个动效用了两个Path，一个是中间放大镜，另一个则是外侧的圆环，将两者全部画出来是这个样子的。

![](/picture/pathend-13.jpg)

其中Path的走向需要把握好，如下（只是一个放大镜)

![](/picture/pathend-14.jpg)

其中图形上面的点可以用PathMeasure测量，无需计算。

**动画状态与时间关联**

此处使用的是 ValueAnimator，它可以将一段时间映射到一段数值上，随着时间变化不断的更新数值，并且可以使用插值器控制数值变化规律(此处使用的是默认插值器)。

#### 具体绘制

绘制部分是根据当前状态以及从`ValueAnimator`获得的数值来截取Path中合适的部分绘制出来。

源码如下：

```java
public class SearchView extends View {

    // 画笔
    private Paint mPaint;

    // View 宽高
    private int mViewWidth;
    private int mViewHeight;

    public SearchView(Context context) {
        this(context,null);
    }

    public SearchView(Context context, AttributeSet attrs) {
        super(context, attrs);
        initAll();
    }

    public void initAll() {

        initPaint();

        initPath();

        initListener();

        initHandler();

        initAnimator();

        // 进入开始动画
        mCurrentState = State.STARTING;
        mStartingAnimator.start();

    }

    // 这个视图拥有的状态
    public static enum State {
        NONE,
        STARTING,
        SEARCHING,
        ENDING
    }

    // 当前的状态(非常重要)
    private State mCurrentState = State.NONE;

    // 放大镜与外部圆环
    private Path path_srarch;
    private Path path_circle;

    // 测量Path 并截取部分的工具
    private PathMeasure mMeasure;

    // 默认的动效周期 2s
    private int defaultDuration = 2000;

    // 控制各个过程的动画
    private ValueAnimator mStartingAnimator;
    private ValueAnimator mSearchingAnimator;
    private ValueAnimator mEndingAnimator;

    // 动画数值(用于控制动画状态,因为同一时间内只允许有一种状态出现,具体数值处理取决于当前状态)
    private float mAnimatorValue = 0;

    // 动效过程监听器
    private ValueAnimator.AnimatorUpdateListener mUpdateListener;
    private Animator.AnimatorListener mAnimatorListener;

    // 用于控制动画状态转换
    private Handler mAnimatorHandler;

    // 判断是否已经搜索结束
    private boolean isOver = false;

    private int count = 0;



    private void initPaint() {
        mPaint = new Paint();
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setColor(Color.WHITE);
        mPaint.setStrokeWidth(15);
        mPaint.setStrokeCap(Paint.Cap.ROUND);
        mPaint.setAntiAlias(true);
    }

    private void initPath() {
        path_srarch = new Path();
        path_circle = new Path();

        mMeasure = new PathMeasure();

        // 注意,不要到360度,否则内部会自动优化,测量不能取到需要的数值
        RectF oval1 = new RectF(-50, -50, 50, 50);          // 放大镜圆环
        path_srarch.addArc(oval1, 45, 359.9f);

        RectF oval2 = new RectF(-100, -100, 100, 100);      // 外部圆环
        path_circle.addArc(oval2, 45, -359.9f);

        float[] pos = new float[2];

        mMeasure.setPath(path_circle, false);               // 放大镜把手的位置
        mMeasure.getPosTan(0, pos, null);

        path_srarch.lineTo(pos[0], pos[1]);                 // 放大镜把手

        Log.i("TAG", "pos=" + pos[0] + ":" + pos[1]);
    }

    private void initListener() {
        mUpdateListener = new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                mAnimatorValue = (float) animation.getAnimatedValue();
                invalidate();
            }
        };

        mAnimatorListener = new Animator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animation) {

            }

            @Override
            public void onAnimationEnd(Animator animation) {
                // getHandle发消息通知动画状态更新
                mAnimatorHandler.sendEmptyMessage(0);
            }

            @Override
            public void onAnimationCancel(Animator animation) {

            }

            @Override
            public void onAnimationRepeat(Animator animation) {

            }
        };
    }

    private void initHandler() {
        mAnimatorHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                switch (mCurrentState) {
                    case STARTING:
                        // 从开始动画转换好搜索动画
                        isOver = false;
                        mCurrentState = State.SEARCHING;
                        mStartingAnimator.removeAllListeners();
                        mSearchingAnimator.start();
                        break;
                    case SEARCHING:
                        if (!isOver) {  // 如果搜索未结束 则继续执行搜索动画
                            mSearchingAnimator.start();
                            Log.e("Update", "RESTART");

                            count++;
                            if (count>2){       // count大于2则进入结束状态
                                isOver = true;
                            }
                        } else {        // 如果搜索已经结束 则进入结束动画
                            mCurrentState = State.ENDING;
                            mEndingAnimator.start();
                        }
                        break;
                    case ENDING:
                        // 从结束动画转变为无状态
                        mCurrentState = State.NONE;
                        break;
                }
            }
        };
    }

    private void initAnimator() {
        mStartingAnimator = ValueAnimator.ofFloat(0, 1).setDuration(defaultDuration);
        mSearchingAnimator = ValueAnimator.ofFloat(0, 1).setDuration(defaultDuration);
        mEndingAnimator = ValueAnimator.ofFloat(1, 0).setDuration(defaultDuration);

        mStartingAnimator.addUpdateListener(mUpdateListener);
        mSearchingAnimator.addUpdateListener(mUpdateListener);
        mEndingAnimator.addUpdateListener(mUpdateListener);

        mStartingAnimator.addListener(mAnimatorListener);
        mSearchingAnimator.addListener(mAnimatorListener);
        mEndingAnimator.addListener(mAnimatorListener);
    }


    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        mViewWidth = w;
        mViewHeight = h;
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        drawSearch(canvas);
    }

    private void drawSearch(Canvas canvas) {

        mPaint.setColor(Color.WHITE);


        canvas.translate(mViewWidth / 2, mViewHeight / 2);

        canvas.drawColor(Color.parseColor("#0082D7"));

        switch (mCurrentState) {
            case NONE:
                canvas.drawPath(path_srarch, mPaint);
                break;
            case STARTING:
                mMeasure.setPath(path_srarch, false);
                Path dst = new Path();
                mMeasure.getSegment(mMeasure.getLength() * mAnimatorValue, mMeasure.getLength(), dst, true);
                canvas.drawPath(dst, mPaint);
                break;
            case SEARCHING:
                mMeasure.setPath(path_circle, false);
                Path dst2 = new Path();
                float stop = mMeasure.getLength() * mAnimatorValue;
                float start = (float) (stop - ((0.5 - Math.abs(mAnimatorValue - 0.5)) * 200f));
                mMeasure.getSegment(start, stop, dst2, true);
                canvas.drawPath(dst2, mPaint);
                break;
            case ENDING:
                mMeasure.setPath(path_srarch, false);
                Path dst3 = new Path();
                mMeasure.getSegment(mMeasure.getLength() * mAnimatorValue, mMeasure.getLength(), dst3, true);
                canvas.drawPath(dst3, mPaint);
                break;
        }
    }
}
```

