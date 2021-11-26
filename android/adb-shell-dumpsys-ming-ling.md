---
description: dumpsys 可以导出各种信息
---

# adb shell dumpsys 命令

## 原理

[dumpsys 介绍](https://duanqz.github.io/2015-07-19-Intro-to-dumpsys#21-dumpsys%E7%9A%84%E4%BB%A3%E7%A0%81%E9%80%BB%E8%BE%91)

简单说：dumpsys 命令最终会执行到系统中 dumpsys.cpp，它是一个单独的进程。它最终通过 serviceManager 拿到系统服务对应的 binder 实体，然后<mark style="color:red;">调用 dump() 方法</mark>，由各个服务自己决定写入哪些内容。

一句话总结 ：<mark style="color:red;">dumpsys 其实是根据参数来找到某个具体的 service（如 ams, wms 等），然后执行其dump() 方法</mark>

## -l

> 列出所有系统服务

从原理一节中知道，dumpsys 最终拿到的是 serviceManager 中的服务实体，所以 **-l 列出的服务必然是已注册到 serviceManager。这些服务的名称也就是代码中通过 getSystemService() 时传入的一样。**

\-l 列出的所有服务名都是 ServiceManager#listServices() 的返回结果

## input

> 命令格式：adb shell dumpsys input。用于<mark style="color:red;">导出 input 事件，包括 touch 事件</mark>等

导出的信息总体上分为几部分：

1. 前几部分是 EventHub 与 InputReader。根据 ims 逻辑，这两部分只是负责读取事件并交由 InputDispatcher，并不会导致 anr，所以信息用法不大。
2. InputClassifier 逻辑。这一部分在 ims 中没有涉及到，可能新版本添加的，也没啥重要消息。
3. InputDispatcher 部分以及 anr 发生时 InputDispatcher 的状态。

## activity

> activity 表示<mark style="color:red;">使用 serviceManager 中的 ams 服务</mark>

1. ams 管理着 activity，broadcastReceive, service，contentProvider 四大组件，如果想单独输出某个类型组件的信息可在命令后跟 a, b, s, prov 分别表示只输出 activity，br, service, cp 的信息。
2. \-p 包名： 只输出某个应用的相关信息

## cpuinfo

> 导出 cpu 信息

## meminfo 内存检测

> 导出内存信息，可通过 -p 指定只查看某个应用

* 不指定具体应用时，返回的信息会按 RSS，PSS 分别列举，<mark style="color:red;">只关注 PSS 相关</mark>的即可

![不指定进程时效果](<../.gitbook/assets/iShot2021-11-26 16.07.03.png>)

* 指定具体应用时，会列出当前应用所使用的内存大小以及一些特殊 object 的个数。应着重关注 <mark style="color:red;">native heap alloc、dalvik heap alloc，</mark>前者表示 native 层分配的大小、后者表示 java 层分配的大小。<mark style="color:red;">views, activities 等数量，如果一直增大就有可能发生内存泄漏</mark>

## gfxinfo 帧率检测

* <mark style="color:red;">adb shell dumpsys gfxinfo 包名：导出指定包的帧率信息</mark>
* 这里面其实主要关注 <mark style="color:red;">janky frames</mark> 行，它表示超时帧的占比

![](<../.gitbook/assets/iShot2021-11-26 16.53.22.png>)

<mark style="color:red;"></mark>

<mark style="color:red;"></mark>
