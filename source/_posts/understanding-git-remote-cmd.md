---
title: 深入浅出git——远端命令
date: 2019-01-29 11:21:02
tags: Git
keywords: Git,fetech,pull,push
---


前面的文章讲的命令都是操作本地仓库的,我相信可以应付大部分的开发状况,提交代码到本地仓库已经不是问题了,那么这次我们就来看看如何和远程仓库进行对接。
### git clone --- 克隆远端代码到本地

当我们进行开发的时候，开发流程是这样的：首先将远程仓库(中央仓库)的代码clone到本地，在本地进行开发，开发完成之后将代码提交到远程仓库。

远程仓库并不复杂,实际上它们只是你的仓库在另外一台计算机上的拷贝,我们可以通过网络和这台计算机通信--也就是增加或是获取提交记录。我们先通过命令将远端仓库clone到本地
```
git clone https://github.com/generalthink/git_learn.git
```
执行命令之后git仓库就从远端clone到本地了,此时本地和远端的代码一致。执行了这个命令之后我们本地有什么变化呢？
先查看我们现在存在哪些分支
```
$ git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/master
```
<!--more-->

当我们执行`git clone`的时候， Git 会为远程仓库中的每个分支在本地仓库中创建一个远程分支（比如 origin/master）。然后再**创建一个跟踪远程仓库中活动分支的本地分支**，默认情况下这个本地分支会被命名为 master。

你可能注意到了我们除了本地的master分支,还多了origin/master的分支,这种类型的分支就叫远程分支,它反映了远程仓库在你上次和它通信的状态。还记得index那篇文章吗?index文件中记录了工作目录,暂存区,本地仓库的版本用于跟踪文件状态,那么远程仓库的状态由谁来维护呢?没错就是这个origin/master分支。

需要注意的是远程分支有一个特别的属性,当我们检出时,自动进入分离HEAD状态(此种状态下提交并不能影响origin/master分支).git这么做是出于不能直接在这些分支上进行操作的原因, 你必须在别的地方完成你的工作。
![分离HEAD](/images/understanding-git/remote-cmd/detached-origin-head.gif)

远程分支的命令规范是这样的:`<remote name>/<branch name>`,当我们使用git clone某个仓库的时候,git已经帮我们把远程仓库的名称设置为origin了。

可以使用下面的命令来查看远程库对应的简短名称

```
$ git remote -v
origin  https://github.com/generalthink/git_learn.git (fetch)
origin  https://github.com/generalthink/git_learn.git (push)
```

上面我们把远端仓库同步到了本地,远端和本地的代码就是一致的了(本地仓库中两个分支都指向的最新的提交记录)。

### 分支跟踪

当我们将本地master分支的代码push到远程的master分支(同时会更新远程分支origin/master)的时候，我们只需要执行`git push`就可以了，就好像git知道我们它们是关联起来的！

其实master和origin/master的关联关系是由分支的"remote tracking"属性决定的，master被设定为跟踪origin/master -- 表示master指定了推送的目的地以及拉取后合并的目标。

可以让任意分支跟踪 origin/master, 然后该分支会像 master 分支一样得到隐含的 push 目的地以及 merge 的目标。 这意味着你可以在分支 bugFix上执行 git push，将工作推送到远程仓库的 master 分支上。我们可以通过下面的两种方法创建一个bugFix的分支，它跟踪远程分支origin/master

+ `git checkout`

	```
	git checkout -b bugFix origin/master
	```

+ `git branch` 

	需要保证bugFix分支已经存在
	```
	git branch -u origin/master bugFix

	如果当前就在bugFix分支上，命令可以优化成为

	git branch -u origin/master
	```

这样bugFix就会跟踪origin/master了,当我们推送代码到远端的时候就可以不用指定目的地了，直接执行`git push`就可以将bugFix分支的代码推送到远端的master分支了。

通过`git branch -vv`命令可以查看本地分支关联的远程分支的对应关系

```
$ git branch -vv
* bugFix     215d0ff [origin/master] add bugFix.md
  foo        e2240d6 [origin/master: behind 2] add foo.md
  master     7b7adf6 [origin/master: behind 5] Revert "bugFix"
  newFeature 3136c72 [origin/master: behind 3] add test2.md
```

当你通过上面的命令设置了跟踪关系之后执行`git pull`的时候你可能会有这样的报错信息：
```
fatal: The upstream branch of your current branch does not match
the name of your current branch.  To push to the upstream branch
on the remote, use
    git push origin HEAD:master
To push to the branch of the same name on the remote, use
    git push origin newFeature
To choose either option permanently, see push.default in 'git help config'.
```

这全是因为`git config push.default`设置，默认是simple(从git 2.0开始)，这表示当本地分支和远端分支的名称不一样的时候git会拒绝提交。为了让其允许push到它跟踪的分支，需要重新设置这个参数
```
git config --global push.default upstream
```
`--global`只改变当前git仓库的配置。关于push.default有哪些值可以通过`git help config`命令查看。

设置完成之后，在执行`git push`命令就可以直接将bugFix分支的内容提交到master分支上了。

```
$ git push
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 8 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (2/2), 327 bytes | 327.00 KiB/s, done.
Total 2 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To https://github.com/generalthink/git_learn.git
   e2240d6..215d0ff  bugFix -> master

```

### 开发流程

上面我们的仓库已经和远端一致之后，我们就可以开发了，现在我们要修改一个bug,做法就是本地新建一个bugFix(命名规范各个公司是不一样的，一般配合bts工具)分支，
然后在这个分支上面修改，修改完成之后将修改提交到线上服务器，然后线上jekins会自动跑一些脚本，验证你提交的代码，或者检测冲突，有冲突就需要合并。等到一切没有问题之后就可以合并master去了，当然我们自己开发是没有这么复杂的，因此我们就通过直接将bugFix分支的代码推送到远端master分支就可以了

