---
title: 认识git的数据模型
date: 2019-01-09 14:07:59
tags: Git
---

自2005年诞生以来，git已经在开源世界中大受欢迎，我们中的许多人也在我们的工作岗位上使用它。 它是一个很棒的VCS工具，具有很多优点，但易于学习并不是其中之一。 对于git如果只会死记硬背命令那么要不了多久你就会忘记,然后一而再而三的背诵,无疑让人很受打击,在我看来，熟悉使用git甚至开始喜欢它的唯一方法是了解它如何在内部工作。 

git命令只是对数据存储的抽象,如果不了解git的工作原理，无论我们在笔记中记忆或存储了多少git命令或技巧我们仍然会对git的使用感到困惑.而git则是通过抽象的命令来暴露它的数据结构的使用方法.

所以这边文章我们更多的要关注git的内部关系-数据模型,当然这篇文章不会涉及到git的源码.

<!--more-->

### 准备工作

#### 初始化仓库
为了讲解数据模型,我们首先要在自己的工作目录下初始化一个空的git仓库
```
git init
```

git会告知我们已经在当前的目录下创建了一个.git目录,我们来看看这个.git长什么样子.

```
$ tree .git/
.git
|-- HEAD
|-- config
|-- description
|-- hooks
|   |-- applypatch-msg.sample
|   |-- commit-msg.sample
|   |-- fsmonitor-watchman.sample
|   |-- post-update.sample
|   |-- pre-applypatch.sample
|   |-- pre-commit.sample
|   |-- pre-push.sample
|   |-- pre-rebase.sample
|   |-- pre-receive.sample
|   |-- prepare-commit-msg.sample
|   |-- update.sample
|-- info
|   |-- exclude
|-- objects
|   |-- info
|   |-- pack
|-- refs
    |-- heads
    |-- tags

8 directories, 15 files
```

其中一些文件和目录是不是看着有些熟悉,现在我们主要还是看`objects`这个目录,现在它是空的,但是一会儿我们就会改变它.


#### 提交文件

首先我们创建一个`Main.java`文件

```
touch Main.java
```
然后输入一部分内容

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}
```
然后以同样的方式在准备一个README.md文件
```
touch README.md
```

向文件中输入以下内容
```
this is my first java project!
```

现在add并且commit他们到仓库
```
git add .
git commit -m 'Initial Commit'
```

### 模型的创建
现在看上去没啥特殊的,现在我们回过头来在看看`.git/objects`目录下已经存在了一些子文件夹以及文件了

```
.git/objects
|-- 84
|   -- 705622ee44f2afbb21087ca7d81fda01fccded
|-- 95
|   -- fc1236534b6f73930367f02895467040f47d4a
|-- b0
|   -- 81e51f448387e72a3e3551ba8610eedc172e60
|-- f1
|   -- a8b89f50a2fd8287578daa2b0374adf3cad8aa
|-- info
|-- pack
6 directories, 4 files
```
需要注意的是在你的电脑上目录和文件名称和我这里是不一样的.

#### blob object的创建
在`.git/objects`下我们注意到每个目录的名称只有2个字符长度,Git为每个对象生成一个40个字符的校验和（SHA-1）哈希，该校验和的前两个字符用作目录名，另外38个字符用作文件（对象）名。
当我们提交一些文件时,git创建的第一类对象是**blob object**,在我们的例子中是两个,每一个**blob object**对应我们提交的每一个文件:

![blob object](/images/understanding-git/data-model-blob-object.png)

blob object包含文件的快照以及拥有文件校验和.

#### tree object的创建
git创建的另外一种对象是`tree object`,在我们的例子中只有一个,它包含我们项目中所有文件的列表,其中包含分配给它们的blob object的指针(这就是git如何将文件与blob object相关联)

![tree object](/images/understanding-git/data-model-tree-object.png)

#### commit object的创建

最后git还创建了一个commit object,该对象具有指向它的tree object的指针(以及一些其他信息)

![commit object](/images/understanding-git/data-model-commit-object.png)

这个时候在来看以下objects目录下的结构就清晰多了
```
.git/objects
|-- 84
|   -- 705622ee44f2afbb21087ca7d81fda01fccded
|-- 95
|   -- fc1236534b6f73930367f02895467040f47d4a
|-- b0
|   -- 81e51f448387e72a3e3551ba8610eedc172e60
|-- f1
|   -- a8b89f50a2fd8287578daa2b0374adf3cad8aa
|-- info
|-- pack
```

### 验证模型的准确性

上面画出了模型图,但是你以为我这个模型是自己猜的吗?我又是如何确定哪个是blob object?哪个是tree object?哪个是commit object的呢?接下来就是见证奇迹的时刻了.

使用`git log`命令我们可以查看我们的提交历史
```
commit f1a8b89f50a2fd8287578daa2b0374adf3cad8aa (HEAD -> master)
Author: zhu.yang <zhu.yang@xxx.com>
Date:   Tue Jan 8 10:12:06 2019 +0800
    Initial Commit
