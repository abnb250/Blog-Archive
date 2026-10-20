# 24 端口、队列与 Socket：TCP/UDP 同端口、bind/listen/accept、半连接/全连接队列
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
**TCP 和 UDP 能同端口吗？多个 TCP 服务能 bind 同一端口吗？重启服务为什么 Address already in use？没有 accept 能建立连接吗？没有 listen 会怎样？**  
先别急着背 `SO_REUSEADDR`、`backlog`、`somaxconn` 这些词，先看几个具体场景：你重启一个 Web 服务，内核报 `Address already in use`，是不是端口被别的进程占了？你压测时客户端 `connect` 成功，但服务端 `accept` 很慢，连接堆在哪里？你把服务改成只 `bind` 不 `listen`，客户端还能连上吗？这一章咱们就按“端口怎么绑、连接怎么排、accept 取什么”这条线走。

<br>

<br>

## 知识点 1：端口绑定与复用
***
### 问题
端口到底怎么分配和复用？TCP 和 UDP 为什么能同时绑同一个端口？多个 TCP 服务为什么有时能绑同一端口、有时报 `Address already in use`？`SO_REUSEADDR` 到底解决了什么？

<br>

### 方案
TCP/UDP 内核模块独立，**可以同时绑定同一端口**，因为 IP 头协议号先区分 TCP 还是 UDP。  
多个 TCP 服务同时 `bind` 相同 IP+端口会 `Address already in use`；如果绑定 IP 不同，则可以共存。  
`SO_REUSEADDR` 有两个常见作用：让新进程绑定与 TIME_WAIT 冲突的 IP+端口；允许 `0.0.0.0:8888` 与 `192.168.1.100:8888` 这类不完全相同的绑定共存。  
客户端连接由四元组唯一确定，所以同一客户端端口可以连接不同服务器；连接同一服务器时，TIME_WAIT 过多可能耗尽端口，`tcp_tw_reuse` 可复用超过 1 秒的 TIME_WAIT 端口，但只对 `connect` 发起方有效。

<br>

### 最小示例
```text
客户端 64992 连服务端 A，也可用 64992 连服务端 B，因为目的 IP 不同，四元组不同。
```

结论：端口是进程门牌号，连接是四元组；同端口不冲突，看协议和四元组。

<br>

### 一句话本质
**端口是进程门牌号，连接是四元组；同端口不冲突，看协议和四元组。**

<br>

### 易错点
坑一：**端口不是进程号**，端口是运输层标识，进程号是操作系统内部编号。  
坑二：**`SO_REUSEADDR` 不是万能**，它主要解决 TIME_WAIT 绑定和部分地址重叠，不能让两个完全相同的 IP+端口同时监听。  
坑三：**`tcp_tw_reuse` 只对 `connect` 发起方有效**，服务端开它不会复用服务端主动关闭的 TIME_WAIT。  
坑四：**`SO_REUSEPORT` 和 `SO_REUSEADDR` 不是一回事**，前者允许多个套接字绑同一端口做负载分担，后者主要解决重用问题。

<br>

### 自测
客户端 TIME_WAIT 过多，为什么不一定影响连其他服务器？`SO_REUSEADDR` 和 `SO_REUSEPORT` 分别解决什么问题？

<br>

<br>

## 知识点 2：listen、accept 与半连接/全连接队列
***
### 问题
`accept` 参与三次握手吗？没有 `accept` 连接能建立吗？没有 `listen` 会怎样？半连接队列和全连接队列满了分别怎么处理？

<br>

### 方案
服务端 `listen` 后内核创建**半连接队列（SYN 队列）**和**全连接队列（accept 队列）**。收到 SYN 放入半连接队列；收到第三次 ACK 后移到全连接队列，等待 `accept()` 取走。**`accept` 不参与三次握手**，只从全连接队列取连接。  
全连接队列长度约 `min(backlog, somaxconn)`；满了默认丢 ACK，`tcp_abort_on_overflow=1` 回 RST。半连接队列满可丢 SYN，开 `tcp_syncookies=1` 可绕过。  
服务端只 `bind` 未 `listen`，客户端 SYN 到达找不到监听 socket，通常回 RST。但没有 `listen` 也能建立连接的特殊情况：TCP 自连接、TCP 同时打开，靠全局 hash 表存放连接信息。

<br>

### 最小示例
```bash
ss -lnt
# LISTEN 状态：Recv-Q 当前全连接队列，Send-Q 最大长度
netstat -s 2>/dev/null | grep overflowed
```

