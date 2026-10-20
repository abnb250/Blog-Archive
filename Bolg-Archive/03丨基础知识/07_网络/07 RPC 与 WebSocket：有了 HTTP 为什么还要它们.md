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
HTTP 用得好好的，为什么公司内部微服务还用 RPC？网页游戏为什么不用 HTTP 轮询，而要用 WebSocket？  
这一章咱们用两个具体问题串起来：**HTTP 做内部调用够不够 → HTTP 不能让服务器主动推怎么办。**  
先别急着背“RPC 是远程过程调用、WebSocket 是全双工”这些定义，先看两个场景：你在微服务里调一个用户服务，用 HTTP 传 JSON 行不行？行，但每次请求都带一堆重复头部、序列化也偏重，内部服务之间是不是有点浪费？再比如网页游戏里服务器要实时告诉你“对手放技能了”，HTTP 是客户端不请求服务器就不回，你总不能每秒发几十个请求问“有消息吗”吧？这两个场景，正好引出 RPC 和 WebSocket。

<br>

<br>

## 知识点 1：纯裸 TCP 的问题与 RPC
***
### 问题
TCP 能收发数据，为什么不能直接用？我直接 `send` 一段 JSON，对面 `recv` 不就行了吗？RPC 到底是什么，它和 HTTP 是什么关系？

<br>

### 方案
TCP 是**字节流**，没有消息边界。你发“夏洛”和“特烦恼”，对面可能一次收成“夏洛特烦恼”，也可能分两次收；直接裸用 TCP，应用层得自己定义“一条消息到哪结束”。常见做法是**消息头 + 消息体**：头里放长度，体里放内容，接收方先读头、再按长度读体。这就是应用层协议要解决的事。HTTP 和各类 RPC 都是基于传输层之上的应用层协议。  
**RPC 本身不是某一个具体协议，而是一种“像调本地方法一样调远端方法”的调用方式**；gRPC、Thrift、Dubbo 才是具体实现。RPC 不一定非基于 TCP，也可以基于 UDP 或 HTTP。  
RPC 和 HTTP 的常见区别可以这样看：  
- **服务发现**：HTTP 常用 DNS；RPC 常用 Consul、Etcd、Redis 等注册中心。  
- **连接管理**：HTTP/1.1 常靠 Keep-Alive 复用连接；RPC 常用连接池，减少频繁建连。  
- **传输内容**：HTTP/1.1 头部冗余、常用 JSON；RPC 常用 Protobuf 等更紧凑的序列化，性能更好。  
但注意：**HTTP/2 性能可能比很多 RPC 还好**，gRPC 底层用的就是 HTTP/2。所以别把 RPC 和 HTTP 对立起来，它们经常互相借用。

<br>

### 最小示例
概念上，RPC 想让你写起来像本地调用：

```text
本地调用：res = localFunc(req)
远程调用：res = remoteFunc(req)
```

RPC 的目标就是让远程调用看起来像本地调用，把网络细节、序列化、连接管理屏蔽掉。  
这里给的是调用方式示意，不是完整代码；真正的 RPC 框架会在背后做序列化、网络传输、反序列化、异常处理。

<br>

### 一句话本质
**RPC 是“远程调用方式”，HTTP 是“通用超文本协议”；内部微服务追求紧凑高效常用 RPC，对外通用常 HTTP。**

<br>

### 易错点
坑一：**RPC 不是某一个协议**，gRPC、Thrift 才是具体实现。  
坑二：**RPC 不一定基于 TCP**，也可以基于 UDP 或 HTTP。  
坑三：**HTTP/2 性能可能比很多 RPC 好**，gRPC 底层就是 HTTP/2。  
坑四：**裸 TCP 不是不能用**，而是你得自己定消息边界和协议；RPC 和 HTTP 都是在帮你做这件事。

<br>

### 自测
为什么公司内部微服务常用 RPC，而对外接口常用 HTTP？

