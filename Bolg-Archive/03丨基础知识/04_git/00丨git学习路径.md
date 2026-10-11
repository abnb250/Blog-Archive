# Git学习路径
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  

<br>

<br>

## 00 前言
***
使用方式：先看目录，再从第 0 节开始。每节都建议在终端里敲一遍。  
练习项目统一叫 todo-app。  
Windows 用户建议用 Git Bash，macOS / Linux 用户用 Terminal。  
如果某个命令报错说 switch 不存在，说明 Git 版本较旧，可以用 checkout 替代，教程里会提醒。  

<br>

<br>

## 01 先认识终端、目录和文件
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23210766)  
终端 / 路径 / 目录 / 文件 · cd、ls、pwd · 零基础命令行入门，分清绝对路径与相对路径。  
适合：Git 零基础 / 命令行新手 · 约 10 分钟 · #Git #终端 #命令行

<br>

<br>

## 02 Git 是什么？为什么需要它
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23210843)  
版本控制 / 分布式 / 快照 / 仓库 · 理解 Git 能解决什么问题，为什么比手动备份强。  
适合：没用过版本控制的人 · 约 8 分钟 · #Git #版本控制

<br>

<br>

## 03 安装 Git 和配置身份
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23210912)  
安装 / git config / user.name / user.email · 配好环境与提交身份，避免提交信息混乱。  
适合：刚接触 Git 的新手 · 约 6 分钟 · #Git #安装配置

<br>

<br>

## 04 创建第一个 Git 仓库
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23211165)  
git init / 仓库 / 工作区 / .git 目录 · 创建并理解第一个 Git 仓库的结构。  
适合：装好 Git 准备动手的人 · 约 8 分钟 · #Git #仓库

<br>

<br>

## 05 第一次提交：add 和 commit
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23215317)  
git add / git commit / 暂存区 / 提交 · 走通第一次提交流程，理解暂存区的作用。  
适合：刚开始用 Git 管理代码的人 · 约 10 分钟 · #Git #提交

<br>

<br>

## 06 查看状态和历史：status、log
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23215385)  
git status / git log / 提交历史 · 随时看清仓库状态与变更记录。  
适合：日常使用 Git 的所有人 · 约 8 分钟 · #Git #状态 #日志

<br>

<br>

## 07 修改文件与查看差异：diff
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23215965)  
git diff / 工作区 / 暂存区 / 差异对比 · 提交前确认到底改了什么，避免误提交。  
适合：需要精细控制提交内容的人 · 约 8 分钟 · #Git #diff

<br>

<br>

## 08 忽略文件：.gitignore
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23216070)  
.gitignore / 忽略规则 / 通配符 · 排除编译产物、日志、依赖目录，保持仓库干净。  
适合：项目中有大量生成文件的开发者 · 约 8 分钟 · #Git #gitignore

<br>

<br>

## 09 分支：平行开发
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23216160)  
branch / checkout / switch / 平行开发 · 创建、切换、查看分支，理解分支的本质。  
适合：多人协作或功能隔离开发 · 约 12 分钟 · #Git #分支

<br>

<br>

## 10 合并与冲突
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23235735)  
merge / 冲突 / 手动解决 · 合并分支并处理冲突，掌握冲突解决流程。  
适合：协作中需要合并代码的人 · 约 12 分钟 · #Git #合并 #冲突

<br>

<br>

## 11 远程仓库与 GitHub
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23236316)  
远程仓库 / GitHub / origin / remote · 关联本地与远程，理解 origin 和远程分支。  
适合：准备把代码推上 GitHub 的人 · 约 10 分钟 · #Git #GitHub #远程

<br>

<br>

## 12 克隆、推送、拉取
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23240104)  
clone / push / pull / fetch · 克隆、推送、拉取更新，日常与远程同步的核心操作。  
适合：需要与远程仓库同步的人 · 约 10 分钟 · #Git #push #pull

<br>

<br>

## 13 撤销与回退
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23242498)  
reset / revert / checkout / 恢复 · 撤销修改、回退版本、找回文件，改错了也不慌。  
适合：想安全撤销操作的人 · 约 12 分钟 · #Git #撤销 #回退

<br>

<br>

## 14 临时保存与标签：stash、tag
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23242795)  
stash / tag / 临时保存 / 版本标记 · 切任务前保存现场，发布时打标签。  
适合：需要临时切换任务或标记版本的人 · 约 8 分钟 · #Git #stash #tag

<br>

<br>

## 15 团队协作流程：分支 → PR → 合并
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23242858)  
分支 / PR / Code Review / 合并 · 走通团队协作标准流程，理解 PR 的作用。  
适合：参与开源或公司项目的人 · 约 12 分钟 · #Git #PR #协作

<br>

<br>

## 16 进阶概念：rebase、cherry-pick、reflog、bisect
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23242928)  
rebase / cherry-pick / reflog / bisect · 进阶操作与救急手段，提升效率与排查能力。  
适合：Git 基础熟练后想进阶的人 · 约 15 分钟 · #Git #进阶

<br>

<br>

## 17 常见错误与救命命令
***
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23242984)  
常见错误 / 救命命令 / 恢复 · 报错时快速定位与补救，建议收藏备用。  
适合：所有 Git 使用者 · 约 10 分钟 · #Git #排错 #救命命令

<br>

<br>