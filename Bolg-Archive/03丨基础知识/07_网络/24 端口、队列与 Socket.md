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
你重启一个 Web 服务，内核报 `Address already in use`。

你可能会想：**端口是不是被别的进程占了？**

`ss -ltnp` 一看，没有别的进程监听这个端口。那为什么绑不上？很可能是上次的进程刚关闭，端口还在 `TIME_WAIT`。

再想一个场景：你压测服务时，客户端 `connect` 成功了，但服务端 `accept` 很慢，连接堆在哪里？**没有 `accept`，连接还能建立吗？**

这一章咱们用一条问题链往前走：

1. TCP 和 UDP 能同时绑同一个端口吗？多个 TCP 服务呢？
2. 重启服务报 `Address already in use`，到底是端口被占，还是 TIME_WAIT 没走？
3. `accept` 参与三次握手吗？没有 `accept` 连接能建立吗？
4. 半连接队列和全连接队列满了，分别会怎样？

<br>

<br>

## 优先级说明
***
这篇是端口与连接队列的核心，面试常问，项目里排查 `bind` 失败、`connect` 超时、`accept` 慢也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- TCP 和 UDP 可以同时绑定同一端口，因为 IP 头协议号先区分
- 多个 TCP 服务完全相同的 IP+端口不能同时监听，不同 IP 可以共存
- `SO_REUSEADDR` 主要解决 TIME_WAIT 绑定和部分地址重叠，不是万能
- `SO_REUSEPORT` 允许多个套接字绑同一端口做负载分担，和 `SO_REUSEADDR` 不是一回事
- `listen` 后内核创建半连接队列和全连接队列
- `accept` 不参与三次握手，只从全连接队列取连接
- 没有 `accept` 也能完成握手，连接会在全连接队列里等
- 全连接队列满默认丢 ACK，`tcp_abort_on_overflow=1` 才回 RST
- `ss -lnt` 在 LISTEN 状态下，`Recv-Q` 是当前全连接队列，`Send-Q` 是最大长度

**了解即可（知道大概，不展开）：**
- `tcp_syncookies` 的具体 cookie 计算方式
- 半连接队列在 Linux 中的哈希表实现
- TCP 自连接、同时打开的特殊情况
- `somaxconn` 和 `backlog` 的精确取小逻辑

**主线**：端口是门牌号，连接是四元组 → TCP/UDP 同端口不冲突 → `SO_REUSEADDR` 解决 TIME_WAIT 绑定 → `listen` 建两个队列 → 握手搬队列 → `accept` 只取货 → 队列满会丢包或回 RST。

<br>

<br>

## 知识点 1：端口绑定与复用——门牌号相同不一定冲突，看协议和四元组
***
### 问题

端口到底怎么分配和复用？TCP 和 UDP 为什么能同时绑同一个端口？多个 TCP 服务为什么有时能绑同一端口、有时报 `Address already in use`？`SO_REUSEADDR` 到底解决了什么？

**你可能会想**：端口被占了就是被占了，还能复用？

能，但要看情况。**TCP 和 UDP 是两个独立模块，端口空间分开；同一个 TCP 端口，多个服务能不能绑，取决于 IP 和端口组合是否完全相同。**

<br>

### 核心概念与方案

先把端口换成你熟悉的场景：**端口是门牌号，协议是小区名，连接是快递单号。**

- **TCP 和 UDP 内核模块独立**，可以同时绑定同一端口，因为 IP 头协议号先区分 TCP 还是 UDP。
- **多个 TCP 服务同时 `bind` 相同 IP+端口** 会 `Address already in use`；如果绑定 IP 不同，则可以共存。
- **`SO_REUSEADDR`** 有两个常见作用：让新进程绑定与 TIME_WAIT 冲突的 IP+端口；允许 `0.0.0.0:8888` 与 `192.168.1.100:8888` 这类不完全相同的绑定共存。
- **`SO_REUSEPORT`** 允许多个套接字绑同一端口做负载分担，和 `SO_REUSEADDR` 不是一回事。

客户端连接由四元组唯一确定，所以同一客户端端口可以连接不同服务器；连接同一服务器时，TIME_WAIT 过多可能耗尽端口，`tcp_tw_reuse` 可复用超过 1 秒的 TIME_WAIT 端口，但只对 `connect` 发起方有效。

<br>

### 最小实验：只看 TCP 和 UDP 能不能同时绑 18080

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 且支持 C++17。下面只在本机回环地址上运行，不对外网开放。

保存为 `bind_both.cpp`：

````cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <iostream>

