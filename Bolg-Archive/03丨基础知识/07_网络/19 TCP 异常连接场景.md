# 19 TCP 异常连接场景：SYN 丢弃、Challenge ACK、TIME_WAIT 收到 SYN、崩溃/断电/拔网线
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
正常握手挥手背得熟，面试和工作常问异常：**SYN 为什么会丢？已建立连接收到 SYN 会怎样？TIME_WAIT 收到 SYN 回 RST 吗？进程崩溃和主机断电有什么区别？拔网线后连接还在吗？**  
先别急着背“半连接队列、Challenge ACK、Keepalive”这些词，先看几个具体场景：你压测服务时，客户端 `connect` 一直超时，服务端却像没收到 SYN 一样，为什么？一台已经 ESTABLISHED 的连接，突然收到一个同四元组的 SYN，内核会怎么处理？进程 `kill -9` 和主机直接断电，对端看到的现象一样吗？这一章咱们就按这些异常场景走一遍。

<br>

<br>

## 知识点 1：SYN 被丢弃的常见场景
***
### 问题
服务端收到 SYN 却不回 SYN+ACK，为什么？是网络丢了，还是服务端根本没处理？半连接队列和全连接队列满了分别会怎样？

<br>

### 方案
SYN 被丢弃，常见有三类原因：  
1. **半连接队列满**：SYN 攻击或突发 SYN 太多，内核丢弃新 SYN。开启 `tcp_syncookies=1` 可以绕过半连接队列，用 cookie 代替队列项，缓解 SYN Flood。  
2. **全连接队列满**：第三次 ACK 到达时 `accept` 队列满，默认丢弃 ACK；设置 `tcp_abort_on_overflow=1` 则回 RST。  
3. **`tcp_tw_recycle` + NAT**：旧参数按“对端 IP”做 PAWS 检查，NAT 后多个客户端同 IP，时间戳小的 SYN 可能被丢；Linux 4.12 后已取消该参数，现代内核不用再担心这一条。  
排查时看内核统计：`netstat -s` 里 “SYNs to LISTEN sockets dropped” 和 “overflowed” 持续增长，说明队列溢出。

<br>

### 最小示例
```bash
netstat -s 2>/dev/null | grep -i "SYNs to LISTEN sockets dropped"
netstat -s 2>/dev/null | grep -i overflowed
```

输出可能类似：

```text
    12345 SYNs to LISTEN sockets dropped
    678 times the listen queue of a socket overflowed
```

结论：数值持续增长说明队列溢出；只看一眼没用，要看趋势。

<br>

### 一句话本质
**SYN 被丢，多半是队列满了或被历史参数误杀；syncookies 是半连接队列的应急旁路。**

<br>

### 易错点
坑一：**只增大 `tcp_max_syn_backlog` 不一定有效**，半连接队列还受 `somaxconn`、应用 `listen` 的 `backlog` 影响。  
坑二：**`tcp_tw_recycle` 在 NAT 下不安全**，Linux 4.12 后已移除，现代内核不用再调这个参数。  
坑三：**全连接队列满默认丢 ACK，不回 RST**，客户端可能一直重试；开 `tcp_abort_on_overflow` 才会回 RST。  
坑四：**syncookies 不是万能**，它绕过半连接队列，但全连接队列满仍然会出问题。

<br>

### 自测
半连接队列满了，开启 syncookies 后 SYN 还会被丢吗？全连接队列满和半连接队列满，客户端看到的现象一样吗？

<br>

<br>

## 知识点 2：已建立连接收到 SYN、TIME_WAIT 收到 SYN
***
### 问题
ESTABLISHED 状态收到同四元组 SYN 怎么办？TIME_WAIT 收到 SYN 一定回 RST 吗？为什么有时能复用连接，有时又回 ACK 让客户端 RST？

<br>

