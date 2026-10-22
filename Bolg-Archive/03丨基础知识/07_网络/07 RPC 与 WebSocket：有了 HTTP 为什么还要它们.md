# 07 RPC 与 WebSocket：有了 HTTP 为什么还要它们
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
你在微服务里调一个用户服务，用 HTTP 传 JSON 行不行？行，但每次请求都带一堆重复头部，序列化也偏重。内部服务之间这么调，是不是有点浪费？

再想一个场景：网页游戏里，服务器要实时告诉你“对手放技能了”。HTTP 是客户端不请求，服务器就不回。你总不能每秒发几十个请求问“有消息吗”吧？

这两个场景，正好引出两个问题：

1. HTTP 做内部调用够不够？为什么公司内部微服务还用 RPC？
2. HTTP 不能让服务器主动推怎么办？为什么网页游戏用 WebSocket，而不是轮询？

<br>

<br>

## 优先级说明
***
这篇是应用层进阶，面试常问，项目里做微服务和实时通信也会遇到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- 裸 TCP 是字节流，没有消息边界，应用层要自己定“一条消息到哪结束”
- RPC 不是某一个具体协议，而是一种“像调本地方法一样调远端”的调用方式
- gRPC 底层用 HTTP/2，RPC 和 HTTP 不是对立的
- HTTP 请求-响应由客户端驱动，服务器不能主动发消息
- 轮询、长轮询、WebSocket 的区别
- WebSocket 握手借 HTTP 升级，101 之后就不是 HTTP 了
- WebSocket 是应用层协议，名字里的 socket 不是系统 socket

**了解即可（知道大概，不展开）：**
- Protobuf 的编码细节
- Consul/Etcd 的服务发现流程
- WebSocket 帧的 opcode 和掩码细节
- SSE（Server-Sent Events）与 WebSocket 的对比

**主线**：裸 TCP 没边界 → 应用层协议解决边界和调用方式 → HTTP 适合通用请求-响应 → RPC 让内部调用更紧凑高效 → HTTP 不能服务器主动推 → 轮询/长轮询是补丁 → WebSocket 握手升级后全双工。

<br>

<br>

## 知识点 1：裸 TCP 没边界，RPC 是“像调本地方法一样调远端”
***
### 问题

TCP 能收发数据，为什么不能直接用？我直接 `send` 一段 JSON，对面 `recv` 不就行了吗？RPC 到底是什么，它和 HTTP 是什么关系？

**你可能会想**：TCP 是可靠的字节流，我发什么对方收什么，直接用不就行了？

但 TCP 是**字节流**，没有消息边界。你发“夏洛”和“特烦恼”，对面可能一次收成“夏洛特烦恼”，也可能分两次收。直接裸用 TCP，应用层得自己定义“一条消息到哪结束”。

<br>

### 核心概念与方案

先把问题换成你熟悉的场景：**TCP 像一根水管，你往里倒水，对方接到的是连续的水流，不知道你倒了几杯。**

要分清“一杯”和“下一杯”，就得在应用层约定：**消息头 + 消息体。** 头里放长度，体里放内容，接收方先读头、再按长度读体。

这就是应用层协议要解决的事。HTTP 和各类 RPC 都是基于传输层之上的应用层协议。

**RPC 本身不是某一个具体协议**，而是一种“像调本地方法一样调远端方法”的调用方式；gRPC、Thrift、Dubbo 才是具体实现。RPC 不一定非基于 TCP，也可以基于 UDP 或 HTTP。

RPC 和 HTTP 的常见区别可以这样看：

- **服务发现**：HTTP 常用 DNS；RPC 常用 Consul、Etcd、Redis 等注册中心。
- **连接管理**：HTTP/1.1 常靠 Keep-Alive 复用连接；RPC 常用连接池，减少频繁建连。
- **传输内容**：HTTP/1.1 头部冗余、常用 JSON；RPC 常用 Protobuf 等更紧凑的序列化，性能更好。

但注意：**HTTP/2 性能可能比很多 RPC 还好**，gRPC 底层用的就是 HTTP/2。所以别把 RPC 和 HTTP 对立起来，它们经常互相借用。

<br>

### 最小实验：只看裸 TCP 一次 recv 能不能收全

环境前提：Linux/macOS，或 Windows WSL；已安装 Python 3。下面只在本机回环地址上运行，不对外网开放。

先启动一个裸 TCP 服务，保存为 `raw_tcp_server.py`：

