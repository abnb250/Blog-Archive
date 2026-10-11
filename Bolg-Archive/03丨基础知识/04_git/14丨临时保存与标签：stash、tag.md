# 14 临时保存与标签：stash、tag
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23210717)  

<br>

<br>

## 00 本节目标
***
学会临时保存修改，  以及给版本打标签。

<br>

<br>

## 01 临时保存：stash
***
场景：你在feature/add-task分支改到一半，  
突然要切换到别的分支，去做别的事情。    
```bash
echo "2. 测试" >> tasks.txt
git stash push -m "任务写到一半"
```

<br>

我们可以查看一下保存列表：
```bash
git stash list
```

<br>

切走，再切回：
```bash
git switch main
git switch feature/add-task
git stash pop
```
修改又回来了  

<br>

常用：
```bash
git stash push -m "说明"
git stash list
git stash pop
git stash apply
git stash drop
```
- pop：恢复并删除最近一条。

- apply：恢复但不删除。

- drop：删除某条。

<br>

<br>

## 02 标签tag
***
标签通常用于版本发布：
```bash
git tag -a v1.0 -m "第一个版本"
```
我们还可以查看：
```bash
git tag
```
推送标签：
```bash
git push origin v1.0
```
推送所有标签:
```bash
git push origin --tags
```

<br>

<br>

## 03 练习
***
1. 修改文件后用 git stash 保存。
2. 用 git stash list 查看。
3. 用 git stash pop 恢复。
4. 打一个 v0.1 标签。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>