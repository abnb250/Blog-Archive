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
你打开一个网页，刷新一下，发现页面很快就出来了，好像没重新下载图片和样式。

你可能会想：刷新不就是重新请求一遍吗？为什么有的资源重新下载了，有的没有？

再往下想：浏览器到底发给服务器什么？服务器怎么表示“成功”“重定向”“找不到”“内部错误”？登录状态又是怎么记住的？为什么用 GET 做删除很危险？

这一章咱们用一条问题链往前走：

1. HTTP 到底是什么，为什么说它无状态？
2. 状态码怎么读，看到 304 为什么不是错误？
3. 常见字段解决什么问题，为什么少了 `Content-Length` 服务器可能不知道 body 到哪结束？
4. GET 和 POST 到底差在哪，为什么 GET 做删除会出事？

这篇的实验全部用 `curl` 观察真实报文，不需要写代码、不需要编译环境，照着敲就能看到结果。

<br>

<br>

## 优先级说明
***
这篇是 HTTP 的地基，面试和项目都会反复用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- HTTP 是请求-响应、本身无状态，状态靠 Cookie/Session/Token 补
- 状态码五大类，重点记 200、301、302、304、404、500、502
- 301 永久 / 302 临时 / 303 / 307 / 308 的区别
- `Host`、`Content-Length`、`Content-Type`、`Content-Encoding`、`Accept` 的作用
- `Content-Length` 是 body 字节长度，不是整个报文长度
- GET 安全幂等、参数在 URL；POST 非安全非幂等、数据在 body
- GET 的“安全”是不改变服务器状态，不是“加密安全”

**了解即可（知道大概，不展开）：**
- HTTP/2 的二进制帧和伪首部细节
- Cookie 的 `SameSite`、`HttpOnly` 等属性
- 分块传输的具体格式

**主线**：一次 HTTP 往返 = 客户端发请求单（请求行 + 请求头 + 可选 body）→ 服务器回结果单（状态行 + 响应头 + 可选 body）。状态码读结果，字段管地址/长度/格式/压缩/连接，GET/POST 定语义。

<br>

<br>

## 知识点 1：HTTP 是“请求单 + 结果单”，本身无状态
***
### 问题

HTTP 只是“超文本传输协议”七个字吗？它到底在两点之间传什么？为什么说它无状态，网站却还能记住你登录了？

**你可能会想**：HTTP 无状态，那网站怎么记住我登录了？是不是 HTTP 其实有状态？

不是。无状态说的是**协议本身不要求服务器记住上一次请求**，登录状态是应用层用 Cookie/Session/Token 另补的。

<br>

### 核心概念与方案

先把场景换成你熟悉的：**HTTP 就是客户端发一张“请求单”，服务器回一张“结果单”。**

- 请求单上写：我要什么（请求行）、我带了什么信息（请求头）、有没有附带数据（body）。
- 结果单上写：结果怎么样（状态行）、我返回了什么信息（响应头）、有没有附带数据（body）。

HTTP 通常基于 TCP，HTTP/3 基于 QUIC/UDP；采用请求-应答模式，本身是**无状态**的。登录状态常靠 Cookie/Session/Token 补上。

HTTP 的名字可以拆成三部分：**超文本、传输、协议**。协议意味着两个以上参与者，并有行为约定和规范；传输意味着两点之间可以双向传，中间允许中转；超文本意味着不只是纯文字，还包括图片、视频、超链接等。

所以 HTTP 不是“只传网页文字”，而是两点之间按约定交换各类资源。

<br>

### 最小实验：只看请求行里有没有 GET

环境前提：Linux/macOS，或 Windows WSL；已安装 `curl` 和 Python 3。下面只监听本机回环地址，不对外网开放。

终端 1 启动一个最小 HTTP 服务：

````bash
python3 -m http.server 8000 --bind 127.0.0.1
````

终端 2 发请求：

````bash
curl -v --max-time 5 http://127.0.0.1:8000/
````

**这次只看一件事**：`curl -v` 输出里以 `>` 开头的那一行，有没有 `GET`。

预期你会看到类似：

````text
> GET / HTTP/1.1
> Host: 127.0.0.1:8000
````

