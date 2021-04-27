# SdkInit

> 利用 CountDownLatch 实现 sdk 初始化

代码如下：

```kotlin
object SdkInit {
    // 初始值设置为 1
    private val countDownLatch = CountDownLatch(1)
    fun initSdk() {
        thread {
            Thread.sleep(3000)
            countDownLatch.countDown()
        }
    }
    // 只有 sdk 初始化完成后才会执行 runnable
    // 如果调用时 sdk 已经初始化，会直接执行
    fun ensureInit(runnable: Runnable) {
        // 使用一个新线程，免得阻塞当前调用线程
        thread {
            countDownLatch.await()
            runnable.run()
        }
    }
}
```

1. 在 ensureInit\(\) 中可以根据需要将 runnable 的执行切换到主线程中

