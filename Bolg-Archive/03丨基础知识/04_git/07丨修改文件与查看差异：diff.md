# 07 修改文件雨查看差异： diff
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23210717)  

<br>

<br>

## 00 本节目标
***
学会看自己改了什么

<br>

<br>

## 01 修改README
我们先对README进行修改，  
有修改才能看改了什么。  
```bash
echo "- 学习 Git" >> README.md
```
`>>` 表示追加内容。  

<br>

<br>

## 02 查看差异
***
使用diff查看不同：
```bash
git diff
```
将会得到类似输出：  
```text
+# - 学习 Git
```
这表示工作区新增了这一行，  
但还没有add。  

<br>

<br>

## 03 加入暂存区后呢
***
我们先加入暂存区：
```bash
git add README.md
```
我们查看一下已经暂存，但还没有提交的内容：
```bash
git diff --staged
```
--staged 看的是已经暂存、但还没有提交的内容。  

<br>

<br>

## 04 提交
***
我们可以提交后再查看两种
```bash
git commit -m "docs: 添加学习待办" 
```

<br>

<br>

## 05 总结
***
`git diff`: 工作区 VS 暂存区  
`git diff --staged`: 暂存区 VS 最近一次提交。  

<br>

<br>

## 06 练习
*** 
1. 修改 README。
2. 用 git diff 看变化。
3. git add README.md。
4. 用 git diff --staged 看变化。
5. 提交。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>