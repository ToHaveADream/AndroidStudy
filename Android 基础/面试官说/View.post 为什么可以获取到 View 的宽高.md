## 面试官：View.post（）为什么能够获取到 View 的宽高？

### 小测试：哪里可以获取到 View 的宽高？

先来一段测试代码：

```java
class MainActivity : BaseLifecycleActivity() {

    private val binding by lazy { ActivityMainBinding.inflate(layoutInflater) }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)

        // 在 onCreate() 中获取宽高
        Log.e("measure","measure in onCreate: width=${window.decorView.width}, height=${window.decorView.height}")

        // 在 View.post() 回调中获取宽高
        binding.activity.post {
            Log.e("measure","measure in View.post: width=${window.decorView.width}, height=${window.decorView.height}")
        }
    }

    override fun onResume() {
        super.onResume()
        // 在 onResume() 回调中获取宽高
        Log.e("measure","measure in onResume: width=${window.decorView.width}, height=${window.decorView.height}")
    }
}
```

大多数人都能直截了当的给出答案：

```
E/measure: measure in onCreate: width=0, height=0
E/measure: measure in onResume: width=0, height=0
E/measure: measure in View.post: width=1080, height=2340
```

在 `onCreate()` 和 `onResume()` 中是无法获取到宽高的，而 `View.post()` 回调中可以。从日志打印顺序可以看出来，`View.post()` 回调中的打印语句是最后执行的。

抛开代码来思考一下这个问题，**什么时候可以获取到 View 的宽高？** 毫无疑问，最起码肯定得在 **View 被测量** 这个时间点之后。从上面的结果来看，`onCreate()` 和 `onResume()` 发生在这个时间点之前，`View.post()` 的回调发生在这个时间点之后。我们只要搞清楚这个时间点，问题就迎刃而解了。































































