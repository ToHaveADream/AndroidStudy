# Android 动画那些事儿--属性动画（Propetry Animation）

上篇文章详细讲解了视图动画，也提到了视图动画存在的先天不足，即补间动画不具有交互性。动画改变的只是显示效果，其响应事件却还是在原来的位置。属性动画相比视图动画具有更加强大的功能，不仅弥补了视图动画的先天不足，而且属性动画不仅仅可以作用于 View，而是作用于任意对象。接下来，我们结合Android 动画的结构图来看。

![](picture/1.image)

从图中可以看到，属性动画有ValueAnimator 和 AnimatorSet 两个类，这两个类均继承自 android.animation.Animator 类，从名字上来看 ValueAnimator 叫做数值动画，是一个动画类，它有两个子类分别是 ObjectAnimator 和 TimeAnimator；而 AnimatorSet 从名字上来看它是一个属性动画的集合类。因此我们就自上而下从 Animator 这个类开始逐步了解 Android 的属性动画。

## 1、Animator类

Animator 类是一个抽象类，是 ValueAnimator 和 AnimatorSet 的父类，因此这个方法中一定声明了许多公共的方法：

| Method                                      | Method description                                           |
| ------------------------------------------- | ------------------------------------------------------------ |
| void start()                                | Starts this animation.                                       |
| void  cancel()                              | Cancels the animation.                                       |
| void end()                                  | Ends the animation.                                          |
| void pause()                                | Pauses a running animation.                                  |
| void resume()                               | Resumes a paused animation                                   |
| void addListener(AnimatorListener listener) | Adds a listener to the set of listeners that are sent events through the life of ananimation, such as start, repeat, and end. |
| setInterpolator(TimeInterpolator value)     | The time interpolator used in calculating the elapsed fraction of theanimation. |

Animator 中常用的方法大概也就折磨多，因为 Animator是一个抽象类，因此这个类中的方法多是抽象方法或者空方法，具体的实现都在子类中，并且从方法的名字上就能看懂方法的用途，因此关于这个类没什么要说的，不过最后一个方法 setInterpolator（TimeInterpolator value）可能会有些陌生，这个方法是为动画设置一个插值器，可以去控制动画的速率。

## 2、ValueAnimator 类

ValueAnimator 在属性动画中有着非常重要的作用，它是属性动画的核心。ValueAnimator 本身没有提供任何动画效果，他会接受一个初始值和结束值，在运行的过程中不断的改变数值的大小从而实现动画的变换。我们来看 ValueAnimator 中提供的 API。

| Method                                                       | Method description                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ValueAnimator ofInt(int... values)                           | Constructs and returns a ValueAnimator that animates between int values. |
| ValueAnimator ofFloat(float... values)                       | Constructs and returns a ValueAnimator that animates between float values. |
| ValueAnimator ofArgb(int... values)                          | Constructs and returns a ValueAnimator that animates between color values. |
| ValueAnimator ofPropertyValuesHolder(PropertyValuesHolder... values) | Constructs and returns a ValueAnimator that animates between the values specified in the PropertyValuesHolder objects. |
| ValueAnimator ofObject(TypeEvaluator evaluator, Object... values) | Constructs and returns a ValueAnimator that animates between Object values. |
| void setRepeatCount(int value)                               | Sets how many times the animation should be repeated.        |
| setRepeatMode(@RepeatMode int value)                         | Defines what this animation should do when it reaches the end. |
| addUpdateListener(AnimatorUpdateListener listener)           | Adds a listener to the set of listeners that are sent update events through the life of an animation. |

上面列出了 ValueAnimator 中常用的一些方法，前面我们提到 ValueAnimator其实是一个数值生成器，那么 ValueAnimator 是如何产生一个变化的数值的呢？下面看一个例子：