````python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(("127.0.0.1", 13000))
s.listen(1)
print("raw TCP server listening on 127.0.0.1:13000")
conn, addr = s.accept()
data = conn.recv(1024)
print("server received:", data.decode())
conn.close()
s.close()
````

终端 1 运行：

````bash
python3 raw_tcp_server.py
````

再写一个客户端，分两次 `send`，保存为 `raw_tcp_client.py`：

````python
import socket
import time

c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
c.connect(("127.0.0.1", 13000))
c.sendall("夏洛".encode())
time.sleep(0.1)
c.sendall("特烦恼".encode())
c.close()
````

终端 2 运行：

````bash
python3 raw_tcp_client.py
````

**这次只看一件事**：服务端一次 `recv` 收到的内容是“夏洛”和“特烦恼”分两次，还是合并成“夏洛特烦恼”。

预期现象：服务端可能打印 `server received: 夏洛特烦恼`，也可能只打印一部分。**具体取决于发送时机、缓冲区大小和系统调度，需按环境核实。**

这个实验说明：TCP 不保证你一次 `recv` 就拿到一条完整消息，也不保证两条消息不粘在一起。**消息边界要应用层自己定。**

<br>

### 易错点与边界

坑一：**RPC 不是某一个协议**，gRPC、Thrift 才是具体实现。

坑二：**RPC 不一定基于 TCP**，也可以基于 UDP 或 HTTP。

坑三：**HTTP/2 性能可能比很多 RPC 好**，gRPC 底层就是 HTTP/2。

坑四：**裸 TCP 不是不能用**，而是你得自己定消息边界和协议；RPC 和 HTTP 都是在帮你做这件事。

坑五：**TCP 的“粘包”不是 bug**，是字节流的正常表现；应用层要按长度、分隔符或固定格式划分消息。

<br>

### 一句话本质

**TCP 是字节流，没有消息边界；RPC 和 HTTP 都是应用层协议，帮你定边界、管连接、做序列化；RPC 不是具体协议，而是一种“像调本地方法一样调远端”的调用方式。**

<br>

### 自测

为什么公司内部微服务常用 RPC，而对外接口常用 HTTP？

<details><summary>答案</summary>内部微服务追求紧凑高效，RPC 常用 Protobuf 等更紧凑的序列化、连接池和服务发现，减少头部冗余和建连开销；对外接口追求通用、可读、跨语言，HTTP + JSON 更常见。但两者不是对立的，gRPC 底层就用 HTTP/2。</details>

<br>

<br>

## 知识点 2：HTTP 不能让服务器主动推，WebSocket 借 HTTP 握手升级
***
### 问题

HTTP 是请求-响应模式，客户端不问，服务器就不能主动发消息。网页游戏要实时推送，怎么办？定时轮询、长轮询、WebSocket 到底差在哪？

**你可能会想**：服务器主动推，不就是服务器先发数据吗？这有什么难的？

难在 HTTP 的通信模型：**HTTP/1.1 严格说是请求-响应模式，服务器不能主动发起消息。** 口语里说“半双工”是为了强调“不能同时双向主动发”，更准确的说法是“通信由客户端请求驱动”。

<br>

### 核心概念与方案

先把三种方案换成你熟悉的场景：

- **定时轮询**：你每隔几秒去问一次“有消息吗？”——延迟高，空请求多。
- **长轮询**：你问一次，服务器先不急着回，等有消息或超时再回；你收到后立刻再问。——少一些空请求，但本质仍是客户端驱动。
- **WebSocket**：你先发一个 HTTP 请求说“咱们升级成双向通信吧”，服务器同意后，双方就能互相主动发消息。——真正的全双工。

WebSocket 的握手借 HTTP 升级：

- 请求头带 `Connection: Upgrade`、`Upgrade: WebSocket`、`Sec-WebSocket-Key`。
- 服务器返回 **101 Switching Protocols** 和 `Sec-WebSocket-Accept`。
- 升级完成后就不再是 HTTP，而用 WebSocket 帧通信。

帧里有 opcode（1 文本、2 二进制、8 关闭等）和 payload 长度字段，接收方按帧解析。

<br>

### 最小实验：只看握手请求里有没有 Upgrade: WebSocket

环境前提：Linux/macOS，或 Windows WSL；已安装 `curl`，需要外网。下面只观察握手请求头，不需要真的升级成功。

用 `curl` 发一个 WebSocket 升级请求：

