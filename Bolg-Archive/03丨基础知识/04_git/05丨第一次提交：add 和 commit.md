# 05 第一次提交：add和commit
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23210717)  

<br>

<br>

## 00 本节目标
***
完成第一次完整提交  

<br>

<br>

## 01 创建README文件
***
每一个标准的项目内，都会有一个README文件，  
可以方便使用者了解如何使用，  
所以我们先创建一个README文件：  
```bash
echo "# Todo App" > README.md
```

<br>

<br>

## 02 查看状态
***
我们使用之前提到的status来查看状态：  
```bash
git status
```
将会得到类似的输出：
```text
Untracked files:
  README.md
```  
这里的Untracked 表示Git还没跟踪这个文件。  

<br>

<br>

## 03 加入暂存区
***
复习一下，Git被分为四个区域：工作区、暂存区、本地仓库和远程仓库。  
我们使用add指令将文件加入暂存区：  
```bash
git add README.md
```
再次查看状态会得到类似：  
```text
Changes to be committed:
  new file: README.md
```  
意思是，REAMDE.md 已经进入暂存区，准备提交。  

<br>

<br>

## 04 提交
***
我们使用commit指令进行提交：
```bash
git commit -m "docs: 添加 README"
```
解释一下：
- commit 表示提交
- -m 后面跟着提交信息
- "..." 表示这次提交的说明

<br>

<br>

## 05 查看历史
***
我们使用下面的指令查看提交历史：
```bash
git log --oneline
```
会得到类似输出：  
```text
a1b2c3d docs: 添加 README
```
其中a1b2c3d 表示的是提交的ID，  
每一次的提交ID都将不一样。  

<br>

<br>

## 06 总结一下
***
总结一下，本次使用的这些语句，  
最常用的会是下面四句：  
```bash
git status
git add .
git commit -m "说明"
git log --oneline
```
其中` git add . ` 表示把当前目录下的所有改动加入暂存区。  

<br>

<br>

## 07 练习
***
1. 创建 README.md。  
2. git add README.md。  
3. git commit -m "docs: 添加 README"。  
4. 用 git log --oneline 查看历史。  


<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>