```java
ValueAnimator valueAnimator = ValueAnimator.ofFloat(0,1);
        valueAnimator.setDuration(200);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                int animatedValue = (int) animation.getAnimatedValue();
                Log.e("ValueAnimator","animatedValue-------"+animatedValue);
            }
        });
        valueAnimator.start();
```

上面的代码初始化了一个从 0-1的数值是 float 类型的 ValueAnimator，我们为其设置了 200 ms 的动画时间，并为其添加了 addUpdateListener 来监听动画更新，并将日志打印出来：

![](picture/8.image)

可以看到在动画的执行过程中控制台打印出了一系列个从0-1平滑过渡的浮点数值。如果传入多个值呢？可以试下如下代码：

```java
ValueAnimator valueAnimator = ValueAnimator.ofFloat(0,1,0);
        valueAnimator.setDuration(200);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                int animatedValue = (int) animation.getAnimatedValue();
                Log.e("ValueAnimator","animatedValue-------"+animatedValue);
            }
        });
        valueAnimator.start();
```

这次我们再 ofFloat 中传入了三个参数，来看控制台输出日志：

![](picture/9.image)

而 ofArgb（int values）这个方法是再 API 26之后才引入的，该方法可以生成一系列的颜色数值，方便我们去做颜色渐变。针对 ofArgb(int ...values)我们可以看一个例子：

```java
        mCircleView = findViewById(R.id.circleView);
        valueAnimator = ValueAnimator.ofArgb(0xffff0000, 0xff00ff00);
        valueAnimator.setDuration(3000);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                mCircleView.setCirCleColor((int) animation.getAnimatedValue());
            }
        });
```

上边代码中初始化了一个从颜色红色到绿色的 Value Animator，并再 addUpdateListener 中为一个自定义的 CircleView 去设置颜色，我们来看效果图：

好了，来看下 ofObject(Object ...values)应该如何理解呢？难不成还能是一个 Object 的变换？不要慌，我们接着看。要说 ofObject（Object ...values) 我们应该先从 TypeEvaluator 开始。TypeEvaluator 是一个接口，官方文档是对他的解释是这样的：

> TypeEvaluator 是一个用在 setEvaluator 方法的接口，它允许开发者在任意属性类型上创建动画，允许开发者提供 Android Animation System 无法理解或使用的 TypeEvaluator

大概意思就是告诉我们这个接口用在 setEvaluator 上，允许我们去自定义 TypeEvaluator。结合例子来看。

```java
public class CircleCenter {
    private float CenterX;
    private float CenterY;

    public CircleCenter(float centerX, float centerY) {
        CenterX = centerX;
        CenterY = centerY;
    }

    public float getCenterX() {
        return CenterX;
    }

    public void setCenterX(float centerX) {
        CenterX = centerX;
    }

    public float getCenterY() {
        return CenterY;
    }

    public void setCenterY(float centerY) {
        CenterY = centerY;
    }
}
```

然后自定义一个 TypeEvaluator

```java
public class CircleCenterEvaluator implements TypeEvaluator<CircleCenter> {
    @Override
    public CircleCenter evaluate(float fraction, CircleCenter startValue, CircleCenter endValue) {
        // x方向匀速移动
        float centerX = startValue.getCenterX() + fraction * (endValue.getCenterX() - startValue.getCenterX());
        // y方向抛物线加速移动
        float centerY = startValue.getCenterY() + fraction * fraction * (endValue.getCenterY() - startValue.getCenterY());
        return new CircleCenter(centerX, centerY);
    }
}
```

接下来我们调用 Object 开启动画，并设置计算出来的圆的圆心坐标：

```
        CircleCenter startValue = new CircleCenter(0, 150);
        CircleCenter endValue = new CircleCenter(ScreenUtils.getScreenWidth(), ScreenUtils.getScreenHeight());
        valueAnimator = ValueAnimator.ofObject(new CircleCenterEvaluator(), startValue, endValue);
        valueAnimator.setDuration(3000);
        valueAnimator.setEvaluator(new CircleCenterEvaluator());
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                CircleCenter circleCenter = (CircleCenter) animation.getAnimatedValue();
                mCircleView.setX(circleCenter.getCenterX());
                mCircleView.setY(circleCenter.getCenterY());
                mCircleView.invalidate();
            }
        });
        valueAnimator.start();

```

