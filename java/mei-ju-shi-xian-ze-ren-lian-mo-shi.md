---
description: 使用枚举实现责任链模式
---

# 枚举实现责任链模式

有两点问题需要注意：

1. **如果只是简单的处理，可直接使用一个枚举类，此时结点的处理顺序与枚举中定义顺序相同**
2. **如果有不同的顺序策略，可以另定义一个枚举，用于定义不同的策略**

代码如下

```java
public class JavaModule {

    public static void main(String[] args) {
        // 使用策略枚举中的策略，它会返回一个处理类的不同组合
        for (Handler h : HandlerStrategy.FIRST.values) {
            if (h.canHandler()) {
                h.handle();
                break;
            }
        }
        for (Handler h : HandlerStrategy.SECOND.values) {
            if (h.canHandler()) {
                h.handle();
                break;
            }
        }
    }
}
// 定义顺序策略
enum HandlerStrategy {
    // 一种策略
    FIRST(Handler.S1, Handler.S2),

    SECOND(Handler.S2, Handler.S1);

    Handler[] values;

    HandlerStrategy(Handler... handlers) {
        this.values = handlers;
    }
}
// 定义责任链上的每一个节点
enum Handler {
    S1() {
        @Override
        boolean canHandler() {
            System.out.println("1 canHandler");
            return false;
        }

        @Override
        void handle() {
            System.out.println("1 handle");
        }
    }, S2() {
        @Override
        boolean canHandler() {
            System.out.println("2 canHandler");
            return true;
        }

        @Override
        void handle() {
            System.out.println("2 handle");
        }
    };

    abstract boolean canHandler();

    abstract void handle();
}
```