````bash
curl -v --max-time 10 \
  -H 'Connection: Upgrade' \
  -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' \
  -H 'Sec-WebSocket-Version: 13' \
  http://example.com/chat -o /dev/null
````

**这次只看一件事**：`curl -v` 输出里以 `>` 开头的请求头，有没有 `Upgrade: websocket`。

预期请求头类似：

````text
> GET /chat HTTP/1.1
> Host: example.com
> Connection: Upgrade
> Upgrade: websocket
> Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
> Sec-WebSocket-Version: 13
````

有这几行，说明 WebSocket 握手首先是一个带特殊 Header 的 HTTP 请求。

`example.com` 不支持 WebSocket，所以响应很可能是 `200` 或其他状态，而不是 `101`。**这正好说明：握手请求是 HTTP，服务器支不支持是另一回事。** 换成支持 WebSocket 的服务器，才会看到 `101 Switching Protocols`。

<br>

### 易错点与边界

坑一：**WebSocket 不是直接用裸 socket 编程**，它是一套应用层协议；名字里的 socket 不是系统 socket 的意思。

坑二：**“WebSocket 基于 HTTP 的新协议”不准确**，它只在握手时用 HTTP，升级后与 HTTP 无关。

坑三：**长轮询不是 WebSocket**，长轮询仍是客户端发请求、服务器等一段时间再响应；WebSocket 升级后是真正的双向帧通信。

坑四：**101 之后就不是 HTTP 了**，别再按 HTTP 报文格式去解析后面的数据。

坑五：**`Sec-WebSocket-Accept` 不是随便写的**，它是拿 `Sec-WebSocket-Key` 加上固定 GUID 做 SHA-1 再 Base64 得到的，用来确认服务器真的理解 WebSocket 握手。

<br>

### 一句话本质

**HTTP 是“你不问我不答”，WebSocket 是“握手后双方都能主动说”；握手借 HTTP 升级，101 之后就不是 HTTP 了。**

<br>

### 自测

长轮询和 WebSocket 都能实现“服务器推送”，网页游戏更该选哪个？为什么？

<details><summary>答案</summary>网页游戏更该选 WebSocket。游戏需要频繁、低延迟的双向交互，长轮询每次收到消息后还要重新发请求，空请求和延迟都更高；WebSocket 升级后是全双工帧通信，服务器可以随时主动推，延迟更低、开销更小。简单、低频的推送场景可以用长轮询或 SSE，不必上 WebSocket。</details>

<br>

<br>

## 完整可复现示例：用 curl 观察 WebSocket 握手请求和响应
***
你需要 Linux/macOS 或 WSL，已安装 `curl`，需要外网。下面每个步骤只看一件事，不展开帧解析。

````bash
set -e

echo "=== 1. 发 WebSocket 升级请求，看请求头 ==="
curl -v --max-time 10 \
  -H 'Connection: Upgrade' \
  -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' \
  -H 'Sec-WebSocket-Version: 13' \
  http://example.com/chat -o /dev/null 2>&1 | grep '^>'

echo "=== 2. 看响应状态行 ==="
curl -s -o /dev/null -w "status=%{http_code}\n" --max-time 10 \
  -H 'Connection: Upgrade' \
  -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' \
  -H 'Sec-WebSocket-Version: 13' \
  http://example.com/chat || true
````

可能的输出/现象：