<br>

<br>

## 知识点 2：WebSocket 解决服务器主动推送
***
### 问题
HTTP 是请求-响应模式，客户端不问，服务器就不能主动发消息。网页游戏要实时推送，怎么办？定时轮询、长轮询、WebSocket 到底差在哪？

<br>

### 方案
简单场景可以用**定时轮询**：客户端每隔几秒问一次“有消息吗”。缺点是延迟和无效请求。  
**长轮询**：客户端发请求，服务器先不急着回，等有消息或超时再响应；客户端收到后立刻再发下一个请求。比定时轮询少一些空请求，但本质仍是客户端驱动。  
复杂频繁交互场景用 **WebSocket**：TCP 本身是全双工的，WebSocket 基于 TCP 提供全双工通信。建立时先借 HTTP 升级：请求头带 `Connection: Upgrade`、`Upgrade: WebSocket`、`Sec-WebSocket-Key`；服务器返回 **101 Switching Protocols** 和 `Sec-WebSocket-Accept`；升级完成后就不再是 HTTP，而用 WebSocket 帧通信。  
帧里有 opcode（1 文本、2 二进制、8 关闭等）和 payload 长度字段，接收方按帧解析。  
注意：HTTP/1.1 严格说是**请求-响应模式**，服务器不能主动发起消息；口语里说“半双工”是为了强调“不能同时双向主动发”，但更准确的说法是“通信由客户端请求驱动”。

<br>

### 最小示例
客户端发起升级请求：

```http
GET /chat HTTP/1.1
Host: example.com
Connection: Upgrade
Upgrade: WebSocket
Sec-WebSocket-Key: T2a6wZlAwhgQNqruZ2YUyg==
Sec-WebSocket-Version: 13
```

服务器若支持，返回：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: WebSocket
Connection: Upgrade
Sec-WebSocket-Accept: iBJKv/ALIW2DobfoA4dmr3JHBCY=
```

之后双方用 WebSocket 帧通信。`Sec-WebSocket-Accept` 是拿 `Sec-WebSocket-Key` 加上固定 GUID 做 SHA-1 再 Base64 得到的，用来确认服务器真的理解 WebSocket 握手。

<br>

### 一句话本质
**HTTP 是“你不问我不答”，WebSocket 是“握手后双方都能主动说”。**

<br>

### 易错点
坑一：**WebSocket 不是直接用裸 socket 编程**，它是一套应用层协议；名字里的 socket 不是系统 socket 的意思。  
坑二：**“WebSocket 基于 HTTP 的新协议”不准确**，它只在握手时用 HTTP，升级后与 HTTP 无关。  
坑三：**长轮询不是 WebSocket**，长轮询仍是客户端发请求、服务器等一段时间再响应；WebSocket 升级后是真正的双向帧通信。  
坑四：**101 之后就不是 HTTP 了**，别再按 HTTP 报文格式去解析后面的数据。

<br>

### 自测
长轮询和 WebSocket 都能实现“服务器推送”，网页游戏更该选哪个？为什么？

<br>

<br>

## 完整可复现示例：用 C++ 发送 WebSocket 握手请求
***
下面这个 C++ 程序用 POSIX socket 连接 `example.com:80`，发送一个 WebSocket 升级请求，并读取服务器响应。  
注意：`example.com` 不一定支持 WebSocket，所以它很可能返回 `200` 或其他状态，而不是 `101`；这个示例的重点是验证“WebSocket 握手首先是一个带特殊 Header 的 HTTP 请求”。

```cpp
// websocket_handshake.cpp
// 编译：g++ -std=c++17 -O2 websocket_handshake.cpp -o websocket_handshake
// 运行：./websocket_handshake
// 环境：Linux/POSIX；Windows 需要改用 Winsock

#include <arpa/inet.h>
#include <netdb.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
#include <iostream>
#include <string>