int main() {
    const int port = 18080;

    int tcp = socket(AF_INET, SOCK_STREAM, 0);
    if (tcp < 0) {
        std::cerr << "tcp socket failed\n";
        return 1;
    }

    int opt = 1;
    setsockopt(tcp, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(port);

    if (bind(tcp, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        std::cerr << "tcp bind failed\n";
        close(tcp);
        return 1;
    }
    listen(tcp, 5);

    int udp = socket(AF_INET, SOCK_DGRAM, 0);
    if (udp < 0) {
        std::cerr << "udp socket failed\n";
        close(tcp);
        return 1;
    }

    if (bind(udp, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        std::cerr << "udp bind failed\n";
        close(tcp);
        close(udp);
        return 1;
    }

    std::cout << "TCP and UDP both bound to 0.0.0.0:" << port << "\n";

    close(udp);
    close(tcp);
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 bind_both.cpp -o bind_both
./bind_both
````

**这次只看一件事**：程序有没有打印 `TCP and UDP both bound to 0.0.0.0:18080`。

预期输出：

````text
TCP and UDP both bound to 0.0.0.0:18080
````

打印成功，说明 TCP 和 UDP 可以同时绑同一端口。**如果报 `udp bind failed`，先检查 18080 是否被别的 UDP 程序占用，换个端口再试。**

**结论**：TCP 和 UDP 端口空间分开，同端口不冲突，因为 IP 头协议号先区分。

<br>

### 易错点与边界

坑一：**端口不是进程号**，端口是运输层标识，进程号是操作系统内部编号。

坑二：**`SO_REUSEADDR` 不是万能**，它主要解决 TIME_WAIT 绑定和部分地址重叠，不能让两个完全相同的 IP+端口同时监听。

坑三：**`tcp_tw_reuse` 只对 `connect` 发起方有效**，服务端开它不会复用服务端主动关闭的 TIME_WAIT。

坑四：**`SO_REUSEPORT` 和 `SO_REUSEADDR` 不是一回事**，前者允许多个套接字绑同一端口做负载分担，后者主要解决重用问题。

<br>

### 一句话本质

**端口是进程门牌号，连接是四元组；TCP/UDP 端口空间分开，同端口不冲突；`SO_REUSEADDR` 解决 TIME_WAIT 绑定，`SO_REUSEPORT` 做负载分担，两者不是一回事。**

<br>

### 自测

客户端 TIME_WAIT 过多，为什么不一定影响连其他服务器？`SO_REUSEADDR` 和 `SO_REUSEPORT` 分别解决什么问题？

<details><summary>答案</summary>客户端连接由四元组唯一确定，连不同服务器时目的 IP 或目的端口不同，四元组不同，即使本地端口相同也不冲突。TIME_WAIT 过多主要影响连同一服务器时本地端口的复用。`SO_REUSEADDR` 主要解决 TIME_WAIT 绑定和部分地址重叠，让新进程能绑与 TIME_WAIT 冲突的端口；`SO_REUSEPORT` 允许多个套接字绑同一端口做负载分担，和 `SO_REUSEADDR` 不是一回事。</details>

<br>

<br>

## 知识点 2：listen、accept 与半连接/全连接队列——握手搬队列，accept 只取货
***
### 问题

`accept` 参与三次握手吗？没有 `accept` 连接能建立吗？没有 `listen` 会怎样？半连接队列和全连接队列满了分别怎么处理？

**你可能会想**：`accept` 应该就是“建立连接”的那一步吧？

不是。**`accept` 不参与三次握手**，三次握手在底层完成，`accept` 只是从全连接队列取一个已经建好的连接。

<br>

### 核心概念与方案

先把队列换成你熟悉的场景：**半连接队列是“正在填表的人”，全连接队列是“填完表等叫号的人”，`accept` 是柜台叫号。**

服务端 `listen` 后内核创建**半连接队列（SYN 队列）**和**全连接队列（accept 队列）**：

- 收到 SYN 放入半连接队列。
- 收到第三次 ACK 后移到全连接队列，等待 `accept()` 取走。

**`accept` 不参与三次握手**，只从全连接队列取连接。

全连接队列长度约 `min(backlog, somaxconn)`；满了默认丢 ACK，`tcp_abort_on_overflow=1` 回 RST。半连接队列满可丢 SYN，开 `tcp_syncookies=1` 可绕过。

服务端只 `bind` 未 `listen`，客户端 SYN 到达找不到监听 socket，通常回 RST。但没有 `listen` 也能建立连接的特殊情况：TCP 自连接、TCP 同时打开，靠全局 hash 表存放连接信息。

<br>

### 最小实验：只看不 accept 时 Recv-Q 是不是增长

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 和 `nc`。下面只在本机回环地址上运行。

保存为 `listen_no_accept.cpp`：

````cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <iostream>

int main() {
    const int port = 18081;

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
````

编译运行：

````bash
g++ -std=c++17 -O2 listen_no_accept.cpp -o listen_no_accept
./listen_no_accept
````

终端 1 保持服务端运行，先不要按 Enter。

终端 2 发起连接：

````bash
nc -vz 127.0.0.1 18081
````

终端 3 查看监听队列：

````bash
ss -lnt | grep 18081
````

**这次只看一件事**：`ss -lnt` 输出里 `Recv-Q` 是不是 1。

可能看到类似：

````text
LISTEN 1      1      0.0.0.0:18081     0.0.0.0:*
````

`Recv-Q=1` 表示当前全连接队列里有 1 个已完成握手的连接，等待 `accept` 取走；`Send-Q=1` 是 `listen(..., 1)` 设置的最大队列长度。

**关键观察点**：`nc -vz` 能连上，说明**没有 `accept` 也能完成三次握手**，连接只是堆在全连接队列里。

如果 `Recv-Q` 不是 1，先检查 `nc` 是否连上、端口是否一致。`nc -vz` 的参数在不同系统上可能不同，**需按环境核实**。

<br>

### 易错点与边界

坑一：**半连接队列不一定是简单链表**，Linux 中常是哈希表，排查时不能只看一个数字。

坑二：**只 `bind` 不 `listen` 通常回 RST**，别以为绑了端口就能收连接。

坑三：**没有 `accept` 也能完成握手**，连接会在全连接队列里等；队列满了才会丢 ACK 或回 RST。

坑四：**`backlog` 不是连接数上限**，它影响队列长度，实际还受 `somaxconn` 限制。

坑五：**`accept` 取走的连接已经完成三次握手**，不是 `accept` 触发握手。

<br>

### 一句话本质

**`listen` 建半连接队列和全连接队列，握手把连接从半连接队列搬到全连接队列，`accept` 只是从全连接队列取货；没有 `accept` 也能完成握手，队列满了才会丢包或回 RST。**

<br>

### 自测

没有 `accept`，TCP 连接能建立吗？没有 `listen` 呢？全连接队列满了默认怎么处理，开 `tcp_abort_on_overflow=1` 又怎么处理？

<details><summary>答案</summary>没有 `accept`，TCP 连接能建立，三次握手在底层完成，连接堆在全连接队列等 `accept` 取。没有 `listen`，客户端 SYN 到达找不到监听 socket，通常回 RST；TCP 自连接、同时打开等特殊情况除外。全连接队列满默认丢 ACK，客户端可能重试；开 `tcp_abort_on_overflow=1` 则回 RST，客户端更快失败。</details>

<br>

<br>

## 完整可复现示例：观察 TCP/UDP 同端口绑定和 listen 队列
***
你需要 Linux/macOS 或 WSL，已安装 `g++`、`nc`、`ss`。下面每个步骤只看一件事，不展开无关细节。

````bash
set -e

echo "=== 1. TCP 和 UDP 同时绑定 18080 ==="
g++ -std=c++17 -O2 bind_both.cpp -o bind_both
./bind_both

echo "=== 2. 启动不 accept 的服务端 ==="
g++ -std=c++17 -O2 listen_no_accept.cpp -o listen_no_accept
./listen_no_accept
````

终端 1 运行第 2 步后先不要按 Enter。终端 2 执行：

````bash
nc -vz 127.0.0.1 18081
ss -lnt | grep 18081
````

可能的输出/现象：

````text
=== 1. TCP 和 UDP 同时绑定 18080 ===
TCP and UDP both bound to 0.0.0.0:18080

=== 2. 监听队列 ===
LISTEN 1      1      0.0.0.0:18081     0.0.0.0:*
````

**关键观察点**：第 1 步看 TCP/UDP 是否都绑定成功；第 2 步看 `Recv-Q` 是否为 1。

验证结果：TCP/UDP 端口空间分开，同端口不冲突；没有 `accept` 也能完成握手，连接堆在全连接队列，`Recv-Q` 显示当前队列长度。

注意：`nc -vz` 的可用性和参数在不同系统上可能不同；`ss -lnt` 的 `Recv-Q`/`Send-Q` 在 LISTEN 状态下的含义可能因内核版本略有差异，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| `bind: Address already in use` | 端口是否被占、是否 TIME_WAIT | `ss -ltnp \| grep ':端口'`；`ss -tan \| grep TIME-WAIT` | 可能被别的进程占用，或旧连接 TIME_WAIT 未走；`SO_REUSEADDR` 可缓解 |
| TCP 和 UDP 同端口，一个能绑一个不能 | 是否绑错协议 | 分别用 `ss -ltnp` 和 `ss -lunp` 查看 | TCP 和 UDP 端口空间分开，要确认服务实际监听的是哪个 |
| `connect` 成功但 `accept` 慢 | 全连接队列是否堆积 | `ss -lnt` 看 `Recv-Q`；`netstat -s \| grep overflowed` | 连接已完成握手，堆在全连接队列；`accept` 慢或队列满都会导致堆积 |
| `connect` 超时，服务端像没收到 SYN | 半连接队列、防火墙 | `netstat -s \| grep "SYNs to LISTEN"` | 半连接队列满会丢 SYN；防火墙也可能拦 SYN |
| 没有 `listen` 也能连上 | 是否 TCP 自连接/同时打开 | 抓包看端口是否相同；查内核行为 | 特殊情况靠全局 hash 表存放连接信息，不是常规路径 |
| `SO_REUSEADDR` 设了还是绑不上 | 是否完全相同 IP+端口被占 | `ss -ltnp` 看谁在监听 | `SO_REUSEADDR` 不能让两个完全相同的 IP+端口同时监听 |
| `SO_REUSEPORT` 多个进程绑同一端口 | 是否真的想负载分担 | 查代码；`ss -ltnp` 看多个进程 | `SO_REUSEPORT` 允许多个套接字绑同一端口做负载分担，和 `SO_REUSEADDR` 不同 |

<br>

<br>

## 常见误区
***
- **以为 TCP 和 UDP 不能同端口**：端口空间按协议分开，同一端口号可以分别绑定 TCP 和 UDP。
- **以为 `bind: Address already in use` 一定是端口被占**：可能是 TIME_WAIT 未走，`SO_REUSEADDR` 可缓解。
- **以为 `SO_REUSEADDR` 万能**：它不能让两个完全相同的 IP+端口同时监听。
- **把 `SO_REUSEADDR` 和 `SO_REUSEPORT` 混了**：前者解决重用，后者做负载分担。
- **以为 `accept` 参与三次握手**：握手在底层完成，`accept` 只从全连接队列取连接。
- **以为没有 `accept` 连接就建立不了**：没有 `accept` 也能完成握手，连接堆在全连接队列。
- **以为 `backlog` 是连接数上限**：它影响队列长度，实际还受 `somaxconn` 限制。
- **以为全连接队列满一定回 RST**：默认丢 ACK，`tcp_abort_on_overflow=1` 才回 RST。

<br>

<br>

## 自测三问
***
1. TCP/UDP 能同时绑定同一端口吗？为什么？多个 TCP 服务在什么情况下能绑同一端口？
2. `accept` 在三次握手的哪一步发生？没有 `accept` 连接能建立吗？
3. 全连接队列满了默认怎么处理？`tcp_abort_on_overflow=1` 呢？`SO_REUSEADDR` 和 `SO_REUSEPORT` 又分别解决什么问题？

<details><summary>参考答案</summary>1. TCP/UDP 可以同时绑定同一端口，因为 IP 头协议号先区分 TCP 和 UDP，内核里两个模块各自按端口分发。多个 TCP 服务只有在绑定 IP 不同时才能共存；完全相同 IP+端口不能同时监听，除非用 `SO_REUSEPORT` 等特殊选项。2. `accept` 不参与三次握手，握手在底层完成，`accept` 只是从全连接队列取一个已完成握手的连接。没有 `accept` 连接也能建立，连接堆在全连接队列等 `accept` 取。3. 全连接队列满默认丢 ACK，客户端可能重试；`tcp_abort_on_overflow=1` 则回 RST，客户端更快失败。`SO_REUSEADDR` 主要解决 TIME_WAIT 绑定和部分地址重叠；`SO_REUSEPORT` 允许多个套接字绑同一端口做负载分担。</details>

<br>

<br>

## 总结
***
**端口是进程门牌号，连接靠四元组区分；TCP/UDP 端口空间分开，同端口不冲突；`listen` 创建半连接队列和全连接队列，握手负责搬队列，`accept` 只取货；队列满会丢 SYN/ACK，`SO_REUSEADDR` 解决 TIME_WAIT 绑定，`SO_REUSEPORT` 做负载分担。**

<br>

<br>

## 参考资料
***
- RFC 9293：Transmission Control Protocol
- RFC 768：User Datagram Protocol
- RFC 6056：Recommendations for Transport-Protocol Port Randomization
- `man 7 socket`、`man 2 bind`、`man 2 listen`、`man 2 accept`
- 需按环境核实：`SO_REUSEADDR`/`SO_REUSEPORT` 在不同系统上的行为差异；`ss -lnt` 的 `Recv-Q`/`Send-Q` 在 LISTEN 状态下的含义可能因内核版本略有不同；`somaxconn` 和 `backlog` 的默认值因系统而异；TCP 自连接、同时打开等特殊情况的行为需按实现核实。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>