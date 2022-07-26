---
description: asm 常用使用流程
---

# 基本流程

## 类

### 关系

> <mark style="color:red;">整个 class 的处理是链条式。起于 ClassReader 读取 class 内容，终于 ClassWriter 将所有内容转成 byte 数组</mark>

* ClassReader 按 class 文件格式将 .class 文件读取至内存
* 在 <mark style="color:red;">accept()</mark> 时会依次对内容进行读取。<mark style="color:red;">读取到一部分内容时会调用 ClassVisitor 相应的方法</mark>。如读取到字段时调用 ClassVisitor#visitField()，读取到方法是会调用 ClassVisitor#visitMethod() 等
  * 字段、方法的内容比较多，拿到 visitField()/visitMethod() 的返回值后，会再调用返回值中的 visitXXX 方法，结字段、方法的具体属性进行读取
* ClassVisitor#visitXXX 方法被调用时，默认会传给它的下一级(别的 ClassVisitor 或者 ClassWriter)
* ClassWriter 是最后一级，**它的 toByteArray() 会将所有处理过、未处理过的内容转成 byte 数组，将该数组存储到一个 .class 文件中，就会生成一个合规的 class 文件**
  * ClassWriter 在生成 class 文件时，会使用自己内部记录的 FieldVisitor、MethodVisitor 去生成字段、方法。因此，如果<mark style="color:red;">修改字段、方法，</mark><mark style="color:red;">**必须先调用 ClassWriter 相应方法**</mark><mark style="color:red;">，用拿到的返回值构建新的 FieldVisitor、MethodVisitor</mark>。

### MethodVisitor

1. <mark style="color:red;">visitMaxs</mark>()：设置局部变量表与操作数栈的最大深度。**调用到该方法时，方法体的所有指令都已访问**到。因此，<mark style="color:red;">该方法里添加的指令理论上</mark><mark style="color:red;">**都在 return 语句以后**</mark>。但要注意，如果方法中有使用 goto 进行跳转，把代码写到 return 以后也无所谓。
   1. <mark style="color:red;">visitMaxs() 并不会在方法体中添加额外的指令</mark>

### AdviceAdapter

> MethodVisitor 的子类，里面定义有 onMethodEnter()，onMethodExit()

1. onMethodExit() 会在执行到 athrow，return 等指令后调用，**首先执行 onMethodExit() 再执行相应的执行**，也就是说 <mark style="color:red;">onMethodExit() 中插入的代码会出现在 return 之前</mark>。
2. onMethodEnter()：在 visitCode() 中调用，visitCode() 会先于所有指令执行。因此 <mark style="color:red;">onMethodEnter（） 中插入的代码会出现在所有指令之前</mark>。
   1. visitCode() 中如果处理的是构造函数，并不会回调 onMethodEnter()。因为 Java 规定构造函数中的 this() super() 必须放到第一行。

## 各种参数

在构造 ClassReader、ClassWriter、ClassVisitor 时都需要传入不同的 int 值。一般情况下按如下传值：

```java
// 自动计算栈帧中的操作数栈、局部变量表、StackMapTable
ClassWriter writer = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
// 第一个参数表示 asm 版本
ClassVisitor visitor = new ClassVisitor(Opcodes.ASM9, writer)
// 最后一个参数是固定的
new ClassReader(clazzName).accept(visitor, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);
```

1. 创建 ClassWriter 时传入 <mark style="color:red;">COMPUTE\_FRAMES</mark>，那么使用 MethodVisitor 生成方法体时，<mark style="color:red;">可以不调用 visitFrame，visitMaxs() 也可随便传值</mark>
2. 每一个类都有一个版本，对应 class 文件中的 majoy\_version 与 minor\_version。在 ClassReader 会将该信息传递到 ClassVisitor#visit() 中，在 <mark style="color:red;">asm 中对应的是 Opcodes.VXXX 系列</mark>。

## 字段

### 删除

> &#x20;<mark style="color:red;">ClassVisitor#visitField() 返回 null</mark> 即可

### 添加

> 向 class 添加字段，修改的是 class 文件，所以<mark style="color:red;">在 class 结尾处调用 visitField() 方法</mark>

1. <mark style="color:red;">**不是修改 FieldVisitor**</mark>。一个 FieldVisitor 代表一个字段，新添加的必须要新建 FieldVisitor，因此<mark style="color:red;">**需要重新调用 ClassVisitor#visitField() 方法**</mark>
2. 不要在 visitField() 中添加字段，因为有可能添加重复。下面的示例只是演示，没有考虑字段重复。

```java
ClassVisitor visitor = new ClassVisitor(Opcodes.ASM9, writer) {
    @Override
    public void visitEnd() {
        // 类访问结束，重新调用 visitField() 添加一个新字段
        // 最后一个参数 1 表示字段在声明时的值
        // 添加的字段为：public final int addTo = 1;
        FieldVisitor fv = cv.visitField(Opcodes.ACC_FINAL | Opcodes.ACC_PUBLIC,
         "addTo", "I", null, 1);
        fv.visitEnd();
        super.visitEnd();
    }
};
new ClassReader(clazzName).accept(visitor, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);
```

## 查看 asm 代码

* 在写 asm 时，如果想自己写 asm 代码，基本上不可能。<mark style="color:red;">asm 提供的 TraceClassVisitor 可帮助得到相应的 asm 代码</mark>。
* 如果想通过查看 asm 代码书写 asm 代码，<mark style="color:red;">**classReader#accept() 中的参数一定要与写的地方一致**</mark>。

```java
    public static void main(String[] args) {
        // Node2 是最终想要生成的样子
        String clazzName = "com.sogle.lib.Node2";
        Printer p = new ASMifier();
        PrintWriter pw = new PrintWriter(System.out);
        try {
            // 使用 TraceClassVisitor 任何为 ClassReader 的下游
            ClassVisitor visitor = new TraceClassVisitor(null, p, pw);
            new ClassReader(clazzName).accept(visitor, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
