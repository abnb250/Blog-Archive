# 11 远程仓库与GitHub
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23210717)  

<br>

<br>

## 00 本节目标
***
把本地仓库推送到 GitHub。

<br>

<br>

## 01 注册GitHub
***
打开GitHub官网，注册账号。  

<br>

<br>

## 02 创建远程仓库
***
在 GitHub 上点 New repository。  
- 仓库名：todo-app
- 不要勾选初始化 README
- 创建

创建后会得到一个地址，例如：  
```text
https://github.com/你的用户名/todo-app.git
```

<br>

<br>

## 03 关联远程仓库
***
在本地的todo-app目录：  
```bash
git remote add origin https://github.com/你的用户名/todo-app.git
```
这里origin是远程仓库的默认名字。    

<br>

使用下面的命令查看：
```bash
git remote -v
```

<br>

<br>

## 04 确保分支叫main
***
```bash
git branch -M main
```

<br>

<br>

## 05 推送
***
使用下面命令推送：  
```bash
git push -u origin main
```
解释：  
- push ：本地上传提交
- -u ：把本地main和远程main关联
- 以后直接git push 即可。
- 这里的main代表本地分支

<br>

我们检查一下：
```bash
git branch -vv
```
会显示类似：
```text
main  xxxxxxx [origin/main] ...
```
就说明本地 main 正在跟踪远程 origin/main。  

## 06 认证问题
***
GitHub 现在不能用密码推送。  
你需要：  
使用 Personal Access Token，或配置 SSH key。  
新手可以先按提示用浏览器登录，或生成 token。  

<br>

<br>

## 07 练习
***
1. 在 GitHub 创建空仓库。
2. 本地关联 origin。
3. 推送到 GitHub。
4. 刷新 GitHub 页面，确认文件出现。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>