有 `GET`，说明客户端在发“获取资源”的请求。如果没有 `GET`，说明你发的不是 GET 请求，先检查命令。

先别管后面的 `Host`、`User-Agent`、响应头，那些是下一层细节。不同 Python 版本下状态行可能是 `HTTP/1.0` 或 `HTTP/1.1`，**需按环境核实**。

<br>

### 易错点与边界

坑一：**“从服务器到浏览器”的说法不完整**，也可以是服务器到服务器，所以用“两点之间”更准确。

坑二：**HTTP 无状态不等于不能登录**，Cookie/Session/Token 可以补状态；无状态说的是协议本身不要求服务器记住上一次请求。

坑三：**POST 不自动加密**，加密靠 HTTPS/TLS；HTTP 明文时 GET 和 POST 的数据都可能被抓包看到。

坑四：**HTTP/1.1 和 HTTP/2 的报文形式不同**。HTTP/1.1 用文本头部；HTTP/2 用二进制帧，`curl -v` 里看到的 `HTTP/2` 和伪首部是工具解码后的展示，不代表线上真的是文本，**需按版本核实**。

<br>

### 一句话本质

**HTTP 就是客户端发“请求单”，服务器回“结果单”；协议本身不记状态，状态要靠 Cookie/Session/Token 另补。**

<br>

### 自测

HTTP 是无状态协议，那网站为什么还能记住你登录了？

<details><summary>答案</summary>协议本身无状态，但服务器可以通过 Cookie、Session、Token 等机制在多次请求之间关联同一个用户。状态不是 HTTP 自带的，而是应用层补上的。</details>

<br>

<br>

## 知识点 2：状态码五大类怎么读
***
### 问题

200、301、304、404、500 到底差在哪？看到状态码，怎么快速判断是客户端的问题还是服务器的问题？

**你可能会想**：304 是不是出错了？为什么页面没重新下载？

304 不是错误，是**缓存有效**。服务器在说：“你本地那份还能用，别重新下了。”这正是“刷新页面没重新下载”的原因之一。

<br>

### 核心概念与方案

先把状态码换成你熟悉的回执编号，按第一位分五类：

- **1xx 提示信息**：如 101 Switching Protocols，协议切换，实际用得少。
- **2xx 成功**：200 OK；201 Created；204 无 body；206 分块下载/断点续传。
- **3xx 重定向**：301 永久重定向；302 临时重定向；303 See Other；304 资源未修改，走缓存；307/308 保持原方法重定向。301/302 常带 `Location`。
- **4xx 客户端错误**：400 报文有误；401 未认证；403 禁止访问；404 资源不存在；405 方法不允许。
- **5xx 服务器错误**：500 笼统内部错误；501 功能不支持；502 网关/代理访问后端出错；503 服务忙；504 网关超时。

排查时先看大类：**4xx 多半是请求方的问题，5xx 多半是服务端或网关的问题，3xx 看跳转和缓存**。

<br>

### 最小实验：只看返回码是不是 200 和 404

继续用知识点 1 的本地服务，终端 1 保持运行。终端 2 执行：

````bash
curl -s -o /dev/null -w "200?=%{http_code}\n" http://127.0.0.1:8000/
curl -s -o /dev/null -w "404?=%{http_code}\n" http://127.0.0.1:8000/not-exist-path
````

**这次只看一件事**：两条命令输出的数字是不是 `200` 和 `404`。

预期输出：

````text
200?=200
404?=404
````

第一条是 200，说明根路径存在；第二条是 404，说明这个路径没有资源。如果第一条不是 200，先检查服务是否启动、端口是否对。

301 和 304 依赖站点配置，可以另找公开站点观察，**需按环境核实**：

````bash
# 301：很多站点会把 http 永久重定向到 https
curl -s -i --max-time 10 http://example.com/ | head -n 5

# 304：带条件请求头，资源未修改时服务器回 304
curl -s -o /dev/null -w "304?=%{http_code}\n" \
  -H 'If-None-Match: "some-etag"' \
  --max-time 10 https://example.com/
````

301 的响应头里通常有 `Location`；304 没有 body。不是所有站点都会返回 301 或 304，看不到不代表命令错。

