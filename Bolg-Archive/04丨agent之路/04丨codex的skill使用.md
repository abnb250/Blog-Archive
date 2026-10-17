# 04 codex的skill使用
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  

<br>

<br>

## 00 前言
***
接上回安装好codex后，这次安装和使用codex的skill。  
由于codex只认C盘路径的，所以本博客尝试弄到D盘去。  

<br>

<br>

## 01 下载skill
***
网上随便找一个skill仓库，仓库内其实是两大部分：skill和示例。  
我们只需要skill，skill就是文件夹下有SKILL.md的文件夹，就下这一个文件夹。  
下载成功后解压。  
我这里以ppt-master这个skill为例。   

<br>

<br>

## 02 安装Skill
***
我们首先知道codex的官方skill路径为：`C/users/用户名/.codex/skills`。  
由于这是在C盘，我们试图使用目录联接 junction让它可以读到D盘文件。  
先在D盘随便创建一个文件夹，我的路径是：`D:/78_codex/ai-skills`。  
使用下面的命令：  
```cmd
mklink /J "%USERPROFILE%\.codex\skills\ppt-master" "D:\78_codex\ai-skills\ppt-master"
```
成功后会显示：
```text
为 C:\Users\用户名\.codex\skills\ppt-master <<===>> D:\78_codex\ai-skills\ppt-master 创建的联接
```
由于这个ppt-master需要安装python依赖，有的并不需要，所以安装python依赖这里跳过了。  
不知道怎么安装可以把README.md发给ai看看。  


<br>

<br>

## 03 验证
***
我们先验证在C盘入口能访问：
```cmd
dir C:\Users\13509\.codex\skills\ppt-master
```
如果能看到和D盘中相同的目录内容，就说明成功了。  
重启codex，看看是否有看到刚刚安装的skill。  
看到就说明已经安装成功！  

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>