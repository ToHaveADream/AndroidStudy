# Android 动画那些事儿---视图动画（View Animation）

![](/picture/1.image)

## 一、视图动画（View Animation）

视图动画又称为 View 动画，视图动画可以分为帧动画（Frame Animation）和 补间动画（Tween Animation）两类。本书中将详细介绍这两种动画。

### 1、帧动画（Frame Animation）

帧动画顾名思义就是逐帧播放的动画，帧动画的实现非常简单，只需要通过编写 drawable 的 xml 和 几行简短的代码即可实现：

```xml
<?xml version="1.0" encoding="utf-8"?>
<animation-list
	xmlns:android="http://schemas.android.com/apk/res/android"
  	android:oneshot="false">
    <item android:drawable="@drawable/progress_1" android:duration="200"/>
    <item android:drawable="@drawable/progress_2" android:duration="200"/>
    <item android:drawable="@drawable/progress_3" android:duration="200"/>
    <item android:drawable="@drawable/progress_4" android:duration="200"/>
    <item android:drawable="@drawable/progress_5" android:duration="200"/>
    <item android:drawable="@drawable/progress_6" android:duration="200"/>
    <item android:drawable="@drawable/progress_7" android:duration="200"/>
    <item android:drawable="@drawable/progress_8" android:duration="200"/>
</animation-list>
```

上面的xml 中添加了8帧图片，Android 会将其作为一个 drawable 文件，因此我们可以直接在 ImageView 的 background 或 src 中引用。接下来就可以通过 ImageView 获取到该 drawable 并实现动画的控制。代码如下：

```java
        ImageView imageView = mDialogView.findViewById(R.id.loadingImageView);
        //	这里要注意ImageView设置的时background还是src,如果是background则调用getBackground，否则调用getDrawable().
        animationDrawable = (AnimationDrawable) imageView.getBackground();
        //  执行帧动画
        animationDrawable.start();
        //  停止帧动画播放
        animationDrawable.stop();
```

帧动画可以说是相当简单，但是不推荐过多的使用帧动画。因为帧动画是多张图片拼合而成。加载图片会占用相当多的资源，因此如果项目中出现大量的帧动画势必会影响到 App 的性能。

## 2、补间动画 （Tween Animation）

从上面的思维导图可以看到补间动画可以分为四类：AlphaAnimation、RotateAnimation、TranslateAnimation以及ScaleAnimation。那么这四个类有什么关系吗？我们不妨看看 Animation 类，Animation 是一个抽象类，下图是 Animation 的继承结构：

![](/picture/2.image)

可以发现我们提到的四个动画类均继承自 Animation。既然如此，那么在 Animation 中必然会提供很多动画类所共有的特性，因此看下提供的 API：

