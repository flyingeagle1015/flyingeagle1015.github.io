---
title: git简易教程
date: 2019-03-19 09:25:18
tags: [git]
categories: [application]
---

# 创建版本库
```
$ mkdir learngit
$ cd learngit
$ git init
$ ls -la
.  ..  .git
```

# 添加文件到版本库
```
git add readme.txt
git commit -m 'wrote a readme file'

#仓库当前的状态
git status
#文件对比
git diff readme.txt
git diff HEAD -- readme.txt
```

# 版本回退
```
git log
git log --pretty=oneline
#撤回上次提交
git reset --hard HEAD^ 
#查看命令历史
git reflog
#撤回到指定提交
git reset --hard 801f
```

# 撤销修改
```
#丢弃工作区修改
git checkout -- readme.txt
#撤销暂存区的修改
git reset HEAD readme.txt
```

# 删除文件
```
git rm test.txt
```

# 分支管理
## 创建与合并分支 
```
查看分支：git branch
创建分支：git branch <name>
切换分支：git checkout <name>
创建+切换分支：git checkout -b <name>
合并某分支到当前分支：git merge <name>
删除分支：git branch -d <name>
```

## 冲突解决
```
git merge dev
git log --graph --pretty=oneline --abbrev-commit
```

## 分支管理
在实际开发中, 我们应该按照几个基本原则进行分支管理:    
首先, master分支应该是非常稳定的, 仅用来发布新版本;  
dev分支是不稳定的, 干活都在dev分支上, 版本发布时再把dev分支合并到master上然后发布版本;  
每个人都在dev分支上干活, 每个人都有自己的分支, 时不时地往dev分支上合并。 

合并分支时, 加上--no-ff参数就可以用普通模式合并, 合并后的历史有分支, 能看出来曾经做过合并, 而fast forward(默认)合并就看不出来曾经做过合并。

## BUG分支
```
git stash 暂存工作区
git checkout master
git checkout -b issue-101
#fix bug
git commit -a -m 'fix bug 101'
git checkout master
git merge --no-ff -m 'merged bug fix 101' issue-101
git checkout dev
git stash list 查看暂存
git stash apply 恢复暂存但需要执行git stash drop来删除暂存
git stash pop  恢复并自动删除
git stash apply stash@{0} 恢复指定的暂存
```

## Feature分支
不让预研代码搞乱主分支代码, 对预研代码做feature分支。
```
git checkout -b feature-vulcan
git commit -a -m 'add feature vulcan'
git checkout dev
git branch -d feature-vulcan
git branch -D feature-vulcan
```

## 多人协作
```
git remote   查看远程库
git remote -v
git push origin master 推送分支
git push origin dev    只推送别人需要查看的分支
git clone git@github.com:user/learngit.git
git checkout -b dev origin/dev 远程分支dev
git pull 拉取其他人的更新
git branch --set-upstream-to=origin/dev dev
```

## Rebase
rebase操作的特点：把分叉的提交历史“整理”成一条直线,看上去更直观,缺点是本地的分叉提交已经被修改过了。
```
git rebase 
```

# 标签管理
## 创建标签
```
git tag v1.0 创建标签
git tag v0.9 f52c633   给历史提交打标签
git tag -a v0.1 -m "version 0.1 released" 1094adb 打标签附带说明
git tag      查看标签列表
git show v0.9 查看标签信息
```

## 操作标签
```
git tag -d v0.1  删除标签
git push origin v1.0推送标签到远程
git push origin --tags 推送全部未推送的标签
从远程删除标签
git tag -d v0.9
git push origin :refs/tags/v0.9
```

# 自定义Git
```
git config --global color.ui true

#别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.br branch
git config --global alias.unstage 'reset HEAD'
git config --global alias.last 'log -1'
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"

#配置文件
$REPO/.git/config
$HOME/.gitconfig

#忽略特殊文件
.gitignore
参考https://github.com/github/gitignore
忽略文件的原则是：
忽略操作系统自动生成的文件，比如缩略图等；
忽略编译生成的中间文件、可执行文件等，也就是如果一个文件是通过另一个文件自动生成的，那自动生成的文件就没必要放进版本库，比如Java编译产生的.class文件；
忽略你自己的带有敏感信息的配置文件，比如存放口令的配置文件。

git add -f App.class  强制添加文件到暂存区
git check-ignore -v App.class 检查.gitignore
```

