# 37 回环地址、localhost、0.0.0.0 与本机地址：断网为什么还能 ping 通 127.0.0.1
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
拔掉网线，`ping 127.0.0.1` 还能通。**为什么？127.0.0.1、localhost、0.0.0.0、本机 IP 有什么区别？服务器 listen 0.0.0.0 是什么意思？**  
先别急着背“回环地址、监听地址”这些词，先看几个具体场景：你把网线拔了，`ping 127.0.0.1` 还能通，是不是说明网卡没坏、网络也没断？你写了个服务监听 `0.0.0.0:8080`，用 `127.0.0.1:8080` 能访问，用 `192.168.1.10:8080` 也能访问，为什么？客户端 `connect` 能不能填 `0.0.0.0`？这一章咱们就按“回环怎么绕、localhost 怎么解析、0.0.0.0 到底绑什么”这条线走。

<br>

<br>

## 知识点 1：127.0.0.1 是回环地址
***
### 问题
断网了为什么还能 ping 通 127.0.0.1？它和私有 IP 是一回事吗？ping 通回环能说明外网正常吗？

<br>

### 方案
`127.0.0.1` 是 IPv4 回环地址，`127.0.0.0/8` 都属于回环范围。数据发到回环地址不会出真网卡，而是在本机协议栈里绕一圈回来：走本地回环接口 `lo`（或 `lo0`），进入链表/队列后触发软中断，由内核线程处理，再层层往上交给应用。  
所以断网也能通；但这不代表能上网。IPv6 的回环地址是 `::1`。  
注意：回环地址不是私有 IP；私有 IP 是 `10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`，回环是单独一类特殊地址。

<br>

### 最小示例
```bash
ping -c 3 127.0.0.1
ping6 -c 3 ::1
```

断网时通常仍能通。  
结论：回环地址走的是本机协议栈，不经过物理网卡，所以网线拔了也能通；但外网通不通，得看真实网卡和路由。

<br>

### 一句话本质
**127.0.0.1 是“本机自己”，数据不出网卡，断网也能在本机协议栈里打个转回来。**

<br>

### 易错点
坑一：**回环地址不是私有 IP**，两者用途和地址段都不同。  
坑二：**断网 ping 通回环不代表外网通**，只能说明本机协议栈和回环接口正常。  
坑三：**`127.0.0.1` 和本机 IP 在访问本机服务时可能都走 `lo`**，具体看路由；不是只有回环地址才走回环接口。  
坑四：**IPv6 回环是 `::1`**，不是 `127.0.0.1`；双栈环境下两者要分开看。

<br>

### 自测
ping 通 127.0.0.1 能说明你的外网连接正常吗？回环地址和私有 IP 有什么区别？

<br>

<br>

## 知识点 2：localhost、0.0.0.0 与本机地址
***
### 问题
localhost 是 IP 吗？它一定解析到 127.0.0.1 吗？0.0.0.0 为什么不能 ping 通，但服务器可以 listen？listen 0.0.0.0 和 listen 127.0.0.1 有什么区别？客户端 `connect` 能不能填 0.0.0.0？

<br>

### 方案
`localhost` 是**域名**，默认解析到 `127.0.0.1`，可在 `/etc/hosts` 修改。它不是 IP，别把它当成地址本身。  
`0.0.0.0` 在 IPv4 中表示“本机所有 IPv4 地址”：  
- 作为**监听地址**时，表示绑所有网卡，`listen 0.0.0.0:80` 表示本机所有 IPv4 地址的 80 端口都收；  
- 作为**目标地址**通常无效，`ping 0.0.0.0` 会失败；客户端 `connect` 不能用 `0.0.0.0`，必须指定具体服务器 IP。  
本机 IP 是网卡上的实际地址，比如 `192.168.1.10`；访问本机 IP 和回环地址，数据通常也走 `lo`，不出网络。  
listen `127.0.0.1:8080` 只绑回环，外部机器连不进来；listen `0.0.0.0:8080` 绑所有网卡，局域网内其他机器可能连进来，注意防火墙。

<br>

### 最小示例
```text
服务 listen 0.0.0.0:80
→ 用 127.0.0.1:80 能访问
→ 用 192.168.1.10:80 也能访问
→ 外部机器可能也能访问（取决于防火墙）

服务 listen 127.0.0.1:80
→ 用 127.0.0.1:80 能访问
→ 外部机器连不进来
```

结论：localhost 是默认指向 127.0.0.1 的域名；0.0.0.0 监听表示“所有本机地址”，但不能当连接目标。

<br>

### 一句话本质
**localhost 是默认指向 127.0.0.1 的域名；0.0.0.0 监听表示“所有本机地址”，但不能当连接目标。**

<br>

### 易错点
坑一：**`0.0.0.0` 不是“任意远程地址”**，它表示本机所有 IPv4 地址；作为目标地址通常无效。  
坑二：**listen `0.0.0.0` 可能暴露到所有网卡**，注意防火墙和访问控制，别把只该本机访问的服务开到公网。  
坑三：**localhost 不一定是 127.0.0.1**，要看 `/etc/hosts`；有些系统可能把它解析到 `::1` 或其他地址。  
坑四：**`0.0.0.0` 和 `127.0.0.1` 监听范围不同**：前者所有网卡，后者只回环；排查“为什么外部连不进来”时先看监听地址。  
坑五：**客户端 `connect` 不能用 `0.0.0.0`**，必须指定具体服务器 IP；`0.0.0.0` 是本地监听语义，不是远程目标。

