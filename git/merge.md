---
description: 使用 rebase 合并提交
---

# rebase

## 基础

1. rebase 弹出的交互界面中，<mark style="color:red;">先提交的排在前面</mark>
2. 交互窗口中常用的指令如下
   1. s：<mark style="color:red;">将当前提交合并到上一个提交中，并保留提交时输入的信息</mark>
   2. f：<mark style="color:red;">与 s 类似，但舍弃 commit 时输入的信息</mark>。相当于使用 s，然后在编辑 commit 信息时直接删除本次提交的信息
   3. r：保留 commit，但<mark style="color:red;">可重新编辑 commit 信息。可用于修改历史上某个 commit 的信息</mark>
   4. e：<mark style="color:red;">重新编辑某次提交</mark>，可用于拆分提交。实际上，git 会在当次提交处停下，可通过 git reset HEAD\~1 重置这次提交，然后可分批 commit

## 拆分提交

> 通过 rebase 中的 edit/e 操作

1. 将想拆分的提交前的字母修改成 e/edit，然后 git 会自动停在需要处理的提交中
2. 通过 <mark style="color:red;">git reset HEAD\~1</mark> 重置本次提交
3. 根据实际使用通过 git commit 额外建立提交
4. 处理完后，使用 <mark style="color:red;">git rebase --continue</mark> 继续 rebase。git 会自动应用这些新建的提交

## 合并提交

### 操作

1. 使用 **git rebase -i \<commitId>**
2. 弹出界面中，**将要合并的 commit 前的 pick 修改为 s**
3. 弹出新的界面中，输入新的 commit 信息
4. 通过 rebase 修改已提交至远程仓库的 commit，**必须要使用强制 push**

### 演示

假设当前分支结构如下图：

![分支结构](img/log1.png)

现在要将 2/3/4 合并到一起，使用如下：

```
git rebase -i HEAD~3
```

弹出界面内容如下：

```
pick d967c73 2
pick e12e0d9 3
pick 10127ad 4
```

将**后两个 pick 改为 s，表示使用 commit 的内容，但不使用 commit 信息**。最终样式如下：

```
pick d967c73 2
s e12e0d9 3
s 10127ad 4
```

在**新界面中输入新 commit 的信息**，然后保存即可生成新的 commit 节点。

最终生成新的 commit 信息如下，其中的 rebasexxxx 就是新的 commit 节点提交信息。

![分支结构](img/log2.png)

## 交换提交

> 用于交换两个 commit

1. 通过 rebase 命令，在弹出界面中直接交换两个 commit 即可

旧提交如下

<figure><img src="../.gitbook/assets/image (3) (2).png" alt=""><figcaption></figcaption></figure>

输入 <mark style="color:red;">git rebase -i</mark> HEAD\~2，然后通过 vim 中的 <mark style="color:red;">**ddp**</mark> 按钮直接交换两行，保存

<figure><img src="../.gitbook/assets/image (4) (1).png" alt=""><figcaption></figcaption></figure>

最终形成的效果如下：

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>
