# 04 创建第一个 Git 仓库
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23210717)  

<br>

<br>

## 00 本节目标
***
学会用 git init 创建仓库。  

<br>

<br>

## 01 创建练习项目
***
先创建一个练习的项目，后续所有操作都在这个项目内部：  
```bash
mkdir todo-app
cd todo-app
```

<br>

<br>

## 02 初始化Git仓库
***
```bash
git init
```
输出类似：  
```text
Initialized empty Git repository in /.../todo-app/.git/
``` 
这意味着todo-app变成了一个Git仓库。  

<br>

<br>

## 03 看看多了什么
***
使用指令：  
```bash
ls -a
```
你会看到：  
```text
. .. .git
```
.git是用来保存历史记录的目录，  
记住不要手动修改里面的东西。  

<br>

<br>

## 04 检查状态
***
输入：  
```bash
git status
```
输出类似：  
```text
On branch main
No commits yet
nothing to commit
```
意思是：当前在 main 分支，还没有任何提交。  

<br>

<br>

## 05 练习
***
1. 创建目录 todo-app。
2. 进入目录。
3. 执行 git init。
4. 执行 git status，观察输出。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>