| ethod                                   | Method description                                           |
| --------------------------------------- | ------------------------------------------------------------ |
| setDuration(long)                       | How long this animation should last                          |
| setStartTime(long)                      | When this animation should start                             |
| start()                                 | Convenience method to start the animation the first time     |
| startNow()                              | Convenience method to start the animation at the current time |
| setRepeatMode(int)                      | Defines what this animation should do when it reaches the end. |
| setRepeatCount(int)                     | Sets how many times the animation should be repeated         |
| setFillEnabled(boolean)                 | If fillEnabled is true, the animation will apply the value of fillBefore. Otherwise, fillBefore is ignored and the animation transformation is always applied until the animation ends. |
| setFillBefore(boolean)                  | If fillBefore is true, this animation will apply its transformation before the start time of the animation. |
| setFillAfter(boolean)                   | If fillAfter is true, the transformation that this animation performed will persist when it is finished. Defaults to false if not set. |
| setInterpolator(Interpolator            | Sets the acceleration curve for this animation. Defaults to a linear interpolation. |
| setStartOffset(long)                    | When this animation should start relative to the start time. |
| setZAdjustment(int                      | Set the Z ordering mode to use while running the animation.  |
| setAnimationListener(AnimationListener) | Binds an animation listener to this animation. The animation listener is notified of animation events such as the end of the animation or the  repetition of the animation. |

我们实现动画效果的时候既可以通过xml 来实现也可以通过代码来实现。本届中只讨论通过代码来实现四种补间动画效果。

- AlphaAnimation（透明度动画）

  AlphaAnimation 即为透明度动画，它可以改变 View 的透明度从而达到一个透明渐变的动画效果，通过代码来看：

  ```java
  mImageView.startAnimation(getAlphaAnimation());
  
   private AlphaAnimation getAlphaAnimation() {
          AlphaAnimation alphaAnimation = new AlphaAnimation(0, 1);
          alphaAnimation.setDuration(2000);
          return alphaAnimation;
      }
  ```

  上面的代码中实例化了一个透明度从 0 - 1 的 AlphaAnimation，并设置了 2 s 的持续时间，然后通过 ImageView 的 startAnimation 来开启透明效果。效果如下：

  ![](/picture/3.image)

  AlphaAnimation 类中几乎没有自己的 API，仅仅是在构造方法中传入了两个透明值：

  | Method                                         | Method description                                           |
  | ---------------------------------------------- | ------------------------------------------------------------ |
  | AlphaAnimation(float fromAlpha, float toAlpha) | fromAlpha： Starting alpha value for the animation, where 1.0 means  fully opaque and 0.0 means fully transparent.        toAlhpa:Ending alpha value for the animation. |

- RotateAnimation （旋转动画）

  从名字上看这是一个旋转相关的动画，来认识一下 RotateAnimation。

  ```java
  float pivotX= (mImageView.getRight() - mImageView.getLeft()) / 2f;
  float pivotY=(mImageView.getBottom() - mImageView.getTop()) / 2f;
  mImageView.startAnimation(getRotateAnimation(0));
  private RotateAnimation getRotateAnimation(int repeatCount) {
          RotateAnimation rotateAnimation = new RotateAnimation(0, 360, pivotX, pivotY);
          rotateAnimation.setDuration(2000);
          rotateAnimation.setRepeatCount(repeatCount);
          return rotateAnimation;
      }
  ```

  代码中先计算出风车图片的中心坐标，接着实例化了一个 RotateAnimation 动画，其构造参数有四个，表示绕着中心左边从0到 360度，效果图如下：

  ![](/picture/4.image)

  同样，Rotate Animation除了构造方法外也几乎没有其他 API：

  | Method                                                       | Method description                                           |
  | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | RotateAnimation(float fromDegrees, float toDegrees)          | Constructor to use when building a RotateAnimation from code. Default pivotX/pivotY point is (0,0). |
  | RotateAnimation(float fromDegrees, float toDegrees, float pivotX, float pivotY | Constructor to use when building a RotateAnimation from code. point is (pivotX,pivotY  ). |

- Translate Animation （位移动画）

  该动画可以给 view 增加位移动画，举一个例子：

  ```java
  mImageView.startAnimation(getTranslateAnimation());
  private TranslateAnimation getTranslateAnimation() {
          TranslateAnimation translateAnimation = new TranslateAnimation(-200, 0, -100, 0);
          translateAnimation.setDuration(2000)；
          return translateAnimation;
      }
  ```

  ![](/picture/5.image)

| Method                                                       | Method description                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| TranslateAnimation(float fromXDelta, float toXDelta, float fromYDelta, float toYDelta) | onstructor to use when building a TranslateAnimation from code |

- Scale Animation（缩放动画）

  该动画为 view 提供了缩放功能，代码如下：

  ```java
  mImageView.startAnimation(getScaleAnimation());
  private ScaleAnimation getScaleAnimation() {
          ScaleAnimation scaleAnimation = new ScaleAnimation(0.2f, 1, 0.2f, 1, pivotX, pivotY);
          scaleAnimation.setDuration(2000);
          return scaleAnimation;
      }
  ```

  以图片为中心轴，从0.2 缩放到 1的动画，如下：

  ![](/picture/6.image)

| Method                                                       | Method description                                          |
| ------------------------------------------------------------ | ----------------------------------------------------------- |
| ScaleAnimation(float fromX, float toX, float fromY, float toY) | Constructor to use when building a ScaleAnimation from code |

- Animation Set(集合动画)

  在 Animation 的继承结构中有一个 AnimationSet 的子类，这是一个动画的集合类。看源代码可以发现其实就是在内部封装了一个 ArrayList 来存放上面的几种动画 ，从而达到一个叠加动画的效果，看一下如何使用：

  ```java
  mImageView.startAnimation(getAnimationSet());
  private AnimationSet getAnimationSet() {
          AnimationSet animationSet = new AnimationSet(true);
          animationSet.setDuration(2000);
          animationSet.addAnimation(getRotateAnimation(0));
          animationSet.addAnimation(getScaleAnimation());
          animationSet.addAnimation(getAlphaAnimation());
          animationSet.addAnimation(getTranslateAnimation());
          return animationSet;
      }
  ```

  可以看到上诉代码中把四种类型的动画都添加到了 AnimationSet 中，从而实现了一个组合的动画效果，如下图所示：

  ![](/picture/7.image)

- 动画的状态监听

  我们在说 Animation 提供的 API 时候已经出现了 setAnimationListener的方法，这个方法是监听动画的状态的。它提供了动画开始、结束以及重复的回调，从而让我们更方便的控制动画，代码如下：

  ```java
   animation.setAnimationListener(new Animation.AnimationListener() {
              @Override
              public void onAnimationStart(Animation animation) {
                  Log.e(tag, "onAnimationStart");
              }
  
              @Override
              public void onAnimationEnd(Animation animation) {
                  Log.e(tag, "onAnimationEnd");
              }
  
              @Override
              public void onAnimationRepeat(Animation animation) {
                  Log.e(tag, "onAnimationRepeat");
              }
          });
  ```

虽然补间动画很轻大，可以实现比较炫酷的效果，但是却有一个先天不足的缺陷，不具有交互性，其改变的只是显示效果，其响应事件还是在原来的位置。因此，视图动画仅适用于布局没有交互性的情况，而对于带有交互性的动画却显得无能为力了。















































































