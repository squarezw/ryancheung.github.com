---
layout: post
title: "使用 git flow 工具管理开发流程"
date: 2012-09-02 23:27
comments: true
categories: git git-flow
---

我们都知道, 在 git 的分支功能相对 svn 确实方便许多，而且也非常推荐使用分支来做开发.
我的做法是每个项目都有2个分支, master 和 dev. master 分支是主分支, 保证程序有一个
稳定版本, dev 则是开发用的分支, 几乎所有的功能开发, bug 修复都在这个分支上, 完成后
再合并回 master.

但是情况并不是这么简单. 有时当我们正在开发一个功能, 但程序突然出现
 bug 需要及时去修复的时候, 这时要切回 master 分支, 并基于它创建一个 hotfix 分支.
有时我们在开发一个功能时, 需要停下来去开发另一个功能. 而且所有这些问题都出现
的时候, 发布也会成为比较棘手问题.

也就是说, git branch 功能很强大，但是没有一套模型告诉我们应该怎样在开发的时候善用
这些分支。于是有人就整理出了一套比较好的方案
[A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/),
今天我们就一起来学习下这套方案.

简单来说, 他将 branch 分成2个主要分支和3个临时的辅助分支:
![git flow](http://nvie.com/img/2009/12/Screen-shot-2009-12-24-at-11.32.03.png)

**主要分支**

*   master: 永远处在即将发布(production-ready)状态
*   develop: 最新的开发状态

**辅助分支**

* feature branches: 开发新功能的分支, 基于 develop, 完成后 merge 回 develop
* release branches: 准备要发布版本的分支, 用来修复 bug. 基于 develop, 完成后 merge 回 develop 和 master
* hotfix branches: 修复 master 上的问题, 等不及 release 版本就必须马上上线. 基于 master, 完成后 merge 回 master 和 develop

作者还提供了 `git-flow` 命令工具:

首先初始化:

    git flow init

初始化动作会让你设置这些分支的命名, 默认就好:

    No branches exist yet. Base branches must be created now.
    Branch name for production releases: [master]
    Branch name for "next release" development: [develop]
    How to name your supporting branch prefixes?
    Feature branches? [feature/]
    Release branches? [release/]
    Hotfix branches? [hotfix/]
    Support branches? [support/]
    Version tag prefix? []

完成后当前所在分支就变成 develop. 任何开发都必须从 develop 开始:

    git flow feature start some_awesome_feature

完成功能开发之后:

    git flow feature finish some_awesome_feature

这时就会合并回 develop 并删除这个 some_awesome_feature 分支

将一个 feature 分支推到远程服务器:

    git flow feature publish some_awesome_feature
    或者
    git push origin feature/some_awesome_feature

track 一个远程分支:

    git flow feature track some_awesome_feature
    或者
    git checkout -b feature/some_awesome_feature -t origin/feature/some_awesome_feature

关于 commit:

分支在 merge 回 develop 或 master 的时候都会添加 --no-ff  参数, 这样做有个好处就是,
每一次的 merge 就代表一个功能完成, 可以清晰地看到这个功能开发下的每个提交历史.

最后 git flow in github: [https://github.com/nvie/gitflow](https://github.com/nvie/gitflow)
