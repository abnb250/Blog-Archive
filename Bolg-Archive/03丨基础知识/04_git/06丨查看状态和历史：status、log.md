# 06 查看状态和历史：status、log
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23210717)  

<br>

<br>

## 00 本节目标
***
学会看Git状态和历史  

<br>

<br>

## 01 git status
***
这是最重要的命令：  
```bash
git status
```
它告诉你：  
- 当前在哪个分支
- 哪些文件改了但没有保存
- 哪些文件暂存的但没提交
- 有没有未跟踪文件

不知道怎么办的时候，  
就敲这个最重要的命令。  

<br>

<br>

## 02 git log
***
查看历史的log，也有很多种版本。  
查看完整历史：  
```bash
git log
```

<br>

查看简洁历史：  
```bash
git log --oneline
```

<br>

查看分支图：
```bash
git log --oneline --graph --all
```

<br>

例子输出：
```text
* a1b2c3d docs: 添加 README
* b2c3d4e feat: 添加任务
```

<br>

<br>

## 03 查看某次提交
***
可以使用show+提交ID查看该提交：  
```bash
git show [ID]
```

<br>

<br>

## 04 练习
***
1. 多做两次提交。
2. 用 git log --oneline 查看。
3. 用 git log --oneline --graph --all 查看分支图。


<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>