````text
=== 1. 请求头 ===
> GET /chat HTTP/1.1
> Host: example.com
> User-Agent: curl/...
> Accept: */*
> Connection: Upgrade
> Upgrade: websocket
> Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
> Sec-WebSocket-Version: 13

=== 2. 响应状态行 ===
status=200
````

**关键观察点**：第 1 步看请求头有没有 `Upgrade: websocket`；第 2 步看状态码是不是 `101`。

`example.com` 不支持 WebSocket，所以状态码很可能是 `200` 而不是 `101`。**这说明握手请求本身是 HTTP，服务器支不支持是另一回事。** 换成支持 WebSocket 的服务器，才会看到 `101 Switching Protocols`。

注意：不同站点、不同网络环境下响应可能不同；`curl` 不会真的升级成 WebSocket 帧通信，它只帮你观察握手请求和响应，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| 裸 TCP 收到的数据粘在一起或截断 | 消息边界 | 看应用层有没有长度字段/分隔符 | TCP 是字节流，不保证一次 recv 一条消息；要应用层自己划分 |
| RPC 调用超时 | 服务发现、连接池、序列化 | 查注册中心、连接池配置、超时设置 | RPC 依赖服务发现和连接管理，任一环出问题都可能超时 |
| gRPC 调不通 | HTTP/2、TLS、代理 | 看客户端是否支持 HTTP/2；查代理是否降级 | gRPC 底层用 HTTP/2，中间代理不支持 HTTP/2 可能失败 |
| 轮询延迟高、请求多 | 是否该换长轮询/WebSocket | 看请求频率和延迟要求 | 定时轮询空请求多；高频双向交互更适合 WebSocket |
| WebSocket 握手返回 200 而不是 101 | 服务器是否支持 WebSocket | `curl -v` 看请求头；查服务端路由 | 服务器没配置 WebSocket 升级，或路径不对 |
| 101 之后收不到消息 | 帧解析、掩码、opcode | 抓包看 WebSocket 帧；查客户端/服务端帧处理 | 101 之后不是 HTTP，要按 WebSocket 帧格式解析；客户端帧要掩码 |
| 长轮询和 WebSocket 分不清 | 通信模型 | 看请求是否持续、是否升级 | 长轮询仍是 HTTP 请求-响应；WebSocket 101 后是双向帧通信 |

<br>

<br>

## 常见误区
***
- **以为 RPC 是某一个协议**：RPC 是调用方式，gRPC、Thrift、Dubbo 才是具体实现。
- **以为 RPC 一定基于 TCP**：RPC 也可以基于 UDP 或 HTTP；gRPC 底层就是 HTTP/2。
- **把 RPC 和 HTTP 对立起来**：它们经常互相借用，gRPC 就用 HTTP/2。
- **以为裸 TCP 直接 send/recv 就行**：TCP 没有消息边界，应用层要自己定协议。
- **把 WebSocket 当成裸 socket 编程**：WebSocket 是应用层协议，名字里的 socket 不是系统 socket。
- **以为 WebSocket 全程基于 HTTP**：只在握手时用 HTTP，101 之后与 HTTP 无关。
- **把长轮询当成 WebSocket**：长轮询仍是 HTTP 请求-响应，WebSocket 升级后是双向帧通信。
- **以为 101 之后还能按 HTTP 解析**：101 之后是 WebSocket 帧，不是 HTTP 报文。

<br>

<br>

## 自测三问
***
1. RPC 和 HTTP 的区别主要在哪几个方面？为什么说 gRPC 底层用 HTTP/2 并不矛盾？
2. WebSocket 握手用了 HTTP 的什么机制？升级后还用 HTTP 吗？
3. 长轮询和 WebSocket 各自适合什么场景？网页游戏为什么更倾向 WebSocket？

<details><summary>参考答案</summary>1. 区别主要在服务发现、连接管理、传输内容：HTTP 常用 DNS、Keep-Alive、JSON；RPC 常用注册中心、连接池、Protobuf。gRPC 底层用 HTTP/2 不矛盾，因为 RPC 是调用方式，HTTP/2 是传输承载，gRPC 借 HTTP/2 做多路复用和头部压缩，同时用 Protobuf 做紧凑序列化。2. WebSocket 握手借 HTTP 的 Upgrade 机制：请求头带 `Connection: Upgrade`、`Upgrade: WebSocket`、`Sec-WebSocket-Key`，服务器返回 101 和 `Sec-WebSocket-Accept`；升级后不再用 HTTP，而是 WebSocket 帧通信。3. 长轮询适合低频、简单推送；WebSocket 适合高频、低延迟、双向交互，比如网页游戏、聊天。网页游戏要实时推技能、位置，长轮询延迟和空请求都更高，所以更倾向 WebSocket。</details>

<br>

<br>

## 总结
***
**RPC 是“像调本地方法一样调远程”的调用方式，追求内部紧凑高效；WebSocket 借 HTTP 握手升级，之后提供全双工通信，解决服务器主动推送问题；两者都不是要取代 HTTP，而是补 HTTP 在特定场景下的不足。**

<br>

<br>

## 参考资料
***
- RFC 6455：The WebSocket Protocol
- RFC 9110：HTTP Semantics
- RFC 9113：HTTP/2
- gRPC 官方文档：https://grpc.io/docs/
- Protocol Buffers 官方文档：https://protobuf.dev/
- 需按环境核实：`curl` 是否支持 WebSocket 握手观察，取决于版本和编译选项；`example.com` 是否返回 101 取决于服务端配置；裸 TCP 实验中一次 `recv` 收到多少数据取决于发送时机、缓冲区和系统调度；RPC 框架的具体服务发现和连接管理实现因框架而异。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>