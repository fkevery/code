---
description: 内存分析工具 mat 使用
---

# MAT

## OQL 语句

> `每一个类就是一张表，实例是行，属性是列`

一个简单示例

![](<../.gitbook/assets/image (2).png>)

1. 使用时，<mark style="color:red;">**一定要类型匹配。**</mark>如上图属性是 Long 类型，所以 OQL 的时候后面加有 L
2. 在表名后加上 s，用 s.xxx 引用 s 中的属性<mark style="color:red;">；也可以不加</mark>
3.  可以使用正常的 sql 语句，也有一些特殊的关键字。例如

    <mark style="color:red;">INSTANCEOF：查出指定类的所有实例，包括子类</mark>

    <mark style="color:red;">OBJECTS：查找当前类的 class</mark>&#x20;

![](<../.gitbook/assets/image (1).png>)
