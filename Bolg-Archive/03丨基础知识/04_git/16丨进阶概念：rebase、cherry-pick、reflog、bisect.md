# 16 进阶概念：rebase、cherry-pick、reflog、bisect
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23210717)  

<br>

<br>

## 00 本节目标
***
知道这些词是什么意思，先不精通。  

<br>

<br>

## 01 rebase
****
```bash
git rebase main
```
把当前分支的提交挪到‘main’更新最后面，  
让历史更加线性。  
> 不要对已经推送到公共分支的历史做rebase，  
>除非你很清楚后果。

## 02 cherry-pick
***
```bash
git cherry-pick <commit-id>
```
把某个提交单独挑到当前分支。  

<br>

<br>

## 03 reflog
***
```bash
git reflog
```
查看HEAD移动记录，  
误删分支或reset后，  
可以用它来找回提交。  

<br>

<br>

## 04 bisect
***
```bash
git bisect start
git bisect bad
git bisect good <commit-id>
```
二分查找哪个提交引入了bug。  

<br>

<br>

## 05 练习
***
先知道名字即可，以后遇到再查。  

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>