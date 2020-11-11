---
title: Git_分支
date: 2020-06-14 00:03:15
categories: Tips
tags: [git]
---
参考资料：https://learngitbranching.js.org/         https://dev.to/lydiahallie/cs-visualized-useful-git-commands-37p1

## 分支

### merge

存在两种merge操作，分别为fast-forward(`--ff`)和no-fast-forward(`--no-ff`).

前者默认发生在当前分支没有相对于待合并分支的额外结点时，此时不会发生新的commit，只是向前移动当前分支指针。
<!-- more -->
![](/images/Git_branch/merge_ff.gif)

后者默认发生在其他情况下，此时会产生一个新的commit，其拥有两个父结点，分别位于两个分支中。如果出现冲突还需要手动解决。

![](/images/Git_branch/merge_noff.gif)

### rebase

rebase将当前分支的提交复制到另一分支之后。此过程中，被复制的commit会重新生成hash值.

![](/images/Git_branch/rebase.gif)

interactive rebase可以以当前分支中的commit为参数，并且可以对需要复制的commit进行修改.
![](/images/Git_branch/interactive-rebase1.gif)
![](/images/Git_branch/interactive-rebase2.gif)

#### cherry-pick

`git cherry-pick [commit id]`在当前分支下新增一个commit，其内容与指定的commit相同.
![](/images/Git_branch/cherry-pick.gif)


## HEAD

`HEAD`指针是一个symbolic reference(符号引用，快捷方式)，指向分支名或者一次commit. 通常情况下`HEAD`总是指向分支名，这意味着其指向当前分支最近一次commit.

`git checkout [commit id]`分离头指针使其指向某次commit而非分支名(commit用其hash值指定，且仅需前6位).

`cat .git/HEAD`查看当前HEAD指向.

`git symbolic-ref HEAD`查看当前HEAD指向（仅限HEAD指向分支名时，即HEAD->master->Commit_1）

### 相对引用

使用^表示指定引用的父节点，如`HEAD^` `master^`

使用`~<num>`表示向上移动多次，如`HEAD~3`

配合`git branch -f [branch name] [ptr]`强制移动（非当前）分支指针位置，如`git branch -f master HEAD~3`

## 撤销变更

`git reset [ptr]`将当前分支HEAD移动到指定位置，如`git reset HEAD^`.从新位置到旧位置之间的提交仍存在，处于未加入暂存区的状态；但是如果使用`--hard`参数，这些提交会消失.这种方式往往应用于个人维护的分支.

![](/images/Git_branch/reset-soft.gif)
![](/images/Git_branch/reset-hard.gif)

`git revert [ptr]`以添加一个新commit的形式撤销ptr指向的提交，因此添加新提交后的状态与ptr的父节点相同.这种方式往往应用于多人协作分支，因为它不会导致提交历史的改变.

![](/images/Git_branch/revert.gif)

## 标签

鉴于分支可以轻易地被移动和更改，引入了tag的概念。tag是一个永久指向某个提交记录的标识，不会随着新的提交而移动.`git tag [tagname][ptr]`将名为tagname的tag打在ptr指向的commit上.

`git describe [ptr]` 返回离ptr指向的commit最近的tag，格式为`<tag>_<numCommits>_g<hash>`，其中`<tag>`是离ptr最近的标签，`<numCommits>`表示ptr与这个标签之间相差多少个commit，`<hash>`是ptr指向的commit的hash值.

## reflog

`git reflog`展示分支的操作历史，可以配合`git reset`实现撤销操作。

![](/images/Git_branch/reflog.gif)