---
description: linux 基础知识，平时使用时总结的
---

# Linux 基础

## 内存相关

### VSS RSS PSS USS

[Linux 中 vss rss pss uss 区别](https://segmentfault.com/a/1190000040077427)

[vss/rss/pss/uss 介绍](https://www.jianshu.com/p/3bab26d25d2e)

[VSZ 与 RSS](https://cloud.tencent.com/developer/article/1411974)

在查看内存时经常可以看到 vss, rss, pss, uss 几个单词，它们的含义如下。首先说明，系统中除了各应用自己占的内存久，还有一些共享库，<mark style="color:red;">共享库无论被多少个进程使用，在内存中都只有一份</mark>

| 名称      | 全称                    | 含义                                                      | 用处                                   |
| ------- | --------------------- | ------------------------------------------------------- | ------------------------------------ |
| VSS/VSZ | Virtual Set Size      | 虚拟内存。包含**共享库占用的全部内存，及分配但未使用内存**                         | 用处不大                                 |
| RSS     | Resident Set Size     | 实际物理内存。**包含共享库占用的全部内存**                                 | 用处一般                                 |
| PSS     | Proportional Set Size | 实际物理内存。但<mark style="color:red;">将共享库占用共享按进程数平均分</mark> | 用处一般                                 |
| USS     | Unique Set Size       |  实际物理内存。<mark style="color:red;">不包含共享库占用的内存</mark>     | <mark style="color:red;">用处最大</mark> |

上面一直提到共享内存，例如 A, B 打开同一个文件，这个文件使用的内存就是共享的。

## 系统文件

### stat 文件

linux 一共有三个 stat 文件，分别记录了系统的整体信息，某个进程的统计信息以及某个线程的统计信息。要注意的是，这些<mark style="color:red;">信息都是动态获取，只存在于内存中</mark>

[进程 stat 文件分析](https://www.cnblogs.com/Jimmy1988/p/10045601.html)

[系统 stat 文件分析](https://www.cpweb.top/504)

| 位置                      | 作用                       |
| ----------------------- | ------------------------ |
| /proc/stat              |  系统进程整体统计信息              |
| /proc/pid/stat          | 某一进程的统计信息                |
| /proc/pid/task/tid/stat |  某一进程的统信息。基本上与进程 stat 相同 |

上面几个文件有啥用我也不知道，目前只在 [square 的 tart 库](https://github.com/square/tart)中见过从进程的 stat 文件读取信息当作进程的启动时间

## 常用命令

### top

<mark style="color:red;">动态显示系统进程信息，隔一段时间会刷新信息</mark>

1. top -p \[pid] 通过指定 pid 监控指定的进程。可以多次使用 -p 监控多个进程，如 top -p P1 -p P2
2. top -d \[time]：指定刷新屏幕时间间隔
3. top -H -p \[pid]：查看 pid 进程下所有的线程
4. 可结合 grep 命令查看指定进程在一定时间内 cpu 使用情况，要注意**如果包名过长，可能 grep 不到，缩短点即可**

```
adb shell top -d 5 | grep com.sogle  // 查看 com.sogle 应用
```

![](<../.gitbook/assets/iShot2021-11-26 15.25.22.png>)

上面信息重点关注：

1. <mark style="color:red;">swap 的 used：</mark>如果数据在<mark style="color:red;">不断变化</mark>，说明在进程内存与 swap 分区的数据交换<mark style="color:red;">，内存真的不够用了</mark>

其余信息如下：

1. PR 与 NI：进程优先级，<mark style="color:red;">值越小优先级越高</mark>
2. RES 与 SHR：实际占用的物理内存与共享内存大小

### ps

<mark style="color:red;">列出所有进程信息</mark>

1. ps -T -p pid：列出 pid 进程中所有线程，但好像 java 层创建的并不会在其中显示。它与 top 命令差不多，只不过 top 会时时刷新

## 进程与线程

### 进程状态

进程状态经常是一些字母的简写。在 android anr 分析时也看到线程 state 也是这些字母。

[linux 进程状态](https://www.cnblogs.com/klb561/p/11945157.html)

| 字母 | 含义                                       | 全称                    |
| -- | ---------------------------------------- | --------------------- |
| R  | <mark style="color:red;">可运行或正在运行</mark> | task\_running         |
| S  | 可中断的睡眠状态                                 | task\_interruptible   |
| D  | 不可中断的睡眠状态                                | task\_uninterruptible |
| T  | 暂停状态                                     | task\_stopped         |
| t  | 跟踪状态，debug 中                             | task\_traced          |
| Z  | 退出，进程成为僵尸进程                              | 僵尸进程指已结束，但未从进程表中删除的进程 |

### 线程状态

源码链接 ：[ThreadState 枚举](https://cs.android.com/android/platform/superproject/+/master:art/runtime/thread\_state.h;l=18?q=thread\_state.h\&sq=)，[nativeGetStatus 方法](https://cs.android.com/android/platform/superproject/+/master:art/runtime/native/java\_lang\_Thread.cc;l=74?q=java\_lang\_Thread.cc)，该方法是 native 层状态转 java 层状态的关键。

android 中线程状态分两部分，一部分是 Thread.java  中 State 枚举，就是常用的 NEW, BLOCKED 等。另一种是 native 层的，在分析 anr 日志中使用的是 native 层的状态。在 ThreadState 枚举中列举了 java 层与 native 层对应关系：

![](<../.gitbook/assets/iShot2021-11-24 14.21.02.png>)

从上图可以看出，native 层将 time\_wait 细化成 timedWaiting 以及 sleeping，分别对应 wait(timeout) 及 sleep() 两个方法。这明显更有利于分析。

上图中有两个关于 gc 的：<mark style="color:red;">kWaitingForGcToComplete 与 kWaitingPerformingGc</mark>，前者表示正在等待 GC，后者表示正在执行 GC，所以主线程出现前者时就应该考虑搜索后者看哪个线程正在执行 gc。

线程处于 native 状态说明在执行 jni 层代码，但这并不意味着线程没有问题，比如它正通过 binder 调用第三方应用，里面导致被阻塞出现  anr。

suspend 表示线程被暂停。有两种可能：自己太忙，cpu 强制暂停；cpu 太忙，高优先级的任务占用了大量的资源，进程得不到执行