输出可能类似：

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 0      128    0.0.0.0:22         0.0.0.0:*
```

结论：`Recv-Q` 是当前全连接队列，`Send-Q` 是最大长度；溢出统计持续增长说明队列可能不够。

<br>

### 一句话本质
**listen 建队列，握手把连接从半连接队列搬到全连接队列，accept 只是从全连接队列取货。**

<br>

### 易错点
坑一：**半连接队列不一定是简单链表**，Linux 中常是哈希表，排查时不能只看一个数字。  
坑二：**只 `bind` 不 `listen` 通常回 RST**，别以为绑了端口就能收连接。  
坑三：**没有 `accept` 也能完成握手**，连接会在全连接队列里等；队列满了才会丢 ACK 或回 RST。  
坑四：**`backlog` 不是连接数上限**，它影响队列长度，实际还受 `somaxconn` 限制。  
坑五：**`accept` 取走的连接已经完成三次握手**，不是 `accept` 触发握手。

<br>

### 自测
没有 `accept`，TCP 连接能建立吗？没有 `listen` 呢？全连接队列满了默认怎么处理，开 `tcp_abort_on_overflow=1` 又怎么处理？

<br>

<br>

## 完整可复现示例：用 C++ 观察 listen/accept 与队列状态
***
这个示例分两部分：先用 C++ 写一个最小 TCP 服务端，`bind` + `listen` 后**故意不立即 `accept`**，让连接堆在全连接队列；再用 `ss` 和 `netstat` 观察队列状态。  
注意：`ss -lnt` 的 `Recv-Q` 在 LISTEN 状态下表示当前全连接队列长度，`Send-Q` 表示最大长度；不同内核版本行为可能略有差异。

C++ 服务端：

```cpp
// listen_no_accept.cpp
// 编译：g++ -std=c++17 -O2 listen_no_accept.cpp -o listen_no_accept
// 运行：./listen_no_accept
// 环境：Linux/POSIX

#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
#include <iostream>

int main() {
    const int port = 18080;

    int serverSocket = socket(AF_INET, SOCK_STREAM, 0);
    if (serverSocket < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    int opt = 1;
    setsockopt(serverSocket, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(port);

    if (bind(serverSocket, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        std::cerr << "bind failed\n";
        close(serverSocket);
        return 1;
    }

    // backlog 设为 1，方便观察全连接队列
    if (listen(serverSocket, 1) < 0) {
        std::cerr << "listen failed\n";
        close(serverSocket);
        return 1;
    }

    std::cout << "listening on " << port << ", but not accepting yet...\n";
    std::cout << "press Enter to accept one connection...\n";
    std::cin.get();

    sockaddr_in clientAddr{};
    socklen_t clientLen = sizeof(clientAddr);
    int conn = accept(serverSocket, reinterpret_cast<sockaddr*>(&clientAddr), &clientLen);
    if (conn >= 0) {
        std::cout << "accepted one connection\n";
        close(conn);
    }

    close(serverSocket);
    return 0;
}
```

运行步骤：

```bash
# 终端 1：启动服务端，先不要按 Enter
./listen_no_accept

# 终端 2：用 nc 或 curl 发起连接（nc 更直接）
nc -vz 127.0.0.1 18080

# 终端 3：观察监听队列
ss -lnt | grep 18080
netstat -s 2>/dev/null | grep -i -E "overflowed|SYNs to LISTEN" || true
```

输出/现象/结论可能类似：

```text
# ss -lnt 输出
State  Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 1      1      0.0.0.0:18080     0.0.0.0:*
```

验证结果：`Recv-Q=1` 表示当前全连接队列里有 1 个已完成握手的连接，等待 `accept` 取走；`Send-Q=1` 是 `listen(..., 1)` 设置的最大队列长度。客户端 `nc -vz` 能连上，说明**没有 `accept` 也能完成三次握手**，连接只是堆在全连接队列里。  
注意：`nc -vz` 的可用性和参数在不同系统上可能不同；`ss -lnt` 的 `Recv-Q`/`Send-Q` 在 LISTEN 状态下的含义也可能因内核版本略有差异。如果连续发起多个连接超过队列上限，可能看到溢出统计增长，客户端可能超时或收到 RST。

<br>

<br>

## 自测三问
***
1. TCP/UDP 能同时绑定同一端口吗？为什么？多个 TCP 服务在什么情况下能绑同一端口？  
2. `accept` 在三次握手的哪一步发生？没有 `accept` 连接能建立吗？  
3. 全连接队列满了默认怎么处理？`tcp_abort_on_overflow=1` 呢？`SO_REUSEADDR` 和 `SO_REUSEPORT` 又分别解决什么问题？

<br>

<br>

## 一句话总结全章
***
**端口是进程门牌号，连接靠四元组区分；listen 创建半连接/全连接队列，握手负责搬队列，accept 只取货；队列满会丢 SYN/ACK，syncookies 和 backlog/somaxconn 是关键调节点。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>