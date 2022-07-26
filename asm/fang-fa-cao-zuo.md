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
