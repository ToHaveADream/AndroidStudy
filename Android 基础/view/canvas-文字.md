### 1、绘制图片

绘制图片有两种方法，drawPicture(矢量图)和drawBitMap(位图)，接下来我们一一了解。

1）drawPicture

使用drawPicture之前请关闭硬加速

**在AndroidMenifest文件中application节点下添加 android:hardwareAcclerated="false"以关闭整个应用的硬件加速**

首先看下官方文档对Picture的解释：

> *A Picture records drawing calls (via the canvas returned by beginRecording) and can then play them back into Canvas (via draw(Canvas) or drawPicture(Picture)).For most content (e.g. text, lines, rectangles), drawing a sequence from a picture can be faster than the equivalent API calls, since the picture performs its playback without incurring any method-call overhead.*

我们把Canvas绘制的点、线、矩形等诸多操作用Picture记录下来，下次需要的时候拿来就能用，使用Picture相比于再次调用绘图API，开销是比较小的，就是对于重复的操作更加的省时省力

> 可以把Picture看作是一个录制Canvas操作的录像机

了解一下Picture的相关方法：

| 相关方法                                                    | 简介                                                         |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| public int getWidth ()                                      | 获取宽度                                                     |
| public int getHeight ()                                     | 获取高度                                                     |
| public Canvas beginRecording (int width, int height)        | 开始录制 (返回一个Canvas，在Canvas中所有的绘制都会存储在Picture中) |
| public void endRecording ()                                 | 结束录制                                                     |
| public void draw (Canvas canvas)                            | 将Picture中内容绘制到Canvas中                                |
| public static Picture createFromStream (InputStream stream) | (已废弃)通过输入流创建一个Picture                            |
| public void writeToStream (OutputStream stream)             | (已废弃)将Picture中内容写出到输出流中**                      |

**很明显，beginRecording 和 endRecording是成对使用的，一个开始录制，一个是结束录制，两者之前的操作将会存储在Picture中**。

使用用例：

**准备工作**：

录制内容，即将一些Canvas操作用Picture存储起来，录制的内容是不会直接显示在屏幕上的，知识存储起来了而已。

```java
// 1.创建Picture
private Picture mPicture = new Picture();

---------------------------------------------------------------

// 2.录制内容方法
private void recording() {
    // 开始录制 (接收返回值Canvas)
    Canvas canvas = mPicture.beginRecording(500, 500);
    // 创建一个画笔
    Paint paint = new Paint();
    paint.setColor(Color.BLUE);
    paint.setStyle(Paint.Style.FILL);

    // 在Canvas中具体操作
    // 位移
    canvas.translate(250,250);
    // 绘制一个圆
    canvas.drawCircle(0,0,100,paint);

    mPicture.endRecording();
}

---------------------------------------------------------------

// 3.在使用前调用(我在构造函数中调用了)
  public Canvas3(Context context, AttributeSet attrs) {
    super(context, attrs);
    
    recording();    // 调用录制
}
```

**具体使用**：

Picture虽然方法就那么几个，但是具体使用起来还是分很多情况的，由于录制的内容不会直接显示，就像存储的视频不点击不会自动播放一样，同样，想要将Picture中的内容显示出来就需要手动调用播放（绘制），将Picture中的内容绘制出来可以有一下几种方法：

| 序号 | 简介                                                         |
| ---- | ------------------------------------------------------------ |
| 1    | 使用Picture提供的draw方法绘制                                |
| 2    | 使用Canvas提供的drawPicture方法绘制                          |
| 3    | 将Picture包装成为PictureDrawable,使用PictureDrawable的draw方法绘制 |

以上几种方法主要区别：

| 主要区别           | 分类                         | 简介                                                 |
| ------------------ | ---------------------------- | ---------------------------------------------------- |
| 是否对Canvas有影响 | 1、有影响   2 3不影响        | 此处绘制完成后是否会影响Canvas的状态（matrix clip)等 |
| 可操作性强弱       | 1 可操作性较弱 2 3可操作性强 | 此处的可操作性可以简单理解为对绘制结果可控程度       |

1、使用Picture提供的draw方法

```java
// 将Picture中的内容绘制在Canvas上
mPicture.draw(canvas);  
```

> 这种方法在低版本上面绘制可能会影响Canvas状态，所以这种方法一般不会使用

2、使用Canvas提供的drawPirture方法绘制

drawPicture有三种方法

```java
public void drawPicture (Picture picture)

public void drawPicture (Picture picture, Rect dst)

public void drawPicture (Picture picture, RectF dst)
```

和使用Picture提供的draw方法不同，Canvas的drawPicture不会影响Canvas的状态

简单示例：

```java
canvas.drawPicture(mPicture,new RectF(0,0,mPicture.getWidth(),200));
```

3.将Picture封装为PictureDrawable，使用PictureDrawable的draw方法绘制

```java
// 包装成为Drawable
PictureDrawable drawable = new PictureDrawable(mPicture);
// 设置绘制区域 -- 注意此处所绘制的实际内容不会缩放
drawable.setBounds(0,0,250,mPicture.getHeight());
// 绘制
drawable.draw(canvas);
```

> setBounds是设置在画布上的绘制区域，并非根据该区域进行缩放，也不是裁剪Picture。每次都从Picture的左上角开始绘制

2）drawBitmap

首先，如何获取一个Bitmap：

| 序号 | 获取方式               | 备注 |
| ---- | ---------------------- | ---- |
| 1    | 通过Bitmap创建         |      |
| 2    | 通过BitmapDrawable获取 |      |
| 3    | 通过BitmapFactory获取  |      |

