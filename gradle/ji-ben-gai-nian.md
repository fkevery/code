---
description: gradle 入门概念
---

# 基本概念

## 基础

1. gradle 会为每一个 build.gradle 生成一个 project 对象
2. <mark style="color:red;">**gradle 的核心是 task，**</mark>我们执行的各种 gradle 命令其实质就是一个 task。每一个 task 由不同的 action 组成。由 doFirst/doLast 向 task 中添加 action
3. 我们依赖的 java 插件，android 插件等都会定义不同的 task

## gradle 命令

> gradle 在执行不同 task 时可配置的参数

1. \-h：查看帮助文档
2. **-m, --dry-run：不执行任何 action**。可以用来查看某个 task 的所有依赖
3. \-q，--quiet：只输出错误日志
4. **-x：不执行某个 task**。比如 task A 依赖 task B，如果在执行 task A 时指定 -x B，那么 task B 就不会执行

## android 插件

### assembleRelease 添加依赖

android 想打一个 apk 包时，执行 assembleRelease task，想给它添加一些依赖，在生成 apk 之前做些操作，代码如下：

```groovy
// 这部分代码一定要写到 app 的 build.gradle 中
tasks.whenTaskAdded { Task task ->
    if (task.name == "assembleRelease")
        task.dependsOn "a"
}
```
