---
title: Git_远程
date: 2020-06-14 21:17:04
categories: Git
tags: [notes, git]
---
参考资料：https://learngitbranching.js.org/         https://dev.to/lydiahallie/cs-visualized-useful-git-commands-37p1

## 远程分支

远程分支位于本地仓库中，反映了最后一次与远程仓库通信时的状态。

远程分支的命名规范是`<remote name>/<branch name>`，默认的远程仓库名是`origin`.

远程分支在成为`checkout`的目标时会自动进入分离HEAD状态，这是为了让远程分支不能成为直接操作的对象。

`git clone`时会自动为远程仓库中的每一个分支，在本地仓库中创建一个远程分支并加以关联。

<!-- more -->

`git fetch origin master`从远程仓库下载本地缺失的提交记录并移动本地的`origin/master`远程分支指针；本地仓库不会发生改变，即`master`分支不变，文件不会被修改。

不带参数的`git fetch`将各commit下载到对应的分支里。

![](/images/Git_remote/fetch.gif)



`git pull`相当于`git fetch`加上`git merge`，效果是与远程仓库同步。

![](/images/Git_remote/pull.gif)



`git push <remote> <place>`(如`git push origin master`)表示找到本地master分支中的所有提交，向远程仓库origin的master分支中添加其中不存在的提交.`<place>`参数指定了要同步的两个分支的位置。

一般的命令是`git push <remote> <source>:<destination>`，适用于本地远程分支与关联的远程仓库中的分支不同名的情况。如果`<source>`留空，会删除远程仓库的`<destination>`分支。

### 当本地仓库基于过时的远程仓库时

`git pull --rebase`相当于`git fetch`加上`git rebase origin/master`，效果是获取远程仓库的提交记录后，将当前分支的工作转移到最新的远程分支下。

而`git pull`相当于`git fetch`加上`git merge origin/master`，效果与前者一样，不过会引入一次新的合并而非移动当前分支的工作。