开启之后，效果如下：

![](picture/10.image)

我们通过自定义 TypeEvaluator 和 ofObject 方法完成了一个小球的抛物线动画，由此可见 ofObject 的强大之处 了。

## 3、ObjectAnimator

上一节我们认识了 ValueAnimator,通过几个例子我们知道了 ValueAniamtor 又多么的强大。接下来看下 ObjectAnimator 给我们提供的 API：

| Method                                                       | Method description                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ObjectAnimator ofInt(Object target, String propertyName, int... values) | Constructs and returns an ObjectAnimator that animates between int values. |
| ObjectAnimator ofMultiInt(Object target, String propertyName, int[][] values) | Constructs and returns an ObjectAnimator that animates over int values for a multiple parameters setter. |
| ObjectAnimator ofArgb(Object target, String propertyName, int... values) | Constructs and returns an ObjectAnimator that animates between color values. |
| ObjectAnimator ofFloat(Object target, String propertyName, float... values) | Constructs and returns an ObjectAnimator that animates between float values |
| ObjectAnimator ofMultiFloat(Object target, String propertyName, float[][] values) | Constructs and returns an ObjectAnimator that animates over float values for a multiple parameters setter. |
| ObjectAnimator ofObject(Object target, String propertyName,  TypeEvaluator evaluator, Object... values) | Constructs and returns an ObjectAnimator that animates between Object values |

ObjectAnimator 实现透明度动画

```java
        ObjectAnimator animator = ObjectAnimator.ofFloat(mImageView, "alpha", 1f, 0f, 1f);
        animator.setDuration(2000);
        animator.start();
```

效果如图：

![](picture/11.image)



ObjectAnimator 实现旋转动画

```java
        ObjectAnimator animator = ObjectAnimator.ofFloat(mImageView, "rotation", 0f, 360f);
        animator.setDuration(2000);
        animator.start();
```

![](picture/12.image)

ObjectAnimation 实现平移动画

```java
        float translationY = mImageView.getTranslationY();
        ObjectAnimator animator = ObjectAnimator.ofFloat(mImageView, "translationY", translationY, translationY + 400);
        animator.setDuration(2000);
        animator.start();
```

![](picture/13.image)

ObjectAnimator 实现缩放动画

```java
    ObjectAnimator animator = ObjectAnimator.ofFloat(mImageView, "scaleY", 1f, 2f, 1f);
    animator.setDuration(2000);
    animator.start();
```

![](picture/14.image)

ObjectAnimator 如何实现 View 的变换？为啥调用 ofFloat 方法时传入一个“alpha"参数就可以实现 View 透明度渐变了。首先看下 View 的源码，可以看到 View 中提供了众多的 view 变换方法，其中就有 setAlpha 和 setRotation，源码如下：

```
  public void setAlpha(@FloatRange(from=0.0, to=1.0) float alpha) {
        ensureTransformationInfo();
        if (mTransformationInfo.mAlpha != alpha) {
            setAlphaInternal(alpha);
            if (onSetAlpha((int) (alpha * 255))) {
                mPrivateFlags |= PFLAG_ALPHA_SET;
                // subclass is handling alpha - don't optimize rendering cache invalidation
                invalidateParentCaches();
                invalidate(true);
            } else {
                mPrivateFlags &= ~PFLAG_ALPHA_SET;
                invalidateViewProperty(true, false);
                mRenderNode.setAlpha(getFinalAlpha());
            }
        }
    }


 public void setRotation(float rotation) {
        if (rotation != getRotation()) {
            // Double-invalidation is necessary to capture view's old and new areas
            invalidateViewProperty(true, false);
            mRenderNode.setRotation(rotation);
            invalidateViewProperty(false, true);

            invalidateParentIfNeededAndWasQuickRejected();
            notifySubtreeAccessibilityStateChangedIfNeeded();
        }
    }

```

