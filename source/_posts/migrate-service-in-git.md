---
title: Git中仓库拆分的两种方式
date: 2024-05-30 15:46:33
tags: [git, filter-repo, subtree]
---

最近要把项目中的子模块单独拆分为一个项目,并且移动到新的仓库地址，同时需要保留所有提交记录以及所有分支(包括未上线-未合并到master分支)的代码,我这里总结了2种方式以供大家参考。

### 项目现状

```
bv_sc_server
|───bvpro-modules
	│  ├─bvpro-job
	│  │  ├─src
	│  │  │  └─main
	│  │  │      ├─java
	│  │  │      │  └─com
	│  │  │      │      └─bvpro
	│  │  │      │          └─job
	|  ├─bvpro-file
	│  │  ├─src
	│  │  │  └─main
	│  │  │      ├─java
	│  │  │      │  └─com
	│  │  │      │      └─bvpro
	│  │  │      │          └─file	
```
<!-more-->

项目中有很多子模块，我需要将他们单独独立出来放到放到一个新的仓库,同时只保留各自仓库的代码。 

比如我有一个分支feature/migrate-data,它同时修改了bvpro-file以及bvpro-job的代码，那么迁移过去的效果希望是
新的bvpro-file和bvpro-job仓库都有这个分支，同时也只包含当前仓库的代码，而不再想之前一样混杂着多个模块的代码。


因此调研了下发现有两种方案可以达到这样的目的，一种是git subtree, 一种是git filter-repo。 

### git subtree

需要拆分的仓库叫做 bv_sc_server, 现在需要拆分的模块是 bv_sc_server/bvpro-modules/bvpro-job, 拆分后的新仓库叫做bvpro-job


1. 先在代码服务器(比如gitlab)上新建一个空的仓库， 比如bvpro-job

2. 在本地文件夹clone刚创建的新仓库, 和bv_sc_server在同一个目录下

```
	git clone http://localhost:8090/sc_group/bvpro-job.git
```

3. 进入bv_sc_server这个仓库的目录下进行拆分,当前处于master分支(这个分支是需要迁移的分支)

```
git subtree split -P bvpro-modules/bvpro-job -b feature/split-bvpro-job
```

4. 进入到bvpro-job这个新仓库,执行以下命令: 
	+ cd ../bvpro-job
	+ git pull ..\bv_sc_server feature/split-bvpro-job
	+ git remote remove origin
	+ git remote add origin http://localhost:8090/sc_group/bvpro-job.git
	+ git push --set-upstream origin --all

到这里我们就把bv_sc_server中bvpro-job模块的master分支代码迁移到了新的仓库，但是其他分支还没有迁移过去,所以这里我们重复执行下操作


这里为了不相互影响，我们将本地bvpro-job目录删除，然后重新新建一个空白bvpro-job目录。然后在重复执行上面的命令,这里为了演示方便我就将命令写到一起。

```
cd bvpro-job
git clone http://localhost:8090/sc_group/bvpro-job.git
cd ../bv_sc_server
#切换到需要迁移的分支
git checkout feature/SCA-5034_AutoApprovalForTcAndSc
git subtree split -P bvpro-modules/bvpro-job -b  split/SCA-5034_AutoApprovalForTcAndSc
cd ../bvpro-job
git pull ..\bv_sc_server split/SCA-5034_AutoApprovalForTcAndSc
git remote remove origin
git remote add origin http://localhost:8090/sc_group/bvpro-job.git
# 新clone的仓库默认是master分支(也可能是main), 然后推送到远端的feature分支
git push --set-upstream origin master:feature/SCA-5034_AutoApprovalForTcAndSc
```

这样就完成了feature/SCA-5034_AutoApprovalForTcAndSc分支的迁移了,这样迁移也只保留了bvpro-job有关的代码，如果这个分支在以前的其他模块也有代码,那么也要按照这样的方式进行迁移，同时有多少个分支需要迁移就需要执行多次上面的命令。

可以发现这样迁移的效率是很低的，要是分支或者提交记录很多一天的时间都耗在这上面了。当然如果你只想要迁移master分支代码，这种方式也是很不错的，关键是这个命令是git自带的。


### git filter-repo

这个命令并不是git自带的,但是它也很赫赫有名，毕竟官方都推荐使用它进行迁移。

使用它是有限制的，首先python版本要在3.5以上,git版本要在2.2以上。因为我使用的是windows,所以下面演示windows上的安装方式

1. 安装git-filter-repo
```
python -m pip install --user git-filter-repo
```

记录下安装的地址，然后配置环境变量，然后就可以使用git filter-repo这个命令了。

2. git clone bv_sc_server项目,默认master分支,最好是最新的,这样子命令运行错了也不影响你的开发

```
git clone http://localhost:8090/sc_group/bv_sc_server.git bvpro-job

```

3. 拆分

```
cd bvpro-job

# 只保留bvpro-modules/bvpro-job这个路径的代码
git filter-repo --path bvpro-modules/bvpro-job

# 将bvpro-modules/bvpro-job这个目录提升为根目录
git filter-repo --subdirectory-filter bvpro-modules/bvpro-job

# 上面两句可以合并成下面这句，还可以指定要保留的分支
git filter-repo --path bvpro-modules/bvpro-job --subdirectory-filter bvpro-modules/bvpro-job --branches <保留的分支名称>

```
上面的命令执行完成后,bv_sc_server这个目录下的代码就只有以前bvpro-job的代码了


4. 推送

```
git remote add origin http://localhost:8090/sc_group/bvpro-job.git

git push --set-upstream origin --all
```
这样子就将bvpro-job这个子模块的所有代码，commit以及分支都迁移到了新的仓库，不用再想subtree一样针对特定的分支反复操作了， 所以我是推荐使用这个命令的。