---
title: 深入简出git--索引
date: 2019-01-21 14:28:42
tags: Git
keywords: Git,index
---

从git的角度来看,文件的修改涉及到以下三个区域:工作目录,stage区(暂存区)以及本地仓库.
![git区域](/images/understanding-git/git-area-when-update.png)

当我们对我们的项目做了一些修改(新增文件,删除文件,修改文件等),我们处理的就是我们的工作目录.这个目录是存在于我们电脑的文件系统上的.所有的修改都会保留在工作目录直到我们把它们加入到暂存区(通过git add命令).
<!--more-->

暂存区这是对下一次提交最好的表示方式,当我们执行`git commit`,git会获取暂存区中的修改,并将这些修改作为下一次的提交内容.暂存区的一个实际作用就是允许你调整你的提交,你可以向暂存区新增和删除修改直到你对你下一次的提交满意,这个时候你就可以用`git commit`提交你的内容了.

在提交修改后,它们就会进入`.git/objects`目录,在其中被保存为commit,blob以及tree objects(参考[数据模型](https://generalthink.github.io/2019/01/09/understanding-git-data-model/)那一篇文章)

把暂存区认为是一个存储修改的真实区域并不准确,git没有专门的stage目录来存放这些文件的修改(blobs),git有一个名为index的文件来跟踪这三个区域的修改:工作目录,暂存区以及本地仓库

当我们添加修改到暂存区的时候,git会更新index文件中的信息,并且创建一个新的blob object,然后将它们放到与之前提交的记录所产生的其他blob相同的.git/objects目录中.

接下来我们就通过一个正常的git流程来演示下git如何使用的index

首先在我们的仓库里面有master以及feature两个分支,如果我们执行下面的命令,会有三件事情发生
```
git checkout feature
```

第一,git会移动HEAD指针来指向feature分支,为了更加便于理解,我们只显示功能分支的最后一次提交

![HEAD指向feature](/images/understanding-git/index-object.png)

第二,git将获取feautre分支指向的提交内容并将其添加到索引中

![添加到索引](/images/understanding-git/index-when-checkout.png)

我们注意到index是一个文件而不是目录,所以git是没有往其中存储内容的,git只是存储我们仓库中每个文件的信息而已,类似于上面这样
	+ mtime : 上次更新时间
	+ file : 文件名称
	+ wdir : 工作目录中文件版本
	+ stage : index中文件版本
	+ repo : 仓库中的文件版本


文件版本以校验和来标识,如果两个文件有相同的校验和,那么它们就有一样的内容以及版本.

最后,git会将你的工作目录和HEAD指向的内容相匹配(它将使用树和blob对象重新创建项目目录的内容)

![索引和工作目录](/images/understanding-git/index-data-workspace.png)

所以,当你使用checkout的时候,工作目录,暂存区以及仓库都是相同的版本
我们来看看当我们编辑Main.java的时候会发生什么?

![工作目录变化](/images/understanding-git/index-when-workspace-change.png)

现在仅仅只影响了我们的工作目录,但是我们运行下面的命令的时候
```
git status
```
git 首先会更新index文件中Main.java的工作目录的版本

![index中变化](/images/understanding-git/index-find-workspace-change.png)

然后我们看到index.php在工作目录和暂存区有不同的版本

![index中变化](/images/understanding-git/index-find-stage-change.png)

然后git会提示我们
```
On branch feature
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)
modified:   Main.java
no changes added to commit (use "git add" and/or "git commit -a")

```
这就表明工作目录的修改不在暂存区中(那么下一次的提交就不会包含index.php的修改).

所以,执行以下命令将Main.java加入到暂存区
```
git add Main.java
```

执行了上面这条命令,就又会发生两件事儿,第一,git会为index.php创建一个blob object然后存储在.git/objects目录下,第二,会再次更新index文件

![执行git add命令之后](/images/understanding-git/index-sync-stage.png)

这个时候我们再次执行命令
```
git status
```
git会发现index.php的暂存区的版本和工作目录版本一致,但是和仓库的版本不一致

![index中变化](/images/understanding-git/index-find-respo-change.png)

所以git就告知我们
```
On branch feature
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)
modified:   Main.java
```
证明index.php已经在暂存区,但是还没有提交到仓库.现在我们就可以提交我们的修改了
```
git commit -m "add some code to Main.java"
```
git会做下面几件事儿:
	1. 新增commit object和tree object,并把它们和执行git add时创建的blob object连接起来
	2. 移动feature的指针到新的commit object
	3. 更新index

![创建commit object](/images/understanding-git/index-create-object.png)

好啦,现在我们的Main.java在所有区域都有相同的版本了.

无论执行 `git add`还是`git commit`index文件都会变更,这也更好的证明了我们上述模型,当然index文件中的内容肯定没有那么清晰,它是一个二进制文件,如果想要查看它的内容就需要借助其他工具来实现


上面就是关于git index的原理了,现在回过头来看发现其实并不复杂,但是对于我们理解在一些在index上操作的命令(add,checkout,revert,commit,add...)却是至关重要的


### 参考文章
[https://www.youtube.com/watch?v=xbLVvrb2-fY](https://www.youtube.com/watch?v=xbLVvrb2-fY)