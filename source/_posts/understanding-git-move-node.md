---
title: 深入浅出git(五)——自由的修改提交记录
date: 2019-01-25 16:15:59
tags: Git
keywords: Git,HEAD,分离式HEAD,revert,reset,cherry-pick
---


[上篇文章](https://generalthink.github.io/2019/01/23/understanding-git-merge-and-rebase/)讲了merge和rebase,我们已经可以在commit object构成的图(当然我更愿意把它看成一棵树)上面进行分支的合并了,在图上我们可以新增节点(git commit),合并节点(merge或者rebase),这篇我们就来讲解下移动和删除节点。

在讲移动和删除之前，我们先来认识下HEAD的分离。

### 分离的HEAD

我们都知道,HEAD是指向当前分支的,而分离的HEAD就是让其指向了某个具体的提交记录而不是分支名。

现在我本地的提交记录是这样的

![git log](/images/understanding-git/move-and-delete/git-log-status.gif)

当我们执行`git checkout  062704b1c3a814dfd95695aba3684c22e3f3fa85`之后HEAD就处于分离状态。

![分离HEAD](/images/understanding-git/move-and-delete/git-detached-head.gif)

![分离HEAD](/images/understanding-git/move-and-delete/git-detached-head-structure.gif)

<!--more-->

### 移动节点

我们可以通过指定提交记录hash的方式移动指针的位置(无论是分支还是HEAD),然而实际中并没有那么直观的图给我们看，就不得不使用`git log`来查看提交记录的hash，然而hash又比较长，幸好git对hash的处理比较智能，我们只需要提供唯一标识提交记录的前几个字符就可以了，因此我们可以只输入`git checkout 0627`就可以检出提交记录了。

通过hash值来移动节点显然并不方便，所以git提供了相对引用，这样我们就可以从一个易于记忆的地方(比如bugFix分支或者HEAD)开始计算。


#### 相对引用

相对引用很给力，常用的两种用法
1. 使用`^`向上移动1个提交记录
2. 使用`~num`向上移动多个提交记录

![移动一个节点](/images/understanding-git/move-and-delete/git-move-one.gif)
![移动多个节点](/images/understanding-git/move-and-delete/git-move-multi.gif)

当我们执行`git checkout master^`的时候，我们的HEAD就指向了上一个提交记录(当前记录的是一个记录)，注意这里移动的是提交记录(commit object)，同理移动多个记录也是一样的。


使用相对引用最多的就是移动分支，我们可以命令直接让分支指向另外一个提交。

![移动分支到指定节点](/images/understanding-git/move-and-delete/git-move-branch.gif)

现在master分支就指向了第一个提交，需要注意的是不能在当前分支操作当前分支的移动，否则你会有这样的错误
```
fatal: Cannot force update the current branch.
```
完成移动之后并不会切换分支，仍然处于之前的分支。

#### 任意移动

如何能将提交树的commit object任意的移动?让我们的修改可以更加的随意，`git cherry-pick`就能做到。
现在我们想把bugFix分支上C2,C4的提交记录移动到master分支上，只需要执行
```
git cherry-pick C2 C4
```
这个命令可以"复制"提交节点并在当前分支做一次完全一样的新提交

![git cherry-pick](/images/understanding-git/move-and-delete/git-cherry-pick.gif)


### 回退代码

有的时候我们的代码提交错了，但是已经提交到git上去了，我想要回退怎么办？还好git提供了两种方法用来撤销变更----`git reset`以及`git revert`。

#### git reset
`git reset`通过把分支记录回退几个提交记录来实现撤销 改动，其实就是移动在图上的指针。

![git reset](/images/understanding-git/move-and-delete/git-reset.gif)

#### git revert

![git revert](/images/understanding-git/move-and-delete/git-revert.gif)

我们本来是要撤销C2提交的，但是为什么还多了一个C2'提交呢？这是因为新提交记录C2'引入了更改--这个更改又是用来撤销C2这个提交的，也就是说C2'的状态于C1是相同的。
revert之后就可以把更改push到远程仓库与别人分享了。


