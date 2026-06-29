---
title: git 常用场景命令集合
tags:
  - 研发工具
categories:
  - 研发工具笔记
cover: https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1280
description: git 常用场景命令集合
katex: false
mermaid: true
mathjax: false
abbrlink: 40921
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
## 1. git rebase 直接跟分支名
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

## 2. git rebase -i 使用
 `git rebase -i`（交互式变基）是 Git 最强大的历史重写工具。它允许你**修改、合并、删除或重新排序**提交记录，让提交历史变得干净整洁。

**⚠️ 黄金法则**：**绝不要**对已经推送到团队公共分支（如 main/master）的提交执行 rebase，否则队友会“爆炸”。只对**本地尚未推送**或**个人功能分支**使用。
```
git rebase -i HEAD~3   # 操作最近最近的 3 条提交
```
或者指定一个基准点（操作此基准点之后的提交）：
```
git rebase -i main     # 操作 main 分支之后的所有当前分支提交
```
执行后，Git 会弹出文本编辑器（Vim/VS Code），内容类似如下：

```
pick a1b2c3d 添加登录功能
pick e4f5g6h 修复登录bug
pick i7j8k9l 优化登录样式

# Rebase 3 commits onto ...
# 命令说明:
# p, pick = 保留该提交
# r, reword = 保留内容，修改提交信息
# e, edit = 停住以便修改该提交内容
# s, squash = 合并到前一个提交，保留新注释
# f, fixup = 合并到前一个提交，丢弃新注释
# d, drop = 删除该提交
```
 ### 2.1  核心命令详解（最常用 4 个）

编辑上面的 `pick` 单词来改变行为：
|命令|简写|作用|适用场景|
|---|---|---|---|
|**pick**|`p`|原样保留该提交|默认选项，不做改动|
|**reword**|`r`|修改提交信息（Message）|发现上次的注释写错了字|
|**edit**|`e`|修改提交内容（文件改动）|忘记加一个文件，或想拆分提交|
|**squash**|`s`|**合并**到前一个提交，**保留**新注释|把“修复bug”合并到“添加功能”里|
|**fixup**|`f`|**合并**到前一个提交，**丢弃**新注释|同上，但懒得写注释，直接用旧的|
|**drop**|`d`|删除该提交|废弃某个没用的中间提交|

### 2.2 场景
### 场景一：合并多个提交（Squash）—— 最常用

假设你有 3 个提交，想把它们合成 1 个干净的提交。  
把第 2、3 行的 `pick` 改为 `squash`（或 `s`）：
```
text
pick a1b2c3d 第一次实现功能
squash e4f5g6h 修复typo
squash i7j8k9l 添加单元测试

# 从下往上合并，这种场景下会呈现到第一行 pick a1b2c3d 第一次实现功能
```

保存退出。Git 会再次弹出编辑器让你编写**合并后**的新提交信息。删除旧的，写一句漂亮的 `feat: 完成某功能` 即可。

### 场景二：修改某次历史提交的注释（Reword）

```
pick a1b2c3d 添加登录功能
reword e4f5g6h 修复登录bug   # 改成 "fix: 修复登录超时问题"
pick i7j8k9l 优化登录样式
```

保存后，Git 会弹出编辑器让你修改该 commit 的信息。

### 场景三：修改某次提交的内容（Edit）

```
pick a1b2c3d 添加登录功能
edit e4f5g6h 修复登录bug   # 想往这次提交里追加一个文件
pick i7j8k9l 优化登录样式
```

保存退出后，Git 会停在 `e4f5g6h` 这个提交处，命令行进入 `rebase` 模式。  
此时你可以修改文件，然后执行：
```
bash
git add .                    # 暂存新改动
git commit --amend           # 追加到当前提交
git rebase --continue        # 继续执行下一个提交
```

---

### 2.3 操作中的常见问题与解决

- **保存后怎么退出（Vim 编辑器）**：按 `Esc`，输入 `:wq`（保存退出）；或 `:q!`（强制放弃）。
- **怎么取消这次变基？**  
    如果搞乱了且还没执行完，随时可以跑路：
    ```
    git rebase --abort   # 彻底回退到变基前的状态
    ```

- **变基到一半遇到冲突怎么办？**
    
    1. 手动解决冲突文件。
        
    2. `git add .`
        
    3. 继续：`git rebase --continue`
        
    4. 如果解不开，随时 `git rebase --abort` 回到原点。
        

### 2.4 高级技巧：改变顺序

直接把文本编辑器里的行**上下拖动**即可改变提交顺序。Git 会按**从上到下**的顺序依次应用。

2.5 举例 git rebase -i
```
pick a1b2c3d 第一次实现功能
squash e4f5g6h 修复typo 
pick i7j8k9l 添加单元测试 代表什
squash d425g6h 修复typo2
```


这是一个**连续合并**的操作，让我们来拆解它的执行逻辑和最终效果。

---

### 1. 你的指令序列

text

pick a1b2c3d 第一次实现功能    <--- 第1个（基准）
squash e4f5g6h 修复typo       <--- 第2个（合并给第1个）
pick i7j8k9l 添加单元测试      <--- 第3个（独立保留）
squash d425g6h 修复typo2      <--- 第4个（合并给第3个）

---

### 2. 执行逻辑（谁合并给谁）

规则依然是：**每个 `squash` 都合并给它紧挨着的上一个提交**。

- **第一组**：`e4f5g6h`（修复typo）→ 合并给 `a1b2c3d`（第一次实现功能）
    
- **第二组**：`d425g6h`（修复typo2）→ 合并给 `i7j8k9l`（添加单元测试）
    

> 注意：`d425g6h` **不会**跨过 `i7j8k9l` 去合并给 `a1b2c3d`，它只找**紧邻的上面那个**，即 `i7j8k9l`。

---

### 3. 最终结果（变基后的提交树）

**变基前（4个提交）**：

text

A(功能) ← B(修复typo) ← C(测试) ← D(修复typo2)

**变基后（2个提交）**：

text

X(功能+修复typo) ← Y(测试+修复typo2)

详细说明：

- **新提交 X**：由 `a1b2c3d` + `e4f5g6h` 合并而成。会弹窗让你编辑合并后的提交信息。
    
- **新提交 Y**：由 `i7j8k9l` + `d425g6h` 合并而成。也会弹窗让你编辑合并后的提交信息（**注意：会弹两次窗！**）
    

---

### 4. 你会经历的操作流程

1. **保存并退出** `git rebase -i` 编辑界面。
    
2. **第一次弹窗**：编辑 `a1b2c3d` + `e4f5g6h` 合并后的提交信息。写完后保存退出。
    
3. **第二次弹窗**：编辑 `i7j8k9l` + `d425g6h` 合并后的提交信息。写完后保存退出。
    
4. 变基完成。
    

---

### 5. 如果你想一次性合并所有 4 个提交呢？

如果你想把这 4 个全部合并成 **1 个**提交，应该改成：

text

pick a1b2c3d 第一次实现功能
squash e4f5g6h 修复typo
squash i7j8k9l 添加单元测试    <--- 把 pick 改成 squash
squash d425g6h 修复typo2       <--- 已经是 squash

这样就是链式合并：B→A，C→(A+B)，D→(A+B+C)，最终 4 个变 1 个。

---

### 6. 常见错误提醒 ⚠️

| 错误写法                        | 问题                          |
| --------------------------- | --------------------------- |
| `squash a1b2c3d` 放在第一行      | 第一个提交没有"上一个"可合并，会报错         |
| 以为 `d425g6h` 会合并给 `e4f5g6h` | 错，它只合并给它**紧邻上面的** `i7j8k9l` |






