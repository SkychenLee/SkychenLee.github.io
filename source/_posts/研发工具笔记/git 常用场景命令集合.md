---
title: git 常用场景命令集合
tags:
  - 研发工具
categories:
  - 研发工具笔记
cover: 'https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1280'
description: >-
  git 常用场景命令集合
katex: false
mermaid: false
mathjax: false
abbrlink: 40991
date: 2026-6-27 10:00:00
top_img:
---

**最近AI，Agent场景Git有很多新玩法，但是之前git老是用图形化界面，命令行一直记不住，这个帖子方便记忆各个场景下的git命令。** 
# 场景1、本地新建项目，推送到远程仓并在远程仓也是同样新建项目 

## 第 1 步：本地初始化 Git 
在项目根目录下打开终端（Git Bash / CMD / Terminal），执行： 
``` bash git init （此时会生成一个隐藏的 .git 文件夹，表示本地版本库建立成功） ``` 
## 第 2 步：添加并提交所有文件 
将当前项目所有文件添加到暂存区，并完成第一次提交： 
``` bash git add . git commit -m "first commit" ``` 
注意：如果项目中有不需要上传的临时文件（如 node_modules、.idea），请务必先新建一个 .gitignore 文件并写好忽略规则，再执行上面的 add . 命令。 
## 第 3 步：去远程仓库平台创建空项目 
打开 GitHub / GitLab / Gitee 等平台，点击 “New repository”（新建仓库）。 
关键点：千万不要勾选“Initialize this repository with a README”（使用 README 初始化），一定要保持仓库完全为空（没有 README、没有 .gitignore、没有 License），否则会和本地已有提交产生冲突。 创建好后，复制远程仓库的 HTTPS 或 SSH 地址（例如：https://github.com/你的用户名/项目名.git）。 
## 第 4 步：关联本地与远程 
在终端执行以下命令，将本地仓库与远程仓库关联起来： 

```bash git remote add origin <你复制的远程仓库地址> ``` 
## 第 5 步：推送本地代码到远程 执行推送命令： 

``` bash git push -u origin main ```
注意：如果你的本地默认分支是 master（旧版 Git），请将上面的 main 替换为 master。如果不确定，先执行 git branch 查看当前分支名。 参数解释：-u 的作用是建立本地分支与远程分支的追踪关系，执行这次后，以后只需要输入 git push 即可推送。 遇到报错怎么办？（常见避坑） 报错 failed to push some refs：说明你在远程平台创建仓库时勾选了 README 或 License。解决办法：强制覆盖推送（仅在首次且确认远程文件无用的情况下执行）：
``` bash git push -u origin main --force ``` 
如果远程创建仓时初始化了README或者有其他提交怎么办： 
``` 拉取远程文件（允许无关联历史） git pull origin master --allow-unrelated-histories 处理可能出现的冲突（很关键） ``` 
# 场景2、git rebase使用
git rebase 命令主要可以改变一个分支的基点（其实就是对一个个commit进行增删改查) ## 用法一 git rebase 分支 这个命令的一般用法: 你想把一个其他分支的修改合入到你自己分支的修改，可以直接rebase（效果和merge一样但原理不一样）

``` 
case1:
 1、git checkout feature # 切换到feature分支 
 2、git rebase master # 把master的修改对feature进行rebase以及合入 
 3、git push origin feature --force # 需要加--force 因为master如果有修改的话，本地feature的基分支会和远程的不一样
case2: 
1、git pull origin feature --rebase 
2、 git pull origin feature 1和2区别是1走rebase, 2走的merger。rebase走的是变基分支（类似fast-forward），merge走的是在head位置生成新commit（--no-ff）。 
``` 
