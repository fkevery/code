---
description: 使用动态代理实现 mixin 功能
---

# 实现 mixin 效果

> mixin 是一个类，它提供了一些方法实现。其他类可使用它定义的方法又不需要继承 mixin 类

两个小问题

1. 通过 invoke 中的 method 对象如何找到对应的接口实例？
   * **可使用 map 存储方法签名与实例对象之间的映射关系**。但不同的同签名方法就无法区分对象，只能择一调用
2. **使用时必须将代理对象强转成接口对象**

代码如下：

```java
public class JavaModule {

    public static void main(String[] args) {
        // 创建两个接口的实现类。A,B 为接口
        final A a = new A() {
        };

        final B b = new B() {
        };
        // 通过动态代理创建实现 A,B 接口的实例，o 可以强转为 A,B
        // 在 invoke() 中将调用转到 A,B 的实例上，所以 o 相当于集合了 A,B 两个实例的所有功能
        Object o = Proxy.newProxyInstance(JavaModule.class.getClassLoader(), new Class[]{A.class, B.class}, new InvocationHandler() {
            @Override
            public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
                // 这里需要根据 method 决定使用哪一个实现类。demo 上直接用 a
                return method.invoke(a, objects);
            }
        });

        // 将 o 强转为 A 实例，后续就可以将 a1 当 A 的实例使用
        A a1 = (A) o;
    }
}
```

