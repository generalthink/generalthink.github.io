---
title: 深入浅出git(四)——merge和rebase
date: 2019-01-23 17:47:11
tags: Git
keywords: Git,merge,rebase,merge vs rebase,mrege和rebase的区别
---

经过前面理论的学习,相信对git的模型已经有了一个比较深入的认识,在讲解常用开发命令之前,先看下git整体操作流程

![git流程](/images/understanding-git/merge-and-rebase/git-process.png)

我们按照一个正常的开发流程来学习,我们现在要开始开发了,首先我们需要把远程仓库同步到本地

```
git clone https://github.com/generalthink/git_learn.git
```
运行完上面的命令,在我们的当前工作目录就会有一个git_learn的文件夹,其中存在.git目录(git维护仓库的基本)以及和远程仓库一样的文件,现在我们的代码环境就和远程仓库一致了,我们就可以开始我们的开发流程了。

<!--more-->

### 开发流程

一般来说，无论是开发新功能(feature)还是修改bug(issue)我们都不会直接在主干上进行开发，而是会在分支上进行开发，然后验证无误之后在同步到master。
现在突然有了一个bug我们要新开一个分支来解决这个bug,那么我们应该怎么做呢?

首先，新建一个bugFix分支,并切换到这个分支
```
git branch bugFix
git checkout bugFix
或者
git checkout -b bugFix
```

![git checkout](/images/understanding-git/merge-and-rebase/git-checkout.png)

新建一个分支,只是多了一个指针指向当前最新的提交而已,后面所有的提交都基于当前的分支,星号代表当前分支。

然后，找到bug的原因，修改对应的文件，现在我们修改了几个文件,需要提交将它加入到我们本地仓库,同步到远程仓库会在后面的文章讲解。

1. 加入修改的文件到暂存区

```
git add *.java
```
上面的操作实际上是使文件工作目录和暂存区的版本一致,但是现在和仓库的版本还不一致.想要查看具体存在哪些文件需要提交可以使用`git status`查看。

2. 提交文件到本地仓库

```
git commit -m "fix bug"
```
`commit`命令实际上是让index中文件在暂存区和仓库的版本保持一致,经过这个步骤,工作目录,暂存区以及仓库的版本都是一致了，下面的图是提交前后commit objects(可以查看数据模型那篇文章)的变化。

![git commit](/images/understanding-git/merge-and-rebase/git-commit.png)


你在修改bug的同时,其他人已经往主干分支上提交了其他功能的代码或者你本地本来就存在两个不同的分支,所以修复了这个bug之后,我们的仓库看上去是这样的

![结构图](/images/understanding-git/merge-and-rebase/git-two-branch.png)



### 合并分支
这个bug修复后,被测试验证通过了,然后接下来想要把这个分支的代码合并到主干分支(或者合并两个不同的分支),常用的合并方式有2种

#### git merge

将bugFix合并到master分支上

```
//切换到master分支
git checkout master

git merge bugFix
```
![git-merge-bugFix](/images/understanding-git/merge-and-rebase/git-merge-bugFix.png)

看到了没有,master指向了一个拥有两个父节点的提交记录,如果从master开始沿着箭头向上看,在到达起点的路上会经过所有的提交记录,这意味着 master 包含了对代码库的所有修改。

这个时候如果你还想把master分支合并到bugFix分支也是可以的
```
git checkout bugFix
git merge master
```
![git-merge-master](/images/understanding-git/merge-and-rebase/git-merge-master.png)

因为 master 继承自 bugFix，Git 什么都不用做，只是简单地把 bugFix 移动到 master 所指向的那个提交记录。
现在所有提交记录的颜色都一样了，这表明每一个分支都包含了代码库的所有修改！

#### git rebase

第二种合并分支的方法是 git rebase。Rebase 实际上就是取出一系列的提交记录，“复制”它们，然后在另外一个地方逐个的放下去。
Rebase 的优势就是可以创造更线性的提交历史.
```
git rebase master
```

![git-rebase-bugFix](/images/understanding-git/merge-and-rebase/git-rebase-bugFix.png)

现在 bugFix 分支上的工作在 master 的最顶端，同时我们也得到了一个更线性的提交序列。
注意，提交记录 C3 依然存在（树上那个虚线节点），而 C3' 是我们 Rebase 到 master 分支上的 C3 的副本。
现在唯一的问题就是 master 还没有更新，下面咱们就来更新它吧

```
git checkout master
git rebase bugFix
```

![git-rebase-master](/images/understanding-git/merge-and-rebase/git-rebase-master.png)

由于 bugFix 继承自 master，所以 Git 只是简单的把 master 分支的引用向前移动了一下而已