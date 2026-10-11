# 03 安装 Git 和配置身份
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23210717)  

<br>

<br>

## 00 本节目标
***
去报安装好 Git，配置用户名和邮箱。  

<br>

<br>

## 01 检查是否安装
*** 
使用如下指令检查是否安装：  
```bash
git --version
```
如果显示类似：
```text
git version 2.53.0.windows.1
```
则说明安装成功。  

<br>

如果没有：  
- Windows：搜索 Git for Windows，下载安装。
- macOS：可以 brew install git，或安装 Xcode Command Line Tools。
- Linux：sudo apt install git 或对应包管理器。

<br>

<br>

## 02 配置用户名和邮箱
***
我们知道，Git是用来控制版本的，知道谁提交的，  
因此，我们需要给自己配置用户名和邮箱，  
这样自己和别人就知道这是由你提交的。  

<br>

在电脑cmd中输入，指令如下：  
```bash
git config --global user.name "你的名字"
git config --global user.email "完整邮箱地址"
```
--global 表示这台电脑上所有Git仓库都用这个配置。

<br>

<br>

## 03 设置默认分支名
***
我们需要给默认的分支进行命名，  
cmd中输入：  
```bash
git config --gloval init.defaultBranch main
```
这样以后 git init 创建的默认分支都叫 main。  

<br>

<br>

## 04 查看配置
***
上面进行配置完之后，需要查看配置，以便于确认是否配置成功。  
cmd中输入：  
```bash
git config --list
```  
你会看到：  
```text
user.name=小明
user.email=xiaoming@example.com
init.defaultbranch=main
```  

<br>

<br>

## 05 练习
***
配置你自己的名字和邮箱，然后用 git config --list 检查。  


<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>