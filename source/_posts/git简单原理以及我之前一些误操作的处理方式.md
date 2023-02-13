---
title: git简单原理以及我之前一些误操作的处理方式
tags:
  - git
typora-root-url: ../../themes/butterfly/source
date: 2020-11-15 15:55:52
description: 简单说一下git的原理，以及这一年多我误操作比如删错分支之类的，谷歌到的解决方案
cover: /blogImg/git的大概流程.png
categories: 后端知识
---

# 什么是git

Git是免费、开源的**分布式版本控制**系统，可以有效、高速地处理从很小到非常大的项目版本管理。

## 版本控制

版本控制是指对软件开发过程中各种程序代码、配置文件及说明文档等文件变更的管理，是软件配置管理的核心思想之一。那些年，我们的毕业论文,其实就是版本变更的真实写照...脑洞一下，版本控制就是这些论文变更的管理

![论文版本](/blogImg/论文版本.png)

### 集中化的版本控制系统

那么，集中化的版本控制系统又是什么呢，说白了，就是有一个集中管理的中央服务器，保存着所有文件的修改历史版本，而协同开发者通过客户端连接到这台服务器，从服务器上同步更新或上传自己的修改。

<img src="/blogImg/集中化的版本控制.png" alt="集中化的版本控制" style="zoom:50%;" />

### 分布式的版本控制系统

分布式版本控制系统，就是远程仓库同步所有版本信息到本地的每个用户。这里分三点阐述：

- 用户在本地就可以查看所有的历史版本信息，但是偶尔要从远程更新一下，因为可能别的用户有文件修改提交到远程。
- 用户即使离线也可以本地提交，push推送到远程服务器才需要联网。
- 每个用户都保存了历史版本，所以只要有一个用户设备没问题，就可以恢复数据了

<img src="/blogImg/分布式的版本控制.png" alt="分布式的版本控制" style="zoom:50%;" />

# **Git的相关理论基础**

## Git的四大工作区域

先复习Git的几个工作区域：

<img src="/blogImg/git的四大工作区域.png" alt="git的四大工作区域" style="zoom:50%;" />

- **Workspace**：你电脑本地看到的文件和目录，在Git的版本控制下，构成了工作区。
- **Index/Stage**：暂存区，一般存放在 .git目录下，即.git/index,它又叫待提交更新区，用于临时存放你未提交的改动。比如，你执行git add，这些改动就添加到这个区域啦。
- **Repository**：本地仓库，你执行git clone 地址，就是把远程仓库克隆到本地仓库。它是一个存放在本地的版本库，其中**HEAD指向最新放入仓库的版本**。当你执行git commit，文件改动就到本地仓库来了~
- **Remote**：远程仓库，就是类似github，码云等网站所提供的仓库，可以理解为远程数据交换的仓库

## Git的工作流程

上一小节介绍完Git的四大工作区域，把git的操作命令和几个工作区域结合起来，个人觉得更容易理解一些，看图：

<img src="/blogImg/git的工作流程.png" alt="git的工作流程" style="zoom:50%;" />

git 的正向工作流程一般就这样：

- 从远程仓库拉取文件代码回来；
- 在工作目录，增删改查文件；
- 把改动的文件放入暂存区；
- 将暂存区的文件提交本地仓库；
- 将本地仓库的文件推送到远程仓库；

## Git文件的四种状态

根据一个文件是否已加入版本控制，可以把文件状态分为：Tracked(已跟踪)和Untracked(未跟踪)，而tracked(已跟踪)又包括三种工作状态：Unmodified，Modified，Staged

![git文件的四种状态](/blogImg/git文件的四种状态.png)

- **Untracked**: 文件还没有加入到git库，还没参与版本控制，即未跟踪状态。这时候的文件，通过git add 状态，可以变为Staged状态
- **Unmodified**：文件已经加入git库, 但是呢，还没修改, 就是说版本库中的文件快照内容与文件夹中还完全一致。Unmodified的文件如果被修改, 就会变为Modified. 如果使用git remove移出版本库, 则成为Untracked文件。
- **Modified**：文件被修改了，就进入modified状态啦，文件这个状态通过stage命令可以进入staged状态
- **staged**：暂存状态. 执行git commit则将修改同步到库中, 这时库中的文件和本地文件又变为一致, 文件为Unmodified状态.