<br>

### 易错点与边界

坑一：**304 不是错误**，是缓存有效，别和 404 搞混。

坑二：**404 是资源没找到，500 是服务器内部错误**；403 是禁止访问，401 是未认证，三者不要混。

坑三：**301 和 302 都会跳转**，301 永久、302 临时，缓存行为不同；303 常用于 POST 后重定向到 GET，307/308 则强调保持原方法。

坑四：**502 不是后端业务返回的 500**，它常表示网关/代理访问后端失败；504 是网关等待后端超时。

坑五：**状态码不是业务码**。很多系统会在 200 响应体里放业务错误码，排查时要同时看 HTTP 状态码和响应内容。

<br>

### 一句话本质

**状态码就是服务器的“回执编号”：2 成功、3 重定向、4 客户端错、5 服务器错；304 是缓存有效，不是错误。**

<br>

### 自测

如果服务器返回 301 和 304，浏览器接下来分别会做什么？

<details><summary>答案</summary>301 表示永久重定向，浏览器通常会按 `Location` 跳转到新地址，并可能缓存这个重定向；304 表示资源未修改，浏览器通常直接使用本地缓存，不再重新下载响应体。</details>

<br>

<br>

## 知识点 3：常见字段干什么
***
### 问题

`Host`、`Content-Length`、`Connection`、`Content-Type`、`Content-Encoding` 这些字段，到底解决什么问题？

**你可能会想**：字段就是一些备注，少几个应该没关系吧？

但少了 `Content-Length`，服务器可能不知道 body 到哪结束；少了 `Host`，同一台服务器上的多个网站就分不清你要访问哪个。

<br>

### 核心概念与方案

先把字段换成你熟悉的“备注栏”：

- `Host`：指定服务器域名，让同一台服务器区分不同网站（虚拟主机）。HTTP/1.1 请求通常必须带 `Host`。
- `Content-Length`：表示消息 body 的**字节长度**，帮助接收方确定 body 的边界；如果使用分块传输，则可能没有这个字段，而用 `Transfer-Encoding: chunked`。
- `Connection: Keep-Alive`：希望复用长连接；HTTP/1.1 默认持久连接。注意 **HTTP Keep-Alive 和 TCP Keepalive 不是一回事**。
- `Content-Type`：告诉接收方 body 的数据格式，如 `text/html; charset=utf-8`；请求侧用 `Accept` 声明可接受格式。
- `Content-Encoding`：body 的压缩方法，如 gzip；请求侧用 `Accept-Encoding` 声明可接受压缩方法。

一句话：这些字段就是请求单和结果单上的“备注栏”，告诉对方地址、长度、格式、压缩方式和连接怎么处理。

<br>

### 最小实验：只看 POST 请求头里的 Content-Length 和 Content-Type

继续用本地服务，终端 1 保持运行。终端 2 发一个 POST：

````bash
curl -v -X POST -H 'Content-Type: text/plain' --data 'hello' http://127.0.0.1:8000/ -o /dev/null
````

**这次只看一件事**：请求头里有没有 `Content-Type: text/plain` 和 `Content-Length: 5`。

预期请求头类似：

````text
> POST / HTTP/1.1
> Host: 127.0.0.1:8000
> Content-Type: text/plain
> Content-Length: 5
````

`Content-Length: 5` 是因为 body 是 `hello`，正好 5 个字节。如果没有 `Content-Length` 或分块传输，服务器可能不知道 body 到哪结束。

`python3 -m http.server` 默认不支持 POST，响应可能是 501，但**请求头里的字段依然能看到**，这正是我们要观察的。先别管响应体。

<br>

### 易错点与边界

坑一：**HTTP Keep-Alive 和 TCP Keepalive 不是同一个东西**。前者是 HTTP 连接复用，后者是 TCP 层探测。

坑二：**`Content-Length` 是 body 长度**，不是整个报文长度，也不是“解决 TCP 粘包”的万能药；它解决的是 HTTP 消息边界。

坑三：**`Content-Type` 和 `Content-Encoding` 别混**：前者是格式，后者是压缩方法。看到 `Content-Encoding: gzip`，客户端需要按 gzip 解压；如果服务端没压缩，就不会有这个字段。