### 方案
**ESTABLISHED 收到 SYN**（通常是旧 SYN 乱序到达）：服务端回 **Challenge ACK**，携带正确序列号和确认号；客户端发现确认号不是自己期望的，回 RST，服务端释放连接。Challenge ACK 的作用是“你发的 SYN 不对，我告诉你我真正想要哪个序号”，而不是直接丢包让客户端干等。  
**TIME_WAIT 收到 SYN**：看 SYN 是否合法。双方开时间戳时，合法 SYN 要求 SYN 的序列号大于期望下一个序列号，且时间戳更大；合法则复用四元组，跳过 2MSL 进入 SYN_RECV；非法则回第四次挥手 ACK，客户端发现不对回 RST。  
**TIME_WAIT 收到 RST**：`tcp_rfc1337=0` 会提前释放；为 1 则丢弃 RST。  
所以不是所有 TIME_WAIT 收到 SYN 都回 RST；合法 SYN 可以复用连接。

<br>

### 最小示例
```text
Established 收到旧 SYN → Challenge ACK → 客户端发现 Ack 不对 → RST
TIME_WAIT 收到非法 SYN → 回上次 ACK → 客户端 RST
TIME_WAIT 收到合法 SYN → 复用四元组，进入 SYN_RECV
```

结论：Challenge ACK 是纠正，不是丢弃；TIME_WAIT 收到 SYN 要看新旧是否合法。

<br>

### 一句话本质
**Challenge ACK 是“你发的 SYN 不对，我告诉你我真正想要哪个序号”；TIME_WAIT 收到 SYN 要看新旧是否合法。**

<br>

### 易错点
坑一：**不是所有 TIME_WAIT 收到 SYN 都回 RST**，合法 SYN 可以复用连接。  
坑二：**Challenge ACK 不是 SYN+ACK**，它不建立新连接，只是告诉对方当前期望的序号。  
坑三：**`tcp_rfc1337` 影响 TIME_WAIT 是否被 RST 提前结束**，默认 0 会提前释放，开 1 更严格。  
坑四：**旧 SYN 乱序到达是常见现象**，不是攻击；Challenge ACK 让客户端自己纠正，避免服务端误建连。

<br>

### 自测
Established 收到 SYN 后为什么回 Challenge ACK 而不是直接丢？TIME_WAIT 收到 SYN 在什么情况下可以复用连接？

<br>

<br>

## 知识点 3：进程崩溃、主机宕机、拔网线
***
### 问题
一端异常后，另一端怎么感知？进程崩溃和主机断电，对端看到的现象一样吗？拔网线后连接会立即断开吗？Keepalive 到底靠不靠谱？

<br>

### 方案
三种异常，内核知道的程度不同：  
- **进程崩溃**：内核回收进程资源，会发 FIN，通常能完成四次挥手。对端看到正常的 FIN，连接正常关闭。  
- **主机宕机/断电**：无法发 FIN；若没有数据要发且没开 TCP Keepalive，对端可能一直 ESTABLISHED，以为连接还活着。  
- **拔网线**：不改变内核 TCP 状态；有数据发送时靠超时重传，重传超阈值后断开；没数据且没 Keepalive 则连接可能一直存在。  
TCP Keepalive 默认 2 小时才启动，适合兜底；应用层可自己做心跳。HTTP Keep-Alive 是应用层长连接，不是 TCP Keepalive。  
一句话：进程崩溃内核会发 FIN；主机宕机/拔网线内核不知道，只能靠数据重传或 Keepalive 探测。

<br>

### 最小示例
查看默认 Keepalive 参数：

```bash
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_intvl net.ipv4.tcp_keepalive_probes
```

输出可能类似：

```text
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
```

结论：默认 7200 秒（2 小时）才开始探测，75 秒一次，9 次失败才判死；发现死亡连接很慢，线上常需应用层心跳。

<br>

### 一句话本质
**进程崩溃内核会发 FIN；主机宕机/拔网线内核不知道，只能靠数据重传或 Keepalive 探测。**

<br>

### 易错点
坑一：**ESTABLISHED 不代表对端一定活着**，可能只是没有数据要发，也没开 Keepalive。  
坑二：**TCP Keepalive 和 HTTP Keep-Alive 不是一回事**；前者是 TCP 层探测，后者是 HTTP 层连接复用。  
坑三：**Keepalive 要应用设置 `SO_KEEPALIVE` 才生效**，不是内核默认对所有连接都开。  
坑四：**拔网线不会立即改变内核 TCP 状态**，没有数据发送时连接可能一直挂着。  
坑五：**Keepalive 默认 2 小时才启动**，不能指望它快速发现故障；应用层心跳更及时。

<br>