int main() {
    const char* host = "example.com";
    const char* port = "80";
    const char* path = "/chat";
    const char* ws_key = "dGhlIHNhbXBsZSBub25jZQ==";

    // 1. 解析主机名
    addrinfo hints{};
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;

    addrinfo* res = nullptr;
    int rc = getaddrinfo(host, port, &hints, &res);
    if (rc != 0) {
        std::cerr << "getaddrinfo failed: " << gai_strerror(rc) << "\n";
        return 1;
    }

    // 2. 建立 TCP 连接
    int sockfd = -1;
    for (addrinfo* p = res; p != nullptr; p = p->ai_next) {
        sockfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
        if (sockfd < 0) continue;
        if (connect(sockfd, p->ai_addr, p->ai_addrlen) == 0) break;
        close(sockfd);
        sockfd = -1;
    }
    freeaddrinfo(res);

    if (sockfd < 0) {
        std::cerr << "connect failed\n";
        return 1;
    }

    // 3. 构造 WebSocket 升级请求
    std::string request;
    request += "GET ";
    request += path;
    request += " HTTP/1.1\r\n";
    request += "Host: ";
    request += host;
    request += "\r\n";
    request += "Upgrade: websocket\r\n";
    request += "Connection: Upgrade\r\n";
    request += "Sec-WebSocket-Key: ";
    request += ws_key;
    request += "\r\n";
    request += "Sec-WebSocket-Version: 13\r\n";
    request += "\r\n";

    std::cout << "=== 发送的握手请求 ===\n" << request << "\n";

    // 4. 发送请求
    ssize_t sent = send(sockfd, request.data(), request.size(), 0);
    if (sent < 0) {
        std::cerr << "send failed\n";
        close(sockfd);
        return 1;
    }

    // 5. 读取响应前几行
    char buf[4096];
    ssize_t n = recv(sockfd, buf, sizeof(buf) - 1, 0);
    if (n > 0) {
        buf[n] = '\0';
        std::cout << "=== 服务器响应（前几行） ===\n";
        std::string response(buf);
        // 只打印前 10 行，避免输出过长
        int lines = 0;
        size_t pos = 0;
        while (pos < response.size() && lines < 10) {
            size_t end = response.find('\n', pos);
            if (end == std::string::npos) end = response.size();
            std::cout << response.substr(pos, end - pos + 1);
            pos = end + 1;
            ++lines;
        }
    } else {
        std::cout << "没有读到响应，或连接已关闭\n";
    }

    close(sockfd);
    return 0;
}
```

输出/现象/结论可能类似：

```text
=== 发送的握手请求 ===
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13


=== 服务器响应（前几行） ===
HTTP/1.1 200 OK
Content-Type: text/html
...
```

验证结果：WebSocket 握手首先是一个带 `Upgrade: websocket` 等特殊 Header 的 HTTP 请求；服务器若支持就返回 `101 Switching Protocols`，之后升级为 WebSocket 帧通信。`example.com` 不支持 WebSocket，所以这里看到的是普通 HTTP 响应；换成支持 WebSocket 的服务器，才会看到 101。  
注意：不同系统、不同网络环境下连接可能失败；如果 `example.com` 的 80 端口被重定向或限制，输出会不同。这段代码的重点是握手请求的结构，不是一定要拿到 101。

<br>

<br>

## 自测三问
***
1. RPC 和 HTTP 的区别主要在哪几个方面？为什么说 gRPC 底层用 HTTP/2 并不矛盾？  
2. WebSocket 握手用了 HTTP 的什么机制？升级后还用 HTTP 吗？  
3. 长轮询和 WebSocket 各自适合什么场景？网页游戏为什么更倾向 WebSocket？

<br>

<br>

## 一句话总结全章
***
**RPC 是“像调本地方法一样调远程”的调用方式，追求内部高效；WebSocket 借 HTTP 握手升级，之后提供全双工通信，解决服务器主动推送问题。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>