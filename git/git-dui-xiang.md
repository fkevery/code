---
description: 基于官方文档的总结
---

# git 对象

[官方文档链接](https://git-scm.com/book/zh/v2/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-Git-%E5%AF%B9%E8%B1%A1)

## .git 目录

使用 git init 一个文件夹后，就会在文件夹下生成 .git 隐藏目录，里面存储有当前 git 库的所有信息。git 对象都存储与此。如下是一个只提交过一次的目录&#x20;

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

## 文件存储

git 所有的文件都存储在上图中的 objects 目录下。同时 git 会为每一个文件生成 40 位的 hash 码。<mark style="color:red;">由前两位作为文件夹名，后 38 位做为文件名。</mark>所以上图中 objects 目录下会有不同的文件夹。

每一次 commit、自己本地的实际文件都会在 objects 中有一个对应的文件。

git 并不会存储本地实际文件每次变化的内容，而是将内容完全复制到 objects 目录下。所以 git 库大起来会非常快。

## 文件分类

git 将文件分为三类，可通过 <mark style="color:red;">`git cat-file -t <40位hash码>`</mark> 查看

commit：每一次 commit 生成的文件对象所属类型

blob：本地实际文件对应类型，不包含文件夹。存储的是文件的实际内容

tree：文件夹对象的类型。里面有多条记录，可以指向别的 tree 或者 blob。当我们使用 commit 命令时，git 会根据<mark style="color:red;">暂存区</mark>(git add 命令会将指定文件添加至暂存区)<mark style="color:red;">内容生成一个 tree 对象</mark>。

## commit 本质

每一个 commit 其实就在 objects 目录下新建一个文件，里面记录了本次 commit 相关的相信。如 commit msg，提交人，提交时间。最重要的是<mark style="color:red;">它 tree 属性，可以关联出本次提交所有涉及到的文件</mark>。

如下 1 处就是 tree 属性，2 处是提交是的 msg

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

## cat-file

上面用到的该命令，它可以用该命令查看一个文件的内容。比如上图就 cat 的是一个 commit 文件。我们现在 cat 它 tree 指向的对象。

```
git cat-file -p 2730834fdc74d5cb9c71996537aafc7e61430a20
100644 blob 72943a16fb2c8f38f9dde202b7a70ccc19c52f34	aa
```

第二行就返回了该对象所指向的所有文件。blob 指名这个对象是一个实际的文件对象，通过 cat 后面的 72xx 就可以得到文件的具体内容。

## hash-object

上面说过 git 会为每一个文件生成 40 位的 hash 码，hash-object 就是用来生成 hash 码的命令。