坑四：**`Accept` 是请求侧“我能接受什么”，`Content-Type` 是发送侧“我发的是什么”**；`Accept-Encoding` 和 `Content-Encoding` 也是同样的“协商 vs 实际”关系。

坑五：**HTTP/1.1 的 `Content-Length` 和分块传输不能随意冲突**；如果两者同时出现且不一致，接收方可能拒绝或按规范处理，**具体需按实现核实**。

<br>

### 一句话本质

**字段就是请求单和结果单上的“备注栏”：告诉对方地址、长度、格式、压缩方式和连接怎么处理。**

<br>

### 自测

服务器返回 `Content-Encoding: gzip`，客户端需要做什么？

<details><summary>答案</summary>客户端需要按 gzip 解压响应体，再按 `Content-Type` 解释内容。`Content-Encoding` 说的是 body 的压缩方法，不是内容格式。</details>

<br>

<br>

## 知识点 4：GET 与 POST
***
### 问题

GET 和 POST 有什么区别？都是安全幂等的吗？用 GET 做删除为什么会出事？

**你可能会想**：GET 和 POST 不就是参数一个在 URL、一个在 body 吗？功能上应该差不多吧？

差别不在“能不能传数据”，而在**语义**。GET 语义是“获取”，不应改变服务器状态；POST 语义是“处理请求负荷”，可能改变状态。用 GET 做删除，违反语义，可能被爬虫、预取、浏览器重试误触发。

<br>

### 核心概念与方案

按 RFC 语义：

- **GET 从服务器获取指定资源**，参数常放 URL，URL 通常只支持 ASCII；GET 是安全、幂等、可缓存的，可保存书签。
- **POST 根据请求负荷对指定资源做处理**，数据常放 body，格式可协商；POST 不是安全方法、不幂等，大部分实现不缓存，也不能保存书签。

这里的“安全”是 RFC 语义上的“不改变服务器状态”，**不是“加密安全”**。HTTP 明文传输时 GET 和 POST 数据抓包都能看到，要安全用 HTTPS。

实际开发不一定遵守 RFC；用 GET 做删除可能被爬虫或预取误触发；GET 理论上可带 body，但语义上不要求，很多实现也不支持。

<br>

### 最小实验：只看 GET 参数在 URL，POST 数据在 body

先看 GET 的请求行：

````bash
curl -v --max-time 5 "http://127.0.0.1:8000/?id=1&name=test" -o /dev/null
````

**这次只看一件事**：请求行里有没有 `?id=1&name=test`。

预期请求行：

````text
> GET /?id=1&name=test HTTP/1.1
> Host: 127.0.0.1:8000
````

参数在 URL 里，这就是 GET 的典型形式。

再看 POST：

````bash
curl -v -X POST -H 'Content-Type: text/plain' --data 'hello' http://127.0.0.1:8000/ -o /dev/null
````

**这次只看一件事**：请求行里有没有 `hello`，body 里有没有。

预期请求行和 body：

````text
> POST / HTTP/1.1
> Host: 127.0.0.1:8000
> Content-Type: text/plain
> Content-Length: 5
hello
````

路径里没有参数，数据在 body 里，靠 `Content-Length` 标长度。本地服务不处理 POST 也没关系，请求头已经能说明问题。

<br>

### 易错点与边界

坑一：**GET 不一定比 POST 不安全**，HTTP 明文时两者抓包都能看到，安全靠 HTTPS；但 GET 参数会进 URL、历史记录和日志，敏感信息不宜放 URL。

坑二：**POST 不自动幂等**，多次提交可能创建多条数据；需要幂等时要靠业务设计，如幂等键。

坑三：**GET 理论上可以带 body**，但 RFC 语义不要求，很多实现也不支持；不要依赖 GET body 传参。

坑四：**GET 的“安全”是不改变服务器状态**，不是“不加密也安全”；用 GET 做删除违反语义，还可能被爬虫、预取、浏览器重试误触发。

坑五：**POST 也不是“天然安全”**，它只是语义上用于处理负荷；加密、身份验证、防 CSRF 仍要另做。

<br>

### 一句话本质

