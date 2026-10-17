# 04 codex的skill使用
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  

<br>

<br>

## 00 前言
***
本次我们给桌面端codex连接上一个mcp，SlideForge，这是一个远程HTTP+Oauth认证的mcp。  
我打算用它来渲染PPT的画面。  
这个mcp需要本机有可用浏览器。  

<br>

<br>

## 01 连接与授权
***
首先需要在终端（cmd）中启用代理，我这里用的是cmd的命令：
```cmd
set HTTPS_PROXY="http://127.0.0.1:7890"
set HTTP_PROXY="http://127.0.0.1:7890"
```
在终端（cmd）中一条命令完成连接与授权：  
```bash
codex mcp add slideforge --url https://api.slideforge.dev/mcp/
```

<br>

<br>

## 02 验证连接
***
输入：
```
codex mcp list
```
会得到类似：
```
Name        Url                              Bearer Token Env Var  Status   Auth
slideforge  https://api.sli... -                     enabled  OAuth
```
就成功了。  

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>