深入看下 ObjectAnimator 的源码：

```java
 public static ObjectAnimator ofFloat(Object target, String propertyName, float... values) {
        ObjectAnimator anim = new ObjectAnimator(target, propertyName);
        anim.setFloatValues(values);
        return anim;
    }
```

propertyName 被传入了 ObjectAnimator 的构造方法中，那么我们跟进 ObjectAnimator 的构造方法：

```java
 private ObjectAnimator(Object target, String propertyName) {
        setTarget(target);
        setPropertyName(propertyName);
    }
```

继续跟进：setPropetryName(propetryName)方法：

```
 public void setPropertyName(@NonNull String propertyName) {
        // mValues could be null if this is being constructed piecemeal. Just record the
        // propertyName to be used later when setValues() is called if so.
        if (mValues != null) {
            PropertyValuesHolder valuesHolder = mValues[0];
            String oldName = valuesHolder.getPropertyName();
            valuesHolder.setPropertyName(propertyName);
            mValuesMap.remove(oldName);
            mValuesMap.put(propertyName, valuesHolder);
        }
        mPropertyName = propertyName;
        // New property/values/target should cause re-initialization prior to starting
        mInitialized = false;
    }

```

搜索 mPropetryName后我们发现：

```java
 private Method getPropertyFunction(Class targetClass, String prefix, Class valueType) {
        // TODO: faster implementation...
        Method returnVal = null;
        String methodName = getMethodName(prefix, mPropertyName);
        Class args[] = null;
        if (valueType == null) {
            try {
                returnVal = targetClass.getMethod(methodName, args);
            } catch (NoSuchMethodException e) {
                
            }
        } else {
          // ... 省略无关代码
        }

 		// ... 省略无关代码
        return returnVal;
    }
```

这个方法中通过 getMethodName(prefix, mPropetryName)获得了一个方法名，并由该方法名通过反射得到这个方法，看下哪里调用了 getPropetryFunction 方法，搜索之后 setupSetterOrGetter 方法中调用该方法，并将返回值赋值给 PropetryValuesHolder 的成员变量 mSetter，最终在 setAnimatedValue 方法中反射调用了这个方法，：

```
 void setAnimatedValue(Object target) {
        if (mProperty != null) {
            mProperty.set(target, getAnimatedValue());
        }
        if (mSetter != null) {
            try {
                mTmpValueArray[0] = getAnimatedValue();
                mSetter.invoke(target, mTmpValueArray);
            } catch (InvocationTargetException e) {
                Log.e("PropertyValuesHolder", e.toString());
            } catch (IllegalAccessException e) {
                Log.e("PropertyValuesHolder", e.toString());
            }
        }
    }
```

那么被反射调用的方法的名字是什么呢？看一下：

```java
getMethodName("set", mPropertyName);

 static String getMethodName(String prefix, String propertyName) {
        if (propertyName == null || propertyName.length() == 0) {
            // shouldn't get here
            return prefix;
        }
        char firstLetter = Character.toUpperCase(propertyName.charAt(0));
        String theRest = propertyName.substring(1);
        return prefix + firstLetter + theRest;
    }
```

此时，一切都明朗了，原来就是通过字符串拼接起来的方法名，prefix即为“set”，而mPropertyName就是我们传进来的那个参数( "alpha")，getMethodName方法中将"scaleY"首字母转成了大写，并最终拼接出了prefix + firstLetter + theRest，即：setScaleY。到这里我们终于明白了为什么在ObjectAnimator中传入一个字符串就可以改变ImageView的变换。