# git的常用命令

## **日常开发中，Git的基本常用命令**

- git clone
- git checkout -b dev
- git add
- git commit
- git log
- git diff
- git status
- git pull/git fetch
- git push

这个图只是模拟一下git基本命令使用的大概流程

![git的大概流程](/blogImg/git的大概流程.png)

### git clone

当我们要进行开发，第一步就是克隆远程版本库到本地

```shell
git clone url  克隆远程版本库
```

### git checkout -b dev

克隆完之后呢，开发新需求的话，我们需要新建一个开发分支，比如新建开发分支dev

创建分支：

```shell
git checkout -b dev   创建开发分支dev，并切换到该分支下
```

### git add

git add的使用格式：

```shell
git add .	添加当前目录的所有文件到暂存区
git add [dir]	添加指定目录到暂存区，包括子目录
git add [file1]	添加指定文件到暂存区
```

有了开发分支dev之后，我们就可以开始开发啦，假设我们开发完HelloWorld.java，可以把它加到暂存区，命令如下

```
git add Hello.java  把HelloWorld.java文件添加到暂存区去
```

### git commit

git commit的使用格式：

```
git commit -m [message] 提交暂存区到仓库区,message为说明信息
git commit [file1] -m [message] 提交暂存区的指定文件到本地仓库
git commit --amend -m [message] 使用一次新的commit，替代上一次提交
```

把HelloWorld.java文件加到暂存区后，我们接着可以提交到本地仓库

```
git commit -m 'helloworld开发'
```

### git status

git status,表示查看工作区状态，使用命令格式：

```
git status  查看当前工作区暂存区变动
git status -s  查看当前工作区暂存区变动，概要信息
git status  --show-stash 查询工作区中是否有stash（暂存的文件）
```

当你忘记是否已把代码文件添加到暂存区或者是否提交到本地仓库，都可以用git status看看

### git log

git log，这个命令用得应该比较多，表示查看提交历史/提交日志，要回滚代码就经常用它看看提交历史。

```
git log  查看提交历史
git log --oneline 以精简模式显示查看提交历史
git log -p <file> 查看指定文件的提交历史
git blame <file> 一列表方式查看指定文件的提交历史
```

### git diff

```
git diff 显示暂存区和工作区的差异
git diff filepath   filepath路径文件中，工作区与暂存区的比较差异
git diff HEAD filepath 工作区与HEAD ( 当前工作分支)的比较差异
git diff branchName filepath 当前分支的文件与branchName分支的文件的比较差异
git diff commitId filepath 与某一次提交的比较差异
```

如果你想对比一下你改了哪些内容，可以用git diff对比一下文件修改差异

### git pull/git fetch

```
git pull  拉取远程仓库所有分支更新并合并到本地分支。
git pull origin master 将远程master分支合并到当前本地分支
git pull origin master:master 将远程master分支合并到当前本地master分支，冒号后面表示本地分支
git fetch --all  拉取所有远端的最新代码
git fetch origin master 拉取远程最新master分支代码
```

我们一般都会用git pull拉取最新代码看看的，解决一下冲突，再推送代码到远程仓库的。

> 有些伙伴可能对使用git pull还是git fetch有点疑惑，其实 git pull = git fetch+ git merge。pull的话，拉取远程分支并与本地分支合并，fetch只是拉远程分支，怎么合并，可以自己再做选择。

### git push

git push 可以推送本地分支、标签到远程仓库，也可以删除远程分支。

```
git push origin master 将本地分支的更新全部推送到远程仓库master分支。
git push origin -d <branchname>   删除远程branchname分支
git push --tags 推送所有标签
```

如果我们在dev开发完，或者就想把文件推送到远程仓库，给别的伙伴看看，就可以使用git push origin dev

### git branch

git branch用处多多呢，比如新建分支、查看分支、删除分支等等

**新建分支：**

```
git checkout -b dev2  新建一个分支，并且切换到新的分支dev2
git branch dev2 新建一个分支，但是仍停留在原来分支
```

**查看分支：**

```
git branch    查看本地所有的分支
git branch -r  查看所有远程的分支
git branch -a  查看所有远程分支和本地分支
```

**删除分支：**

```
git branch -D <branchname>  删除本地branchname分支
```

