---
title: 写给新手的 github 设置指南
date: 2019-08-30 12:14:47
tags: github
---

如果你的团队决定在github上管理项目代码,那么下面这些设置你一定要会。

首先进入到仓库的Settings页面,左边会有菜单选项


### 设置谁能提交代码

![设置成员](/images/github/github_setting_member.png)

添加你的team成员账号,这样他们才能向这个仓库提交代码

<!--more-->

### 设置默认分支

开发过程中,我们并不直接向master提交分支,而是会新建一个开发分支develop/1.0,所有人都只能向这个分支提交代码

![设置默认分支](/images/github/github_setting_default_branch.png)


### 设置提交规则

有了分支之后，我们可以针对不同的分支设置不同的提交规则(Settings-->Branches-->add rule)

![设置分支规则](/images/github/github_setting_branch_rule.png)

设置状态检查我们一般会集成jekins，它是CI/CD的主要工具，当我们提交代码的时候会通过WebHooks和Jekins通信，然后在jekins上编译代码，运行其他任务等等。

### 不允许merge代码,只能rebase或者squash

![禁止merge](/images/github/github_setting_forbidden_merge.png)

一般在实践中，merge后的分支会使得git tree很难看，所以我们一般不做merge，直接squash或者rebase

![不允许merge](/images/github/github_no_merge.png)

### 设置pull request模板

在当前project下新建.github文件夹,然后再文件夹下放置pull_request_template.md文件,文件内容如下:

```
## Description: 
-
## Changes:
-
## Test Scope:
-
## Screenshots (optional)

```

然后将此次修改推送到远端并合并此次请求。这样当我们提交pr的时候就出现了下面的模板

![提交pr](/images/github/github_setting_pr_template.png)