### 提交代码到远程仓库

`git push`命令负责将我们的变更上传到指定的远程仓库，现在直接将我们的代码推送到远程分支

```
$ git push origin bugFix:master
To https://github.com/generalthink/git_learn.git
! [rejected]        bugFix -> master (fetch first)
error: failed to push some refs to 'https://github.com/generalthink/git_learn.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally。 This is usually caused by another repository pushing
hint: to the same ref。 You may want to first integrate the remote changes
hint: (e。g。, 'git pull 。。。') before pushing again。
hint: See the 'Note about fast-forwards' in 'git push --help' for details。
```
执行命令发现报错了，为什么会这样呢？是因为你在开发的过程中，你的同事也在开发，并且他开发的代码已经合并到了主干上，这个时候你本地的代码就不是最新的了,这个时候如果你往远端push代码,那么git就会给你抛出这个错误提示。

此时，远端和本地分支的情况是这样的。
![远端和本地分支对比](/images/understanding-git/remote-cmd/git-rep-compare.gif)

可以看到远端master节点和本地的origin/master指向的并不是同一个commit object,而我们执行的git push命令显然不能智能的帮助我们合并。此时我们应该先同步远端更改到本地，合并这些修改，然后在push到主干。

#### git fetch -- 同步代码到本地

下面的命令用来和远端进行通信,把远端的代码先同步到本地
```
git fetch origin master
```
![get fetch](/images/understanding-git/remote-cmd/get-fetch.gif)

git fetch完成了仅有的但是特别重要的两步
	1. 从远程仓库下载本地仓库中缺失的提交记录
	2. 更新远程分支指针(如 origin/master)

现在本地仓库的远程分支更新成了远程仓库相应分支最新的状态。它通常通过互联网(http://或者git://协议)与远程仓库通信。
需要注意的是**git fetch 并不会改变你本地仓库的状态。它不会更新你的 master 分支，也不会修改你磁盘上的文件。**

#### 合并代码

现在我们已经获取到了远程的数据,只需要将这些变化更新到我们的工作目录中就可以了,我们只需要像合并本地分支那样来合并远程分支就可以了,我们可以通过以下三种方式来完成合并
	1. git cherry-pick origin/master
	2. git rebase origin/master
	3. git merge origin/master


实际上，由于先抓取更新再合并到本地分支这个流程很常用，因此 Git 提供了一个专门的命令来完成这两个操作。它就是我们的 git pull。
```
git pull ===== git fetch;  git merge origin/master

git pull --rebase == git fetch;git rebase origin/master

```
现在本地的远程分支和远程仓库的代码保持了一致,我们终于可以使用服务器`git push origin bugFix:master`提交我们的代码了。

看着还挺不错,现在我们可以开心的使用git工作了,但是需要记住的是git push之前一定要保证要本地的远程指针一定要和远端一致,要不然你就只有等着报错吧。


### 远程命令语法

上面我们看到了和远程仓库交互的命令主要有git fetch/pull/push这个三个,有人经常使用的可能就只有get pull,git push这样的，可能第一次看到`git push orgin bugFix:master`这样的命令很惊奇，所以这里对这几个命令的语法做一些简介，如果有了解过的就可以不用看下面的文章了。

#### git push语法
```
git push <remote> <localPlace:remotePlace>
```
例子：
```
git push origin master:master
```
表示切换到本地的master分支，获取所有提交，再到远程仓库"origin"中找到"master"分支(如果没有会新建一个)，将远程仓库中没有的提交记录添加上去。
通过"localPlace"参数来告诉git提交记录来自master，在推送到远程仓库中的master,后面的两个参数实际上是要同步的两个仓库的位置。
当只指定localPlace的时候remotePlace的值默认是我们跟踪的分支名称(需要注意push.default参数的值),如果当前分支就是你想要提交的分支，那么你可以直接写成`git push`

这里的localPlace和remotePlace按照官方说明是一个refspec，“refspec” 是一个自造的词，意思是 git 能识别的位置（比如分支 bugFix或者 HEAD~1）。


#### git fetch
git fetch和git push的参数及其类似，它们概念相同，只是方向相反(因为你现在是下载，而非上传)
```
git fetch <remote> <remotePlace:localPlace>
```
举个例子
```
git fetch origin master
```
git会到远程仓库的master分支上，然后获取所有本地不存在的提交，放到本地的origin/master上，注意fetch并不会更新本地的非远程分支，而是下载提交记录。

如果想要直接更新本地master分支也不是不可以，运行下面的命令即可
```
git fetch origin master:master
```
**理论上是可以的，但是强烈建议不要那么做。**

当我们只执行`git fetch`不带任何参数的时候，它就会下载所有的提交记录到各个远程分支。

#### git pull

学会了git fetch，那么git pull就很简单了,git pull唯一关注的是提交最终合并到哪里。之前说过
```
git pull == git fetch;git merge
```
那么
```
git pull origin bugFix ====  git fetch orign master; git merge origin/bugFix
```

![get pull](/images/understanding-git/remote-cmd/git-pull-structure.gif)

上图可以看到当前分支是bugFix，执行pull命令之后origin/master的指向改变了，bugFix分支的内容和远端master分支的内容进行了合并，把这条命令拆解为2条来记忆我相信更容易让人理解。

### 总结

git系列写到现在已经结束了，总共写了6篇文章，三篇理论，三篇实际运用。理论是实战的基础，弄明白了理论理解起来更加容易，git将再也不难，当然我也不可能将每个命令都进行细致的讲解，
但是2-8理论在git中同样适用,如果想看更加详细的命令，我相信官方文档才是最好的。

