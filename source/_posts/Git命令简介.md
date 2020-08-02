---
title: Git命令简介
date: 2020-08-01 16:58:57
tags: git
---

# 配置

```shell
git config --global user.name 'dreamgyf' //设置用户名
git config --global user.email g2409197994@gmail.com //设置邮箱
```



# 基本命令

```shell
git init //在当前目录创建版本库
git add README.md //将文件添加到暂存区
git commit //提交暂存区中的修改 -m"xxx":本次提交的说明 -a:相当多加了一步于git add .
git branch 分支名 //创建分支，不加分支名可以查看所有分支 -d:删除指定分支
git checkout 分支名 //切换分支 -b:创建并切换分支，相当于多加了一步git branch 分支名
git remote add https://github.com/xxx/xxx.git //添加远程仓库，一般支持https和ssh两种协议
git fetch //将远程主机的更新全部取回本地，后面加分支名的话只会取回指定分支
git merge 分支名 //将选中分支合并到当前分支
git pull 分支名 //相当于git fetch 分支名 + git merge 分支名
git push //将当前本地分支推送至远程分支，如果是第一次推送则需要-u参数指定远程分支并建立联系，如git push -u origin master，下一次便可不加参数直接推送 -f:强制推送，会覆盖远程分支
```

# Rebase

- 合并commit记录

  ```shell
  git rebase -i  [startpoint]  [endpoint]
  ```

  其中`-i`的意思是`--interactive`，即弹出交互式的界面让用户编辑完成合并操作，`[startpoint] [endpoint]`则指定了一个编辑区间，如果不指定`[endpoint]`，则该区间的终点默认是当前分支HEAD所指向的commit(注：该区间指定的是一个前开后闭的区间)。

  命令可以按如下方式：

  ```shell
  git rebase -i [commit id]
  ```

  或

  ```shell
  git rebase -i HEAD~3
  ```

  然后会出现一个vi编辑器界面，会提供给我们一个命令列表：

  > pick：保留该commit（缩写:p）
  >
  > reword：保留该commit，但我需要修改该commit的注释（缩写:r）
  >
  > edit：保留该commit, 但我要停下来修改该提交(不仅仅修改注释)（缩写:e）
  >
  > squash：将该commit和前一个commit合并（缩写:s）
  >
  > fixup：将该commit和前一个commit合并，但我不要保留该提交的注释信息（缩写:f）
  >
  > exec：执行shell命令（缩写:x）
  >
  > drop：丢弃该commit（缩写:d）

  然后我们就可以在里面修改提交了，例如:

  ```shell
  pick d2cf1f9 fix: 第一次提交
  s 47971f6 第二次提交
  s fb28c8d 第三次提交
  ```

  将第二、三次的commit合并到第一次上

  编辑完保存退出vi就可以完成对commit的合并了

  如果保存时出现错误，输入`git rebase --edit-todo`便会回到编辑模式里，修改完后保存，`git rebase --continue`

  [参考链接](https://juejin.im/entry/6844903600976576519)

- 合并分支

  `rebase`和`merge`都是合并分支的操作

  `merge`会在合并分支时产生一个新的commit记录，而`rebase`会以指定分支作为基础分支，之前所做的改动全部会在指定分支的基础上提交，不会产生新的commit记录。

  ```shell
  git checkout dev
  git rebase master
  ```

  分析一下上面命令进行的操作：

  首先，切换到`dev`分支上；

  然后，`git` 会把 `dev` 分支里面的每个 `commit` 取消掉；

  其次，把之前的`commit`临时保存成 `patch` 文件，存在 `.git/rebase` 目录下；

  然后，把`dev`分支更新到最新的 `master` 分支；

  最后，把上面保存的 `patch` 文件应用到 `dev` 分支上；

  - 使用场景
    1. 想要干净简洁的分支树
    2. 在一个过时的分支上面开发的时候，执行 `rebase` 以同步 `master` 分支最新变动

  **注意：当同一个分支有多个人使用的情况下，谨慎使用rebase，因为它改变了历史，可能会出现丢失commit的情况**

  [参考链接](http://jartto.wang/2018/12/11/git-rebase/)