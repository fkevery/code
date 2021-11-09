---
description: 绕过 mCallback 的 UnsupportedAppUsage 注解
---

# 反射 Handler

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

```java
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
