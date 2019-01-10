---
title: 深入浅出git——分支
date: 2019-01-10 10:00:11
tags: Git
keywords: git branch
---

在开发软件的时候,可能很多人会同时为同一个软件开发功能或者修复bug,但是如果都在主分支来进行开发,引起冲突的概率将会大大增加,而且也不利于维护,如果你同时修改多个bug该怎么办?所幸,git的分支功能很好的帮助我们解决了这个问题,它可以帮助我们同时进行多个功能的开发和版本管理.
请在阅读这篇文章之前,务必先阅读[深入浅出git——数据模型](http://generalthink.github.io/2019/01/09/understanding-git-data-model/),这样才能更好的帮助你理解git中的分支.想知道为什么git中新建一个分支那么快,代价那么小吗?接下来我们就来揭开分支神秘的面纱.
<!--more-->

这次我会只显示commit objects来简化它,并且为了让它更容易理解我会给他们取个别名来代替原本的检验和.所以对于提交记录,我们会得到一个像下面这样的图.
![commit objects](/images/understanding-git/branch-graph.png)

熟悉图论的应该注意到了上面的是一个有向无环图(DAG),这意味着从一个节点开始沿着边的方向不会经过相同的节点.
在我们的例图中可以清晰的发现存在三个不同的分支,我们分别用红色(包含A,B,C,D,E),蓝色(A,B,F,G),以及绿色(A,B,H,I,J)来标记它们

![三个分支](/images/understanding-git/branch-three-branchs.png)

这就是定义分支的一种方式-包含所有的提交列表.但是这不是git使用的方式,git使用更简单更便宜的方式,git只跟踪分支上的最后一次提交,而不是持有某个分支的所有列表并更新它们,只需要知道分支的最后一次提交,然后根据图的有向边就可以获取整个提交列表.例如要定义我们的蓝色分支,只需要知道蓝色分支的最后一次提交是G,如果我们需要蓝色分支包含的所有提交的列表,就从G沿着图有向边遍历即可.
![三个分支](/images/understanding-git/branch-git-branch-list.png)

这就是git管理分支的方式,通过保持执行提交记录的指针即可,接下来我们会进行一个演示.
首先通过`git init`初始化一个空仓库,然后查看.git目录下存在的文件
```
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
|   -- exclude
|-- objects
|   |-- info
|   |-- pack
|-- refs
    |-- heads
    |-- tags
```

这次我们关注`refs`这个子目录,这个地方是git保留分支指针的地儿.当我们没有提交任何东西的时候,`refs`目录下只存在两个空目录,现在我们提交几个文件
```
echo "Hello Java" > helloJava.txt
git add .
git commit -m "Hello Java Commit"
echo "Hello Php" > helloPhp.txt
git add .
git commit -m "Hello Php Commit"
echo "Hello Python" > helloPython.txt
git add .
git commit -m "Hello Python Commit"
```
当我们执行`git branch`的时候我们可以看到下面这样的输出
```
* master
```
意味着我们现在处于master分支上(这个是当我们第一次提交的时候git自动给我们创建的),此时`refs`目录下是这样
```
.git/refs
|-- heads
|   `-- master
`-- tags
```
我们看到refs/heads子目录中有一个文件，它就像我们的分支一样被命名为master,我们使用cat命令查看下文件内容
```
$ cat .git/refs/heads/master
49cd903b2bf247de040118ce60d1931ff587e801
```

而使用`git log`命令我们可以看到我们的提交记录是这样的

```
commit 49cd903b2bf247de040118ce60d1931ff587e801 (HEAD -> master)
Author: zhu.yang <zhu.yang@xxx.com>
Date:   Tue Jan 8 17:48:36 2019 +0800
    Hello Python Commit

commit dd7c1bc9c125067f5658bcc6bc35567d07bc4f35
Author: zhu.yang <zhu.yang@xxx.com>
Date:   Tue Jan 8 17:48:31 2019 +0800
    Hello Php Commit

commit c6bd5c991dbcf9c50bbab682796ab3e06672f5a7
Author: zhu.yang <zhu.yang@xxx.com>
Date:   Tue Jan 8 17:48:30 2019 +0800
    Hello Java Commit

```
从上面可以看出来一个分支仅仅只是一个文本文件,其中记录了这个分支最后一次提交的校验和.也就是指向commit的一个指针

![master分支](/images/understanding-git/branch-master.png)
现在我们新建一个`feature`分支并切换到新建的这个分支上面
```
git checkout -b feature
```
使用tree命令在来看看.git/refs的样子
```
.git/refs
|-- heads
|   |-- feature
|   |-- master
|-- tags
```

同样的我们使用cat命令查看下.git/refs/heads/feature文件的校验和
```
$ cat  .git/refs/heads/feature
49cd903b2bf247de040118ce60d1931ff587e801
```
我们会发现和master文件中的内容一致,现在为止我们没有往feature分支提交任何内容

![两个分支](/images/understanding-git/branch-branch-feature.png)

这就是git创建一个分支那么快以及方便的原因所在,git仅仅只是创建了一个包含最近一次提交校验和的文件而已.

现在我们的仓库里面就有2个分支了,但是git怎么知道我们当前检出的分支是哪个分支呢?这里其实存在一个特殊的指针叫做**HEAD**,它之所以特殊是因为它并不指向具体的commit object,而是指向分支,git使用它来跟踪最近检出的分支.
```
$ cat .git/HEAD
ref: refs/heads/feature
```

![HEAD指针](/images/understanding-git/branch-HEAD.png)

如果我们执行
```
git checkout master
```
然后查看HEAD,会发现当前分支是master,然后HEAD会指向master
```
$ cat .git/HEAD
ref: refs/heads/master
```


![HEAD指针指向当前分支](/images/understanding-git/branch-HEAD-to-master.png)

这就是git的分支模型,很简单但是很重要,了解它有助于理解在这个图上的其他操作(merge,rebase,checkout,revert...)