**GET 是“我要看”，POST 是“我要交/改”；GET 安全幂等但不加密，POST 不自动幂等也不自动安全。**

<br>

### 自测

为什么用 GET 实现“删除博客”很危险？如果爬虫爬到会怎样？

<details><summary>答案</summary>GET 语义上应是安全、幂等的获取操作，不应改变服务器状态。用 GET 做删除违反语义，可能被爬虫、预取、浏览器重试或用户刷新误触发，导致数据被误删。删除应使用 POST/DELETE 等非安全方法，并做权限和防误操作设计。</details>

<br>

<br>

## 完整可复现示例：一次本地 HTTP 实验串起状态码、字段和方法
***
你需要 Linux/macOS 或 WSL，已安装 `curl` 和 Python 3。终端 1 启动本地服务：

````bash
python3 -m http.server 8000 --bind 127.0.0.1
````

终端 2 依次执行：

````bash
set -e

echo "=== 1. 200 和响应头 ==="
curl -s -i http://127.0.0.1:8000/ | head -n 8

echo "=== 2. 404 ==="
curl -s -o /dev/null -w "status=%{http_code}\n" http://127.0.0.1:8000/not-exist-path

echo "=== 3. GET 参数在 URL ==="
curl -v --max-time 5 "http://127.0.0.1:8000/?id=1" -o /dev/null 2>&1 | grep '^>'

echo "=== 4. POST 数据在 body，观察 Content-Length 和 Content-Type ==="
curl -v -X POST -H 'Content-Type: text/plain' --data 'hello' http://127.0.0.1:8000/ -o /dev/null 2>&1 | grep '^>'

echo "=== 5. 真实站点的 301/304（需外网，按环境核实） ==="
curl -s -i --max-time 10 http://example.com/ | head -n 5 || true
curl -s -o /dev/null -w "304?=%{http_code}\n" \
  -H 'If-None-Match: "some-etag"' --max-time 10 https://example.com/ || true
````

可能的输出/现象：

````text
=== 1. 200 和响应头 ===
HTTP/1.0 200 OK
Server: SimpleHTTP/...
Content-type: text/html
...

=== 2. 404 ===
status=404

=== 3. GET 参数在 URL ===
> GET /?id=1 HTTP/1.1
> Host: 127.0.0.1:8000
> User-Agent: curl/...
> Accept: */*

=== 4. POST 数据在 body ===
> POST / HTTP/1.1
> Host: 127.0.0.1:8000
> User-Agent: curl/...
> Accept: */*
> Content-Type: text/plain
> Content-Length: 5

=== 5. 真实站点的 301/304（需外网） ===
HTTP/1.1 301 Moved Permanently
Location: https://example.com/
...
304?=304
````

**关键观察点**：第 1 步看状态行是不是 200；第 2 步看是不是 404；第 3 步看请求行有没有 `?id=1`；第 4 步看请求头有没有 `Content-Type` 和 `Content-Length`；第 5 步看有没有 `Location` 或 304。

验证结果：状态码告诉你结果；响应头告诉你格式和长度；GET 把参数放在 URL，POST 把数据放在 body；301 带 `Location`，304 没有 body。

注意：第 5 步依赖外网和站点配置，看不到 301/304 不代表命令错；`head` 可能提前关闭输出，看到部分头部是正常的；不同 Python 和 `curl` 版本下状态行、头部顺序可能不同，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| 浏览器一直转圈或 `curl` 超时 | 网络、端口、服务是否启动 | `curl -v --max-time 5 URL`；`ss -ltnp \| grep ':8000'` | 先分清“连不上”还是“连上了但响应慢” |
| 返回 404 | 路径、方法、路由 | `curl -i` 看请求行和响应行；查服务端路由 | 404 是资源没找到，不一定是服务器坏了 |
| 返回 403/401 | 权限、认证 | 看请求是否带 Cookie/Token；看响应头 | 401 未认证，403 禁止访问，别和 404 混 |
| 返回 301/302 但没跳转 | 是否跟随重定向 | `curl -i` 看 `Location`；`curl -L` 跟随 | 301 永久、302 临时；303/307/308 语义不同 |
| 返回 304 但页面没更新 | 缓存策略 | 看 `Cache-Control`、`ETag`、`Last-Modified` | 304 表示缓存有效，不是错误 |
| 返回 502/504 | 网关和后端 | 查网关日志、后端是否存活、超时配置 | 502 常是网关访问后端失败，504 是网关等待超时 |
| POST 收不到 body | `Content-Length`、`Content-Type`、分块传输 | `curl -v` 看请求头；服务端读 body 的逻辑 | 没有正确长度或分块时，服务端可能不知道 body 到哪结束 |
| 长连接表现异常 | `Connection`、中间件、HTTP 版本 | `curl -v` 看连接复用；查代理配置 | HTTP Keep-Alive 是连接复用，不是 TCP Keepalive |
| 用 GET 删除了数据 | 方法语义 | 查访问日志、爬虫、预取、浏览器重试 | GET 应安全幂等，删除应使用非安全方法并做确认 |