### git checkout

**切换分支：**

```
git checkout master 切换到master分支
```

### git merge

我们在开发分支dev开发、测试完成在发布之前，我们一般需要把开发分支dev代码合并到master，所以git merge也是程序员必备的一个命令。

```
git merge master  在当前分支上合并master分支过来
git merge --no-ff origin/dev  在当前分支上合并远程分支dev
git merge --abort 终止本次merge，并回到merge前的状态
```

比如，你开发完需求后，发版需要把代码合到主干master分支

## **Git进阶之撤销与回退**

Git的撤销与回退，在日常工作中使用的比较频繁。比如我们想将某个修改后的文件撤销到上一个版本，或者想撤销某次多余的提交，都要用到git的撤销和回退操作。

代码在Git的每个工作区域都是用哪些命令撤销或者回退的呢，如下图所示：

![git的撤销与回退](/blogImg/git的撤销与回退.jpg)

有关于Git的撤销与回退，一般就以下几个核心命令

- git checkout
- git reset
- git revert

### git checkout

如果文件还在**工作区**，还没添加到暂存区，可以使用git checkout撤销

```
git checkout [file]  丢弃某个文件file
git checkout .  丢弃所有文件
```

### git reset

#### git reset的理解

> git reset的作用是修改HEAD的位置，即将HEAD指向的位置改变为之前存在的某个版本.

为了更好地理解git reset，我们来回顾一下,Git的版本管理及HEAD的理解

> Git的所有提交，会连成一条时间轴线，这就是分支。如果当前分支是master，HEAD指针一般指向当前分支，如下：

![reset1](/blogImg/reset1.png)

假设执行git reset，回退到版本二之后，版本三不见了,如下：

![reset2](/blogImg/reset2.png)

#### git reset的使用

Git Reset的几种使用模式

![reset的几种使用模式](/blogImg/reset的几种使用模式.png)

```
git reset HEAD --file回退暂存区里的某个文件，回退到当前版本工作区状态
git reset –-soft 目标版本号 可以把版本库上的提交回退到暂存区，修改记录保留
git reset –-mixed 目标版本号 可以把版本库上的提交回退到工作区，修改记录保留
git reset –-hard  可以把版本库上的提交彻底回退，修改的记录全部revert。
```

先看一个栗子demo吧，代码**git add到暂存区，并未commit提交**,可以酱紫回退，如下：

```
git reset HEAD file 取消暂存
git checkout file 撤销修改
```

再看另外一个栗子吧，代码已经git commit了，但是还没有push：

```
git log  获取到想要回退的commit_id
git reset --hard commit_id  想回到过去，回到过去的commit_id
```

如果代码已经push到远程仓库了呢，也可以使用reset回滚

```
git log
git reset --hard commit_id
git push origin HEAD --force
```

### git revert

> 与git reset不同的是，revert复制了那个想要回退到的历史版本，将它加在当前分支的最前端。

**revert之前：**

![revert之前](/blogImg/revert之前.png)

**revert 之后：**

![revert之后](/blogImg/revert之后.png)

当然，如果代码已经推送到远程的话，还可以考虑revert回滚

```
git log  得到你需要回退一次提交的commit id
git revert -n <commit_id>  撤销指定的版本，撤销也会作为一次提交进行保存
```

### git rebase

rebase又称为衍合，是合并的另外一种选择。

假设有两个分支master和test

```
      D---E test      
     /
 A---B---C---F--- master
```

执行 git merge test得到的结果

```
       D--------E    
      /          \
 A---B---C---F----G---   test, master
```

执行git rebase test，得到的结果

```
A---B---D---E---C‘---F‘---   test, master
```

**rebase好处是：** 获得更优雅的提交树，可以线性的看到每一次提交，并且没有增加提交节点。所以很多时候，看到有些伙伴都是这个命令拉代码：git pull --rebase，就是因为想更优雅，哈哈

### git stash

stash命令可用于临时保存和恢复修改

```
git stash  把当前的工作隐藏起来 等以后恢复现场后继续工作
git stash list 显示保存的工作进度列表
git stash pop stash@{num} 恢复工作进度到工作区
git stash show ：显示做了哪些改动
git stash drop stash@{num} ：删除一条保存的工作进度
git stash clear 删除所有缓存的stash。
```