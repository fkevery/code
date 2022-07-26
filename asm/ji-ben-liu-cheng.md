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
   1. <mark style="color:red;">visitMaxs() 并不会在方法体中添加额外的指令</mark>。

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

## 方法

### 删除

> <mark style="color:red;">visitMethod 直接返回 null 即可</mark>

### 添加

跟字段的添加相同，<mark style="color:red;">只在 visitEnd() 中重新调用 ClassVisitor#visitMethod()</mark>&#x20;

### 添加 try-catch

> 由于要修改方法体，所以需要修改 ClassVisitor#visitMethod 的返回值。只要<mark style="color:red;">调用 MethodVisitor#visitxxx 方法即向方法体中添加操作</mark>。

示例代码如下：

```java
public static void main(String[] args) {
        String clazzName = "com.sogle.lib.Node";
        // 创建 classWriter，最终由它的 toByteArray 得到转换后的 class 
        ClassWriter writer = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
        try {
            // 将 ClassWriter 设置成 classVisitor 的下游节点
            ClassVisitor visitor = new ClassVisitor(Opcodes.ASM9, writer) {

                @Override
                public MethodVisitor visitMethod(int access, String name, final String descriptor, String signature, String[] exceptions) {
                    // 拿到的 MethodVisitor 是 ClassWriter 的返回值
                    // 向该 visitor 中写入指令就会在生成的 class 字节码中生成相应的操作
                    MethodVisitor method = super.visitMethod(access, name, descriptor, signature, exceptions);
                    if ("test".equals(name)) {
                        // 只对 test 方法进行处理
                        
                        // 找到想要修改的方法，所以返回自己的 MethodVisitor
                        // 注意将 ClassWriter 设置成我们自己的 MethodVisitor 的下游节点
                        return new AdviceAdapter(Opcodes.ASM9, method, access, name, descriptor) {
                            // 这些都是添加 try-catch 操作的基本流程
                            private final Label from = new Label(),
                                    to = new Label(),
                                    target = new Label();
                            private final Label label3 = new Label();

                            @Override
                            protected void onMethodEnter() {
                                mv.visitTryCatchBlock(from, to, target, "java/lang/Exception");
                                mv.visitLabel(from);
                            }
                            // 注意这里是重写 visitMaxs() 而不是 visitEnd()
                
                            @Override
                            protected void onMethodExit(int opcode) {{
                                // 到这里，说明正常的方法体已经执行完毕
                                // 加个 label，表示 try 住的内容
                                mv.visitLabel(to);
                                // 如果没有出异常，直接 goto 到 Label3，也就是方法结束
                                mv.visitJumpInsn(GOTO, label3);
                                
                                // 添加 catch 代码块的内容
                                mv.visitLabel(target);
                                // 由于 classWriter 用的是 compute_frame
                                // 所以这里不需要调用 visitFrame
                                // mv.visitFrame(Opcodes.F_NEW, 1, new Object[]{"com/sogle/lib/Node"}, 1, new Object[]{"java/lang/Exception"});
                                // 出异常时操作数栈中的 1 元素就是异常类
                                // 先存储至局部变量表
                                mv.visitVarInsn(ASTORE, 1);
                                // 下面三行是额外输出 catch，可以不要
                                mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                                mv.visitLdcInsn("catch");
                                mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
                                // 将局部变量表中的异常类添加到操作数栈中
                                mv.visitVarInsn(ALOAD, 1);
                                // 调用它的 printStackTrace()
                                mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Exception", "printStackTrace", "()V", false);

                                // 在方法结束时插入 label3，用于正常执行完成时直接跳转至这里
                                mv.visitLabel(label3);
                            }
                        };
                    }
                    return method;
                }
            };
            
            new ClassReader(clazzName).accept(visitor, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);
            byte[] bytes = writer.toByteArray();
            // 将新生成的 class 文件存储至某个地方
            File f = new File("/Users/xx/Desktop/com/sogle/lib");
            f.mkdirs();
            f = new File(f, "Node.class");
            FileOutputStream fos = new FileOutputStream(f);
            fos.write(bytes);
            fos.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

1. onMethodExit() 会在执行到 return，throw 指令时调用，onMethodExit() 执行完后才会执行相应的指令。因此，<mark style="color:red;">onMethodExit() 就是方法执行结束时的回调</mark>。

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
