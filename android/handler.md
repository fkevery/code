---
description: 绕过 mCallback 的 UnsupportedAppUsage 注解
---

# Handler

## 处理 mCallback

Handler 处理消息时，按三种顺序处理：

1. Message 自身的 callback
2. Handler 自己设置的 mCallback
3. Handler 的 handleMessage() 方法

所以有时候为了能提前处理某些信息，都需要反射修改 mCallback，特别是与 ActivityThread 结合时。

但 mCallback 被注解 UnsupportedAppUsage 修饰，因此直接反射会报错

```java
@UnsupportedAppUsage
final Callback mCallback;
```

解决方法是：使用 <mark style="color:red;">**superclass 而不是直接使用 Handler 自身的 class**</mark>

```kotlin
// 拿到 Handler 对象
val mH: Handler = getDeclaredField("mH").run {
    isAccessible = true
    get(sCurrentActivityThread)
} as Handler
// 通过 superclass#getDeclaredField 拿到 mCallback，不能直接使用 mH.javaClass.getDeclaredField
val mCallback: Handler.Callback? = mH.javaClass.superclass.getDeclaredField("mCallback").run {
    isAccessible = true
    get(mH)
} as Handler.Callback?
```

## 所有 msg

通过反射获取线程中 MessageQueue 中现存的所有 msg

```kotlin
    fun getAllMsg() {
        try {
            val queue = Looper.getMainLooper().queue
            val messages = queue.javaClass.getDeclaredField("mMessages")
            messages.isAccessible = true
            val o1 = messages[queue] as Message
            printMsg(o1)
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }

    private fun printMsg(msg: Message) {
        Log.e("TAG", "msg == $msg")
        try {
            val next = msg.javaClass.getDeclaredField("next")
            next.isAccessible = true
            val msgNext = next[msg] as? Message
            msgNext?.let { printMsg(it) }
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }
```
