# 03 HTTP 基础：请求-响应、状态码、字段、GET 与 POST
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
打开网页时，浏览器到底发给服务器什么？服务器怎么表示成功、重定向、找不到或内部错误？登录状态又是怎么记住的？  
这一章咱们把 HTTP 最常用的基础串起来：**HTTP 是什么 → 状态码怎么读 → 常见字段干什么 → GET 和 POST 到底差在哪。**  
先别急着背状态码表，先跟着几个具体问题走：浏览器发的“请求单”长什么样？服务器回的“结果单”又长什么样？为什么有时候刷新页面不会重新下载？为什么用 GET 做删除很危险？

<br>

<br>

## 知识点 1：HTTP 是什么
***
### 问题
HTTP 只是“超文本传输协议”七个字吗？它到底在两点之间传什么？为什么说它无状态，网站却还能记住你登录了？

<br>

### 方案
更准确地说：**HTTP 是计算机世界里专门在两点之间传输文字、图片、音频、视频等超文本数据的约定和规范。**  
它通常基于 TCP（HTTP/3 基于 QUIC/UDP），采用请求-应答模式，本身是**无状态**的；登录状态常靠 Cookie/Session/Token 补上。  
HTTP 的名字可以拆成三部分：**超文本、传输、协议**。协议意味着两个以上参与者，并有行为约定和规范；传输意味着两点之间可以双向传，中间允许中转；超文本意味着不只是纯文字，还包括图片、视频、超链接等。  
所以 HTTP 不是“只传网页文字”，而是两点之间按约定交换各类资源。

<br>

### 最小示例
用 `curl` 看一次 HTTP 响应头，你会看到 HTTP 实际交换的内容：

```bash
curl -I --max-time 10 https://example.com/
```

预期输出可能类似：

```text
HTTP/2 200
content-type: text/html; charset=UTF-8
content-length: 1256
server: ...
```

结论：状态行、头部字段、内容类型都是 HTTP 协议的一部分。不同站点、不同 HTTP 版本下输出可能不同，重点看结构。

<br>

### 一句话本质
**HTTP 就是浏览器发“请求单”，服务器回“结果单”。**

<br>

### 易错点
坑一：**“从服务器到浏览器”的说法不完整**，也可以是服务器到服务器，所以用“两点之间”更准确。  
坑二：**HTTP 无状态不等于不能登录**，Cookie/Session/Token 可以补状态。  
坑三：**POST 不自动加密**，加密靠 HTTPS/TLS；HTTP 明文时 GET 和 POST 的数据都可能被抓包看到。

<br>

### 自测
HTTP 是无状态协议，那网站为什么还能记住你登录了？

<br>

<br>

## 知识点 2：状态码五大类怎么读
***
### 问题
200、301、304、404、500 到底差在哪？面试常问，实际排查也常遇到。看到状态码，怎么快速判断是客户端的问题还是服务器的问题？

<br>

### 方案
状态码按第一位分五类，先记大类，再记常用码：  
- **1xx 提示信息**：如 101 Switching Protocols，协议切换，实际用得少。  
- **2xx 成功**：200 OK；204 无 body；206 分块下载/断点续传。  
- **3xx 重定向**：301 永久重定向；302 临时重定向；304 资源未修改，走缓存。301/302 常带 `Location`。  
- **4xx 客户端错误**：400 报文有误；403 禁止访问；404 资源不存在。  
- **5xx 服务器错误**：500 笼统内部错误；501 功能不支持；502 网关/代理访问后端出错；503 服务忙。  
排查时先看大类：4xx 多半是请求方的问题，5xx 多半是服务端或网关的问题，3xx 看跳转和缓存。

<br>

