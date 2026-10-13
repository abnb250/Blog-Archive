# 使用Skill
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  

<br>

<br>

## 00 前言
***
我这里使用的是之前下载的Codewhale，  
是用的是GitHub上花书的女娲skill。  
这次我的目标就是：  
根据我对一个人收集的语录、文本资料等，  
蒸馏出一个属于这个人的skill。  

<br>

<br>

## 01 下载Skill
***
首先是这个skill的GitHub链接：  
https://github.com/alchaincyf/nuwa-skill  
然后，是如何安装这个skill，  
跟你的agent对话，比如我的codewhale：  
```text
帮我安装这个 skill：https://github.com/alchaincyf/nuwa-skill
```
这是最简单的方式之一，或者你可以加我的群，我发在群里面。  

<br>

<br>

## 02 使用Skill
***
这里依旧是使用我用的codewhale为例。  
首先查看是否在skill列表内：
```bash
/skills
```
使用方向键上下寻找女娲skill，可以会重命名。  
然后使用下面命令直接调用：
```bash
/skill [skill名]
```

<br>

直接使用提示词：
```text
用女娲 skill，基于工作区中的 `文本.md` 这个文件，蒸馏一个新的 Skill，名称为 lin-min。

要求：
1. 素材仅限该文件内容，不要自行搜索或补充外部信息。
2. 生成后把 SKILL.md 保存到工作区的 `lin-min/` 目录下。
```

<br>

<br>

## 03 添加README
***
codewhale直接把skill做好了，  
内容可能有点多，但是我不想读，  
直接让ai写一个README.md。  
最后放到codewhale可以调用的skill路径中：
C:\Users\用户名\.codewhale\skills

<br>

<br>