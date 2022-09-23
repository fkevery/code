---
description: 通过 asm 对方法 crud
---

# 方法操作

## 删除

> <mark style="color:red;">visitMethod 直接返回 null 即可</mark>

## 添加

跟字段的添加相同，<mark style="color:red;">只在 visitEnd() 中重新调用 ClassVisitor#visitMethod()</mark>&#x20;

## 添加 try-catch

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

## 方法前输出参数

由于需要修改方法体，所以需要自定义 MethodVisitor，在 onMethodEnter 中添加输出语句

```java
protected void onMethodEnter() {
    Type type = Type.getMethodType(descriptor);
    // 当前方法非 static 方法，所以第一个参数为 this，占据了局部变量表中的第一个 slot
    // 所以 offset 偏移 1 位。如果是 static 就为 0
    // 是不是 static 需要根据 visitMethod 中的 access 判断
    int offset = 1;
    // 遍历所有参数
    for (Type argumentType : type.getArgumentTypes()) {
        // 这里只挑选两种类型进行处理
        if (argumentType.getSort() == Type.INT) {
            super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            // 第二个参数为要加载的元素在局部变量表中的位置
            // 用的是 Iload，处理的是 Int 数字
            super.visitVarInsn(ILOAD, offset);
            super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(I)V", false);
        }else if (argumentType.getSort() == Type.OBJECT) {
            super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            // 这里用的命令是 aload。处理的是对象
            super.visitVarInsn(Opcodes.ALOAD, offset);
            super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/Object;)V", false);
        }
        // 记录偏移量
        offset += argumentType.getSize();
    }
}
```

## 替换调用

> 一定要注意：<mark style="color:red;">保证局部变量表与操作数栈一致</mark>。替换前后的方法可能需要的参数不一致，在替换时要保证两者一致。

1. 替换调用，需要修改方法体，所以需要<mark style="color:red;">自定义 MethodVisitor</mark>。由于是替换调用，所以需要修改其 <mark style="color:red;">visitMethodInsn() 方法</mark>。

```java
@Override
public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface) {
    if ("println".equals(name)) {
        // 将 println 方法替换成自己的
        // 这里只判断了方法名，正常情况下需要判断各种条件
        super.visitMethodInsn(Opcodes.INVOKESTATIC, "com/xx/lib/Log", "e", "(Ljava/io/PrintStream;Ljava/lang/String;)V", false);
    } else {
        super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
    }
}
```

2\. 为了保证操作数栈的正确性：

* 如果被替换的方法需要的<mark style="color:red;">操作数少，额外在方法参数中添加相应的参数但不使用</mark>。这个思路<mark style="color:red;">**适合由非静态方法转为静态方法调用**</mark>。

## 删除语句

> 输出语句一般涉及到多个命令，因此在处理时需要记录前面的命令。使用<mark style="color:red;">状态机模式</mark>

比如要处理的命令涉及 a,b,c 三个指令。遇到  a 指令时将状态变为 1，遇到  b 指令时如果状态是 1 且指令满足要求，就将状态记为 2；遇到 c 指令，且状态是 2 表示这是要处理的三条指令，直接返回即可。如果中途遇到其余指令，就需要将状态回归到最原始状态，即 0。

下面以删除 system.out.println 为例进行说明。

因为要处理的是方法内部逻辑，所以需要自定义 MethodVisitor。

![](<../.gitbook/assets/image (2).png>)

上面缺了最后一步，如果中间遇到别的指令怎么办？就需要将原来被拦截的指令再次发出去

```java
if (state == 1) {
    // 此种状态只拦截了 System.out
    super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
} else if (state == 2) {
    // 此种状态拦截了 System.out 与 加载操作数
    super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitLdcInsn(msg);
}
// 将状态回 0
state = 0;
```

## 方法运行时间统计

> 统计方法运行时间需要额外添加局部变量，就需要额外使用到 <mark style="color:red;">LocalVariablesSorter</mark> 中的功能

1. <mark style="color:red;">由于 AdviceAdapter 是 LocalVariablesSorter 子类</mark>，所以这里直接使用 AdviceAdapter 即可

```java
private int slotIndex;
@Override
protected void onMethodEnter() {
    // LocalVariablesSorter 中方法，表示要新添加一个 long 类型的局部变量
    // 返回参数是新变量在局部变量表中的下标
    slotIndex = newLocal(Type.LONG_TYPE);
    invokeStatic(Type.getType(System.class), Method.getMethod("long currentTimeMillis()"));
    storeLocal(slotIndex);
}
@Override
protected void onMethodExit(int opcode) {
    // getStatic() dup() 等方法都是 GeneratorAdapter 中提供的方法，可方便地执行某些命令
    getStatic(Type.getType(System.class), "out", Type.getType("Ljava/io/PrintStream;"));
    // 以下三行是生成一个 StringBuilder 类对象
    // 因为输出的时候有字符串拼接，默认会使用 StringBuilder 拼接
    newInstance(Type.getType(StringBuilder.class));
    dup();
    invokeConstructor(Type.getType(StringBuilder.class), Method.getMethod("void <init>()"));
    push("dur = ");
    invokeVirtual(Type.getType(StringBuilder.class), Method.getMethod("StringBuilder append(String)"));
    // 拿到运行时截止时间
    invokeStatic(Type.getType(System.class), Method.getMethod("long currentTimeMillis()"));
    // 将开始时间添加到操作数栈中
    Type type = getLocalType(slotIndex);
    // 这里不要直接调用 visitVarInsn()，会出错
    method.visitVarInsn(Opcodes.LLOAD,slotIndex);
    visitInsn(Opcodes.LSUB);
    invokeVirtual(Type.getType(StringBuilder.class), Method.getMethod("StringBuilder append(long)"));
    invokeVirtual(Type.getType(StringBuilder.class), Method.getMethod("String toString()"));
    invokeVirtual(Type.getType(PrintStream.class), Method.getMethod("void println(String)"));
}
```
