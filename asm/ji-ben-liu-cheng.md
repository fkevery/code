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
2. <mark style="color:red;">visitMethodInsn</mark>：调用别的方法时

```java
// owner 表示被调用的方法所属的类
// name 表示被调用方法名
// descriptor 表示被调用方法的描述符
visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface)
```

### AdviceAdapter

> MethodVisitor 的子类，里面定义有 onMethodEnter()，onMethodExit()

1. onMethodExit() 会在执行到 athrow，return 等指令后调用，**首先执行 onMethodExit() 再执行相应的执行**，也就是说 <mark style="color:red;">onMethodExit() 中插入的代码会出现在 return 之前</mark>。
2. onMethodEnter()：在 visitCode() 中调用，visitCode() 会先于所有指令执行。因此 <mark style="color:red;">onMethodEnter（） 中插入的代码会出现在所有指令之前</mark>。
   1. visitCode() 中如果处理的是构造函数，并不会回调 onMethodEnter()。因为 Java 规定构造函数中的 this() super() 必须放到第一行。

## GeneratorAdapter

> AdaviceAdapter 的父类，MethodVisitor 的子类。里面<mark style="color:red;">封装了常用的方法</mark>，比如对局部变量的访问等。

1. <mark style="color:red;">**push 系列：将参数加到操作数栈中**</mark>。内部就是对 push 等字节码指令的封装。
2. <mark style="color:red;">**arg 系列：操作方法参数，非局部变量表**</mark>。参数需要 index 指的是方法参数中的第几个，不需要考虑 this 的影响，内部会自己处理。比如 test(a,b,c) 如果传 0 表示要操作 a，
3. local 系列：处理局部变量表。比如将操作数添加至局部变量表中等。
   1. **loadThis**：<mark style="color:red;">非静态方法中用于读取方法调用对象</mark>。也就是将局部变量表中的第 0 个元素加载到操作数栈中。
4. <mark style="color:red;">**box 与 unbox 系列：自动装箱拆箱**</mark>。将 int 与 Integer 等来回互转。

例如下面语句就可以很简单的向方法中添加 sout&#x20;

```java
getStatic(Type.getType(System.class),"out",Type.getType(PrintStream.class));
push("你好");
// 注意后面的 Method 参数，它并不是反射中的 Method
// 我们只需要像写java 代码一样即可
invokeVirtual(Type.getType(PrintStream.class), Method.getMethod("void println(String)"));
```

## Type

> asm 提供的一个工具类

对同一个类，java 代码与 class 文件全量限定名不同：java 代码中包名使用点分隔，而 class 文件中使用 / 分隔（asm 中叫 internal name）。而且方法、字段都有自己的签名。**通过提供的 Type 类可以在不同的模式间转换**

实际使用中 Type 可分为两种：class Type 以及 methodType。前者通过 **Type#getType()** 生成，后者通过 **Type#getMethodType()** 生成。

* **getSort() 返回当前 Type 的类型**，与 Type 内静态常用 INT, METHOD 等比较
* **getSize() 返回当前 Type 所占 slot 数**。在局部变量表、操作数栈中一个 int 占一个 slot，一个 long/double 占两个 slot。这里返回的就是当前 Type 所能占用的 slot 数量。
* classType 常用的有：**getClassName()，getInternalName() 与 getDescriptor()**：第一个返回 className，第二个返回 internalName，第三个返回描述。示例如下：

```java
// 根据 class 创建对应的 Type
// 也可写成，根据类描述符获取类相关信息，此时就相当于对描述符进行解析
// Type type = Type.getType("Lcom/sogle/lib/Node2;");
Type type = Type.getType(Node2.class);
System.out.println(type.getClassName()); // com.sogle.lib.Node2
System.out.println(type.getInternalName()); // com/sogle/lib/Node2
System.out.println(type.getDescriptor()); // Lcom/sogle/lib/Node2;
```

* methodType 常用的有：**getReturnType 与 getArgumentTypes() 与 getArgumentsAndReturnSizes**。前者返回方法返回值对应的 Type，后者返回方法的参数对应的 Type 数组，最后一个返回参数与返回值所占的 slot 数

```java
Type type = Type.getMethodType("(IIJ)V");
System.out.println(type.getReturnType() == Type.VOID_TYPE); // 方法返回值 void
System.out.println(type.getArgumentTypes().length); // 有三个参数

int sizes = type.getArgumentsAndReturnSizes();
// 拿到 sizes 后需要再处理一下
System.out.println(sizes >> 2); // 返回 5，因为还有一个 this。
System.out.println(sizes & 3); // 返回 0
```

* 如果 type 指向数组，可通过 **getDimensions**() 拿到数组的维数，**getElementType**() 拿到数组元素对应的 type

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