### 自测
拔掉网线后，原本 TCP 连接一定立即断开吗？进程崩溃和主机断电，对端看到的 TCP 行为有什么关键区别？

<br>

<br>

## 完整可复现示例：观察 Keepalive 参数并设置 SO_KEEPALIVE
***
这个示例分两部分：先用 `sysctl` 和 `netstat` 看系统默认参数和队列统计；再用 C++ 写一个最小程序，给 TCP 套接字打开 `SO_KEEPALIVE`，并查看当前 Keepalive 参数。  
注意：改系统参数需要权限；这里只读不改，避免影响本机网络。

```bash
set -e

echo "=== 1. TCP Keepalive 参数 ==="
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_intvl net.ipv4.tcp_keepalive_probes

echo "=== 2. 队列溢出统计 ==="
netstat -s 2>/dev/null | grep -i -E "SYNs to LISTEN|overflowed" || true
```

输出可能类似：

```text
=== 1. TCP Keepalive 参数 ===
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9

=== 2. 队列溢出统计 ===
    12345 SYNs to LISTEN sockets dropped
    678 times the listen queue of a socket overflowed
```

C++ 程序：给套接字打开 `SO_KEEPALIVE`，并读取当前 Keepalive 参数。

```cpp
// keepalive_demo.cpp
// 编译：g++ -std=c++17 -O2 keepalive_demo.cpp -o keepalive_demo
// 运行：./keepalive_demo
// 环境：Linux/POSIX

#include <netinet/in.h>
#include <netinet/tcp.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
#include <iostream>

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    // 打开 TCP Keepalive
    int keepalive = 1;
    if (setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE,
                   &keepalive, sizeof(keepalive)) < 0) {
        std::cerr << "setsockopt SO_KEEPALIVE failed\n";
        close(sock);
        return 1;
    }

    // 读取当前 Keepalive 参数
    int idle = 0, intvl = 0, cnt = 0;
    socklen_t len = sizeof(idle);

    if (getsockopt(sock, IPPROTO_TCP, TCP_KEEPIDLE, &idle, &len) == 0) {
        std::cout << "TCP_KEEPIDLE = " << idle << "s\n";
    } else {
        std::cout << "TCP_KEEPIDLE not available\n";
    }

    len = sizeof(intvl);
    if (getsockopt(sock, IPPROTO_TCP, TCP_KEEPINTVL, &intvl, &len) == 0) {
        std::cout << "TCP_KEEPINTVL = " << intvl << "s\n";
    } else {
        std::cout << "TCP_KEEPINTVL not available\n";
    }

    len = sizeof(cnt);
    if (getsockopt(sock, IPPROTO_TCP, TCP_KEEPCNT, &cnt, &len) == 0) {
        std::cout << "TCP_KEEPCNT = " << cnt << "\n";
    } else {
        std::cout << "TCP_KEEPCNT not available\n";
    }

    close(sock);
    return 0;
}
```

输出可能类似：

```text
TCP_KEEPIDLE = 7200s
TCP_KEEPINTVL = 75s
TCP_KEEPCNT = 9
```

验证结果：默认 Keepalive 很慢；线上常需应用层心跳；队列统计可判断 SYN/accept 队列是否溢出。`SO_KEEPALIVE` 只是打开开关，具体空闲时间、间隔、次数可以用 `TCP_KEEPIDLE`、`TCP_KEEPINTVL`、`TCP_KEEPCNT` 按连接调。  
注意：不同系统、不同内核版本输出可能不同；`netstat -s` 的字段名也可能有差异。没有权限或没有 `netstat` 时，只看 C++ 部分的输出也能理解 Keepalive 参数。

<br>

<br>

## 自测三问
***
1. Established 收到旧 SYN 回什么？客户端随后做什么？  
2. TIME_WAIT 收到 SYN 一定回 RST 吗？什么情况下可以复用连接？  
3. 进程崩溃和主机宕机在 TCP 层面有什么关键区别？Keepalive 为什么不能快速发现故障？

<br>

<br>

## 一句话总结全章
***
**TCP 异常场景核心看状态和报文合法性：SYN 可能因队列满被丢，Established 收到旧 SYN 用 Challenge ACK 纠正，TIME_WAIT 收到 SYN 看新旧，进程崩溃会发 FIN，主机宕机/拔网线只能靠重传或 Keepalive 兜底。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>