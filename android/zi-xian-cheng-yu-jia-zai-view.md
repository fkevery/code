# 子线程预加载 view

启动优化时往往可以在<mark style="color:red;">子线程中提前 inflate 主界面要使用的布局</mark>，主界面使用时可以直接使用，省去了主线程 inflate 的时间。这里有几个问题需要解决：

## context 的替换

虽然可以**通过 application/service 等去拿到 LayoutInflater 或者直接当作参数传递到 View 构造函数中**，但真实使用时还必须替换成相应的 activity。系统提供了 <mark style="color:red;">MutableContextWrapper 用于方便地进行替换</mark>

```kotlin
object ViewFactory {
    private val vs = LinkedList<View>()
    fun init(context: Application) {
        // 创建时传入 application，并封装成 MutableContextWrapper
        val i = AsyncLayoutInflater(MutableContextWrapper(context))
        i.inflate(
            R.layout.rv_item, null
        ) { view, resid, parent -> vs.add(view) }
        i.inflate(
            R.layout.rv_item, null
        ) { view, resid, parent -> vs.add(view) }
    }

    // 真正使用时替换成 activity
    fun get(context: Activity): View {
        val mc = vs[0].context as MutableContextWrapper
        mc.baseContext = context // 替换成 activity。【代码一】
        return vs[0]
    }
}
```

在 contextWrapper 源码中，所有操作都是通过 mBase 属性传递到 ContextImpl 中。而 MutableContextWrapper 继承于 ContextWrapper，【代码一】处修改了 mBase 属性。因此修改后，所有的操作都转交到 activity 中，与实际通过 activity 创建的 view 并没有区别。

## 使用 activity theme

activity 继承于 ContextThemeWrapper，它又继承于 ContextWrapper，而 Application 直接继承自 ContextWrapper。

ContextThemeWrapper 与 ContextWrapper 相比最大的区别就是：前者可以有自己单独的 theme 及  Resource，它的 getTheme() 返回的是自己单独的 theme。因此，对于 application 创建的 view，它的 theme 肯定和 activity 创建的不一样。

但<mark style="color:red;">可重写 Application#getTheme，将 theme 强制使用 activity 的  theme。</mark>

```kotlin
    override fun getTheme(): Resources.Theme {
        val t = super.getTheme()
        t.applyStyle(R.style.AndroidDemo, true)
        return t
    }
```

**上述代码只是测试使用**，真实使用时不能直接对 theme 强制替换，需根据不同的使用场景决定是否替换。

## Handler

在自定义 view 的构造函数时，有可能会创建一个主线程 Handler，用于通过主线程更新 UI。但由于我们是在子线程中创建的 view，就会导致该 Handler 并没有达到使用者想要的效果。这里会有两个问题：

1. 由于子线程并没有调用 Looper#prepare()，所以使用 Handler 时直接报错
2. 即使子线程使用了 Looper#prepare()，但使用者本意是想向主线程发消息，现在改成了子线程，使用时也有可能报错

上述两个问题可以提供一个解决思路。**Looper#prepare 通过 ThreadLocal 向当前线程绑定一个 Looper，Handler 在创建时通过 ThreadLocal 读取跟线程绑定的 Looper。Handler 发送的 msg 怎么会在哪个线程处理，完全取决于它使用的 Looper 是在哪个线程中启动的 loop()。**因此，有如下思路

### 思路一

1. 先通过 Thread#currentThread() 拿到当前线程，进而拿到跟线程相关的 ThreadLocalMap
2. 遍历 ThreadLocalMap，拿到 value 值是当前 Looper 的 key-value 字段（其实是一个 Entry 对象）
3. 将 Entry 中的值修改成 mainLooper（通过 Looper#getMainLooper() 拿到）

### 思路二

上面逻辑用于 Looper#sThreadLocal 无法通过反射拿到时，否则拿到后直接调用 set 方法修改成 mainLooper 即可。如下

```kotlin
val local = Looper::class.java.getDeclaredField("sThreadLocal").run {
    isAccessible = true
    this[null] as ThreadLocal<Looper>
}
local.set(Looper.getMainLooper())
```