```

根据我们前面说的命名约定,我们可以在objects中发现`f1a8b89f50a2fd8287578daa2b0374adf3cad8aa`这个对象.
想要查看文件内容我们不能简单的使用`cat`命令,因为这些不是纯文本文件,但是好在git给我们提供了一个cat-file命令

```
git cat-file commit f1a8b89f50a2fd8287578daa2b0374adf3cad8aa
```
可以通过它获取到commit object中的内容

```
tree 95fc1236534b6f73930367f02895467040f47d4a
author zhu.yang <zhu.yang@xxx.com> 1546913526 +0800
committer zhu.yang <zhu.yang@xxx.com> 1546913526 +0800
Initial Commit
```

从上面可以看到commit指向tree object并且我们可以使用`git ls-tree`命令来检查下其中的内容
```
git ls-tree 95fc1236534b6f73930367f02895467040f47d4a
```
正如我们说预料的一样,其中包含了指向blob object的文件列表

```
100644 blob 84705622ee44f2afbb21087ca7d81fda01fccded    Main.java
100644 blob b081e51f448387e72a3e3551ba8610eedc172e60    README.md
```

如果想要查看Main.java中的内容则使用`cat-file`命令即可

```
git cat-file blob 84705622ee44f2afbb21087ca7d81fda01fccded
```
我们可以看到其中返回了Main.java文件的内容
```
public class Main {
        public static void main(String[] args) {
                System.out.println("Hello World");
        }
}
```
上面就是当我们创建并提交了一些文件的时候就会发生的事情.同时也验证了我们模型的准确性.

#### 修改文件时模型的改变

现在我们修改一下main.java然后重新提交一下

![新增blob object](/images/understanding-git/data-model-when-create-new-blob.png)

正如我们看到的一样,git以快照的方式为`Main.java`新建了一个blob object,由于`README.md`没有被修改,因此不会为其创建新的blob object.而且**git会重用现有的blob object.**

现在，当git创建一个tree object时，分配给`Main.java`的blob指针会被更新，并且分配给`README.md`的blob指针将保持与前一个提交树中的相同。

![新增tree object](/images/understanding-git/data-model-when-create-new-tree.png)

在最后,git创建一个commit object并指向它的tree object.同时还有一个指向它的父提交对象的指针(每个提交除了第一个提交至少还有一个父提交)
![新增commit object](/images/understanding-git/data-model-when-create-new-commit.png)

到现在为止我们已经知道了git是如何处理文件的新增以及编辑的,唯一还遗留的就是如何处理删除了,我们先删除Main.java:

![删除文件](/images/understanding-git/data-model-when-delete.png)

请注意上图中红色的连线,我们发现删除同样也是非常简单,只需要删除tree object指向blob object的指针即可.在这种情况下我们在新的提交中删除了Main.java,因此我们的提交的树对象不再具有指向表示Main.java的blob object的指针.

#### 模型对文件夹的处理
我们提供的这个数据模型还有一个附加功能-tree object是可以被嵌套的(它们可以指向其他树对象),你可以这样想:每个blob object代表一个文件,每个树对象代表一个目录,所以如果我们有嵌套目录,我们就有嵌套的tree object.

由于上面的图已经是提交多次结果画出来的了,再在上面的基础上画结构就不是那么清晰了,这次我重新初始化一个仓库来演示,现在该仓库下存在存在的数据如下:
```
|-- README.md
`-- app
    `-- user.json
```
然后提交,最后可以看到如下的数据模型

![tree object嵌套](/images/understanding-git/data-model-dir.png)

Git使用blob object以及tree object来重现项目的文件夹结构.到这里我相信你肯定对git的数据模型有了较为深入的了解,它真的是很简单,我相信基于它再去学习Git一定会是事半功倍.

### 总结
1. 创建一个提交的时候git会新增blob object,tree object,commit object并会形成链路图
2. 嵌套的tree object用来表示文件夹
3. git从复用blob object
4. 除了第一个提交之外,每一个提交都有一个父提交