---
description: linux 基础知识，平时使用时总结的
---

# Linux 基础

## 内存相关

### VSS RSS PSS USS

[Linux 中 vss rss pss uss 区别](https://segmentfault.com/a/1190000040077427)

[vss/rss/pss/uss 介绍](https://www.jianshu.com/p/3bab26d25d2e)

在查看内存时经常可以看到 vss, rss, pss, uss 几个单词，它们的含义如下。首先说明，系统中除了各应用自己占的内存久，还有一些共享库，<mark style="color:red;">共享库无论被多少个进程使用，在内存中都只有一份</mark>

| 名称  | 全称                    | 含义                                                      | 用处                                   |
| --- | --------------------- | ------------------------------------------------------- | ------------------------------------ |
| VSS | Virtual Set Size      | 虚拟内存。包含**共享库占用的全部内存，及分配但未使用内存**                         | 用处不大                                 |
| RSS | Resident Set Size     | 实际物理内存。**包含共享库占用的全部内存**                                 | 用处不大                                 |
| PSS | Proportional Set Size | 实际物理内存。但<mark style="color:red;">将共享库占用共享按进程数平均分</mark> | 用处一般                                 |
| USS | Unique Set Size       |  实际物理内存。<mark style="color:red;">不包含共享库占用的内存</mark>     | <mark style="color:red;">用处最大</mark> |

<mark style="color:red;"></mark>