<br>

<br>

## 常见误区
***
- **把 HTTP 无状态当成不能登录**：Cookie/Session/Token 可以补状态。
- **把 304 当成错误**：304 是缓存有效，没有响应体。
- **把 404 和 500 混为一谈**：404 是资源没找到，500 是服务器内部错误。
- **把 502 当成后端业务 500**：502 常是网关/代理访问后端失败。
- **把 `Content-Length` 当成整个报文长度**：它是 body 的字节长度。
- **把 `Content-Type` 和 `Content-Encoding` 混了**：前者是格式，后者是压缩方法。
- **把 HTTP Keep-Alive 和 TCP Keepalive 混了**：前者是 HTTP 连接复用，后者是 TCP 层探测。
- **以为 GET 一定比 POST 安全**：明文 HTTP 下两者都能被抓包，安全靠 HTTPS；但 GET 参数会进 URL、日志和历史记录。
- **以为 POST 自动幂等**：POST 多次提交可能创建多条数据，幂等要靠业务设计。
- **以为 GET 不能带 body 所以绝对不能带**：协议语义不要求，很多实现不支持，不要依赖它传参。

<br>

<br>

## 自测三问
***
1. 200、301、304、404、500 分别代表什么？301 和 302 有什么区别？
2. `Content-Type` 和 `Content-Encoding` 有什么区别？`Accept` 和 `Content-Type` 又分别是谁对谁说的？
3. GET 和 POST 在语义、参数位置、安全性、幂等性上有什么不同？为什么用 GET 做删除危险？

<details><summary>参考答案</summary>1. 200 成功；301 永久重定向；304 资源未修改、走缓存；404 资源不存在；500 服务器内部错误。301 永久、302 临时，缓存行为不同；303 常用于 POST 后重定向到 GET，307/308 强调保持原方法。2. `Content-Type` 是 body 的数据格式，`Content-Encoding` 是 body 的压缩方法；`Accept` 是请求方说“我能接受什么格式”，`Content-Type` 是发送方说“我发的是什么格式”。3. GET 语义是获取资源，参数常在 URL，安全且幂等，可缓存；POST 语义是处理请求负荷，数据常在 body，非安全且不幂等，通常不缓存。两者都不自动加密，安全要靠 HTTPS；GET 参数会进 URL 和日志，敏感信息不宜放 URL。用 GET 做删除违反“安全”语义，可能被爬虫、预取、浏览器重试误触发，导致误删。</details>

<br>

<br>

## 总结
***
**HTTP 是请求-响应式的应用层协议：状态码读结果，字段管地址/长度/格式/压缩/连接，GET 用于获取、POST 用于提交或修改；无状态靠 Cookie 等补，安全靠 HTTPS。**

<br>

<br>

## 参考资料
***
- RFC 9110：HTTP Semantics
- RFC 9112：HTTP/1.1
- RFC 9113：HTTP/2
- RFC 9114：HTTP/3
- RFC 6265：HTTP State Management Mechanism（Cookie）
- 需按环境核实：不同 `curl` 和 Python 版本的输出差异；`python3 -m http.server` 的响应版本和对 POST 的处理；301/304 是否出现取决于站点配置和外网环境；HTTP/2 中 `Host`/`:authority` 的具体行为。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>