### 最小示例
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://example.com/
```

输出可能类似：

```text
200
```

结论：`200` 表示请求成功；如果看到 `404` 是资源没找到，`500` 是服务器内部错误，`304` 是缓存有效。  
注意：不同站点返回码可能不同，这里重点学会读码。

<br>

### 一句话本质
**状态码就是服务器的“回执编号”：2 成功、3 重定向、4 客户端错、5 服务器错。**

<br>

### 易错点
坑一：**304 不是错误**，是缓存有效，别和 404 搞混。  
坑二：**404 是资源没找到，500 是服务器内部错误**。  
坑三：**301 和 302 都会跳转**，301 永久、302 临时，缓存行为不同。  
坑四：**502 不是后端业务返回的 500**，它常表示网关/代理访问后端失败。

<br>

### 自测
如果服务器返回 301 和 304，浏览器接下来分别会做什么？

<br>

<br>

## 知识点 3：常见字段干什么
***
### 问题
Host、Content-Length、Connection、Content-Type、Content-Encoding 这些字段，到底解决什么问题？为什么少了某个字段，服务器可能不知道 body 到哪结束？

<br>

### 方案
- `Host`：指定服务器域名，让同一台服务器区分不同网站（虚拟主机）。  
- `Content-Length`：表示消息 body 的字节长度，帮助接收方确定 HTTP 消息 body 的边界；如果使用分块传输，则可能没有这个字段。  
- `Connection: Keep-Alive`：希望复用长连接；HTTP/1.1 默认持久连接，但兼容老版本或中间件时会显式指定。注意 **HTTP Keep-Alive 和 TCP Keepalive 不是一回事**。  
- `Content-Type`：告诉客户端数据格式，如 `text/html; charset=utf-8`；请求侧用 `Accept` 声明可接受格式。  
- `Content-Encoding`：数据压缩方法，如 gzip；请求侧用 `Accept-Encoding` 声明可接受压缩方法。  
一句话：这些字段就是请求单和结果单上的“备注栏”，告诉对方地址、长度、格式、压缩方式和连接怎么处理。

<br>

### 最小示例
```http
Host: www.A.com
Content-Length: 1000
Connection: Keep-Alive
Content-Type: text/html; Charset=utf-8
Content-Encoding: gzip
Accept: */*
Accept-Encoding: gzip, deflate
```

结论：`Host` 帮服务器区分网站；`Content-Length` 帮定 body 边界；`Connection` 管连接复用；`Content-Type` 管格式；`Content-Encoding` 管压缩。请求侧的 `Accept`、`Accept-Encoding` 是在说“我能接受什么”。

<br>

### 一句话本质
**字段就是请求单和结果单上的“备注栏”：告诉对方地址、长度、格式、压缩方式和连接怎么处理。**

<br>

### 易错点
坑一：**HTTP Keep-Alive 和 TCP Keepalive 不是同一个东西**。前者是 HTTP 连接复用，后者是 TCP 层探测。  
坑二：**Content-Length 是 body 长度**，不是整个报文长度，也不是“解决 TCP 粘包”的万能药；它解决的是 HTTP 消息边界。  
坑三：**Content-Type 和 Content-Encoding 别混**：前者是格式，后者是压缩方法。  
坑四：`Accept` 是请求侧“我能接受什么”，`Content-Type` 是发送侧“我发的是什么”。

<br>

### 自测
服务器返回 `Content-Encoding: gzip`，客户端需要做什么？

<br>

<br>

## 知识点 4：GET 与 POST
***
### 问题
GET 和 POST 有什么区别？都是安全幂等的吗？用 GET 做删除为什么会出事？

<br>

### 方案
按 RFC 语义：  
- **GET 从服务器获取指定资源**，参数常放 URL，URL 通常只支持 ASCII；浏览器和服务器对 URL 长度有限制，但 HTTP 协议本身没规定统一长度。GET 是安全、幂等、可缓存的，可保存书签。  
- **POST 根据请求负荷对指定资源做处理**，数据常放 body，格式可协商，浏览器一般不限 body 大小；POST 不是安全方法、不幂等，大部分实现不缓存，也不能保存书签。  
这里的“安全”是 RFC 语义上的“不改变服务器状态”，**不是“加密安全”**。实际开发不一定遵守 RFC；用 GET 做删除可能被爬虫或预取误触发；GET 理论上可带 body，但语义上不要求，很多实现也不支持；HTTP 明文传输时 GET 和 POST 数据抓包都能看到，要安全用 HTTPS。

<br>

### 最小示例
GET 请求常见形式：

```http
GET /article?id=1 HTTP/1.1
Host: example.com
```

POST 请求常见形式：

```http
POST /comment HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

text=hello
```

结论：GET 常用于“我要看”，参数在 URL；POST 常用于“我要提交/修改”，数据在 body。

<br>

### 一句话本质
**GET 是“我要看”，POST 是“我要交/改”。**

<br>

### 易错点
坑一：**GET 不一定比 POST 不安全**，HTTP 明文时两者抓包都能看到，安全靠 HTTPS；但 GET 参数会进 URL、历史记录和日志，敏感信息不宜放 URL。  
坑二：**POST 不自动幂等**，多次提交可能创建多条数据。  
坑三：**GET 理论上可以带 body**，但 RFC 语义不要求，很多实现也不支持。  
坑四：**GET 的“安全”是不改变服务器状态**，不是“不加密也安全”。

<br>

### 自测
为什么用 GET 实现“删除博客”很危险？如果爬虫爬到会怎样？

<br>

<br>

## 完整可复现示例：看状态码、字段和 GET 请求
***
```bash
set -e

echo "=== 1. 查看状态码 ==="
curl -s -o /dev/null -w "%{http_code}\n" https://example.com/

echo "=== 2. 查看响应头 ==="
curl -sI https://example.com/ | head -n 20

echo "=== 3. 发送一个 GET 请求 ==="
curl -v --max-time 10 "https://example.com/?q=test" -o /dev/null
```

输出/现象/结论可能类似：

```text
=== 1. 查看状态码 ===
200

=== 2. 查看响应头 ===
HTTP/2 200
content-type: text/html; charset=UTF-8
content-length: ...
server: ...

=== 3. GET 请求 ===
> GET /?q=test HTTP/2
> Host: example.com
< HTTP/2 200
```

验证结果：状态码告诉你结果；响应头告诉你格式、长度、服务器信息；GET 请求把参数放在 URL 中。POST 则是把数据放在 body 中，常用于提交。  
注意：`example.com` 的返回头会随站点和网络环境变化；`head` 可能提前关闭输出，看到部分头部是正常的。

<br>

<br>

## 自测三问
***
1. 200、301、304、404、500 分别代表什么？  
2. `Content-Type` 和 `Content-Encoding` 有什么区别？`Accept` 和 `Content-Type` 又分别是谁对谁说的？  
3. GET 和 POST 在语义、参数位置、安全性、幂等性上有什么不同？

<br>

<br>

## 一句话总结全章
***
**HTTP 是请求-响应式的应用层协议：状态码读结果，字段管地址/长度/格式/压缩/连接，GET 用于获取、POST 用于提交或修改；无状态靠 Cookie 等补，安全靠 HTTPS。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>