<br>

### 自测
服务器 listen `0.0.0.0:8080`，用本机 IP 和 127.0.0.1 都能访问吗？如果改成 listen `127.0.0.1:8080`，外部机器还能连吗？

<br>

<br>

## 完整可复现示例：看回环接口、监听地址并验证访问范围
***
这个示例先看回环接口和本机监听地址，再用 C++ 写一个最小 TCP 服务端分别监听 `127.0.0.1` 和 `0.0.0.0`，对比外部可达性。  
注意：需要 Linux/POSIX 环境；测试外部访问需要另一台机器或从本机用非回环 IP 访问。

先看回环接口和监听：

```bash
set -e

echo "=== 1. 回环接口 ==="
if command -v ip >/dev/null 2>&1; then
  ip addr show lo
else
  ifconfig lo0 2>/dev/null || true
fi

echo "=== 2. 监听 0.0.0.0 的服务 ==="
if command -v ss >/dev/null 2>&1; then
  ss -lnt | grep '0.0.0.0' || true
else
  netstat -lnt 2>/dev/null | grep '0.0.0.0' || true
fi

echo "=== 3. ping 回环 ==="
ping -c 2 127.0.0.1 || true
```

输出/现象/结论可能类似：

```text
=== 1. 回环接口 ===
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    inet 127.0.0.1/8 scope host lo

=== 2. 监听 0.0.0.0 的服务 ===
LISTEN 0 128 0.0.0.0:22 0.0.0.0:*

=== 3. ping 回环 ===
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.03 ms
```

C++ 服务端：通过命令行参数选择监听地址。

```cpp
// listen_addr_demo.cpp
// 编译：g++ -std=c++17 -O2 listen_addr_demo.cpp -o listen_addr_demo
// 运行：./listen_addr_demo 127.0.0.1
//      ./listen_addr_demo 0.0.0.0
// 环境：Linux/POSIX

#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
#include <iostream>
#include <string>

int main(int argc, char* argv[]) {
    std::string bind_ip = (argc > 1) ? argv[1] : "127.0.0.1";
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
    addr.sin_port = htons(port);

    if (inet_pton(AF_INET, bind_ip.c_str(), &addr.sin_addr) != 1) {
        std::cerr << "invalid bind ip\n";
        close(serverSocket);
        return 1;
    }

    if (bind(serverSocket, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        std::cerr << "bind failed\n";
        close(serverSocket);
        return 1;
    }

    if (listen(serverSocket, 5) < 0) {
        std::cerr << "listen failed\n";
        close(serverSocket);
        return 1;
    }

    std::cout << "listening on " << bind_ip << ":" << port << "\n";

    while (true) {
        sockaddr_in client{};
        socklen_t clientLen = sizeof(client);
        int conn = accept(serverSocket,
                          reinterpret_cast<sockaddr*>(&client),
                          &clientLen);
        if (conn < 0) continue;

        char clientIp[INET_ADDRSTRLEN] = {0};
        inet_ntop(AF_INET, &client.sin_addr, clientIp, sizeof(clientIp));
        std::cout << "accepted connection from " << clientIp << "\n";

        const char* msg = "hello\n";
        send(conn, msg, std::strlen(msg), 0);
        close(conn);
    }

    close(serverSocket);
    return 0;
}
```

运行步骤：

```bash
# 终端 1：只监听回环
./listen_addr_demo 127.0.0.1

# 终端 2：本机回环访问（应成功）
nc -vz 127.0.0.1 18081

# 终端 3：用本机非回环 IP 访问（通常失败）
HOST_IP=$(ip -4 addr show scope global | awk '/inet /{print $2; exit}' | cut -d/ -f1)
nc -vz "$HOST_IP" 18081
```

输出/现象/结论可能类似：

```text
# 终端 1
listening on 127.0.0.1:18081
accepted connection from 127.0.0.1

# 终端 2
127.0.0.1:18081 open

# 终端 3
nc: connect to 192.168.1.10 port 18081 (tcp) failed: Connection refused
```

再把服务端改成 `./listen_addr_demo 0.0.0.0`，重复用本机非回环 IP 访问，通常就能连上，并且服务端打印的客户端 IP 是本机非回环地址。  
验证结果：listen `127.0.0.1` 只绑回环，用本机非回环 IP 访问通常失败；listen `0.0.0.0` 绑所有 IPv4 地址，本机 IP 和回环都能访问，外部是否可达还取决于防火墙。  
注意：`nc -vz` 的可用性和参数在不同系统上可能不同；用本机非回环 IP 访问本机服务时，数据可能仍走 `lo`，但监听范围决定了连接是否被接受。容器或权限受限环境里结果可能不同。

<br>

<br>

## 自测三问
***
1. 断网为什么还能 ping 通 127.0.0.1？回环地址和私有 IP 有什么区别？  
2. localhost 和 127.0.0.1 是什么关系？它一定解析到 127.0.0.1 吗？  
3. `0.0.0.0` 作为监听地址和作为目标地址分别什么意思？listen `0.0.0.0` 和 listen `127.0.0.1` 在访问范围上有什么不同？

<br>

<br>

## 一句话总结全章
***
**127.0.0.1 是回环地址，数据不出真网卡，断网也能通；localhost 是默认指向它的域名；0.0.0.0 监听表示绑本机所有 IPv4 地址，但不能作为连接目标；本机 IP 和回环访问本机服务通常都走 lo。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>