明白了ObjectAnimator如何实现View的变换，那么此时就有了一个问题，既然变换最终是通过set方法实现的，那么如果我们自己添加一个set方法是不是也应该能被调用到呢？猜测应该是可行的吧，心动不如行动，我们不妨来试试看。 首先来改造上述代码中的自定义CircleView，在该方法中添加setProgress方法，并添加一个startAnimate()的方法，最终代码如下：

```java
public class CircleView extends View {
    private Paint mPaint;

    public CircleView(Context context) {
        this(context, null);
    }

    public CircleView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public CircleView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mPaint = new Paint();
        mPaint.setColor(Color.parseColor("#FF0000"));
        mPaint.setAntiAlias(true);
    }

    public void setCirCleColor(int color) {
        mPaint.setColor(color);
        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        int width = getWidth();
        int height = getHeight();
        float radius = Math.min(width, height) / 2f;
        canvas.drawCircle(width / 2f, height / 2f, radius, mPaint);
    }

    public void setProgress(float progress) {
        Log.e("CircleView", "progress---->" + progress);
    }

    public void startAnimate() {
        ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(this, "progress", 0, 1f);
        objectAnimator.setDuration(200);
        objectAnimator.start();
    }
}
```

很棒！当我们在CircleView加了setProgress(float progress)方法后，再通过ObjectAnimator.ofFloat(this, "progress", 0, 1f)去实例化ObjectAnimator，此时编译器并没有报错。接下来我们在Activity中调用startAnimate方法来看控制台是否能打印出progress的日志：

## 4、AnimatorSet

从名字上就可以知道AnimatorSet是一个动画的集合类，它跟补间动画中的AnimationSet有些类似。使用也非常简单，举个例子如下：

```
 ObjectAnimator animator1 = ObjectAnimator.ofFloat(mImageView, "scaleY", 1f, 2f, 1f);
 ObjectAnimator animator2 = ObjectAnimator.ofFloat(mImageView, "rotation", 0f, 360f);
 ObjectAnimator animator3 = ObjectAnimator.ofFloat(mImageView, "alpha", 1f, 0f, 1f);
  AnimatorSet animatorSet = new AnimatorSet();
  animatorSet.setDuration(2000);
  animatorSet.playTogether(animator1, animator2, animator3);
  animatorSet.start();
```

或者可以按照顺序播放：

```java
animatorSet.playSequentially(animator1,animator2,animator3);
```

属性动画中除了AnimatorSet之外还提供了PropertyValuesHolder，这个类上边我们已经提到，它也可以实现同时播放多种动画，效果与AnimatorSet.playTogether()一样，其使用代码如下：

```java
 PropertyValuesHolder alpha = PropertyValuesHolder.ofFloat("alpha", 1f, 0f, 1f);
 PropertyValuesHolder rotation = PropertyValuesHolder.ofFloat("rotation", 0f, 360f);
 PropertyValuesHolder scaleY = PropertyValuesHolder.ofFloat("scaleY", 1f, 2f, 1f);
ObjectAnimator.ofPropertyValuesHolder(mImageView, alpha, rotation, scaleY).setDuration(2000).start();
```

## 5、Animator 的监听事件

Animator动画具有Start、Repeat、End以及Cancel事件。系统为我们提供了这些事件的监听，代码如下：

```java
animator.addListener(new Animator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animation) {
          	      // Animation Start
            }

            @Override
            public void onAnimationEnd(Animator animation) {
 				  // Animation End
            }

            @Override
            public void onAnimationCancel(Animator animation) {
 				  // Animation Cance
            }

            @Override
            public void onAnimationRepeat(Animator animation) {
 			 	 // Animation nRepea
            }
        });
```

如果只想监听某一个事件则可以：

```java
animator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                super.onAnimationEnd(animation);
            }
        });
```

这里AnimatorListenerAdapter是一个抽象类，我们监听时只重写了onAnimationEnd。

到这里关于属性动画的东西已经全部讲解完了，看完相信大家对属性动画一定有了比较深刻的认识。那么下篇文章我们将来讲解控制动画速率的Interpolator。





























