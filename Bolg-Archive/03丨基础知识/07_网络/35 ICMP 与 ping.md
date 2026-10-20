# 35 ICMP 与 ping：网络层的“报错与探活”
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
网络不通时，大家第一反应是 `ping`。**ping 到底发了什么？为什么它能判断通不通？ICMP 除了 ping 还负责什么？**  
先别急着把 `ping` 当成一个命令背下来，先看几个具体场景：你 ping 一台主机，显示“目标不可达”，这个提示是谁发回来的？你 ping 通了一台服务器，是不是就说明它的 80 端口一定开着？你 ping 一个域名，显示 `ttl=64`，这个 TTL 是什么，和 IP 首部里的 TTL 是一回事吗？这一章咱们就按“ICMP 报什么、ping 怎么探活”这条线走。

<br>

<br>

## 知识点 1：ICMP 是 IP 的助手
***
### 问题
IP 包送不到，怎么通知源主机？ICMP 是传输层协议吗？它负责可靠传输吗？ping 通目标主机，能说明目标主机的 80 端口一定开着吗？

<br>

### 方案
ICMP 是**网络层协议**，封装在 IP 包里，负责报告 IP 包传输中的问题，比如目标不可达、超时、重定向等。它分两类：  
- **查询报文**：如回送请求/应答、时间戳请求/应答；  
- **差错报文**：如目标不可达、时间超时、参数问题。  
关键边界：**ICMP 不负责可靠传输，只负责“报信”**。它本身也可能丢，不会重传；它告诉发送方“为什么没送到”，但不保证一定送到。  
另外，ICMP 差错报文通常不会对 ICMP 差错报文再发差错，避免无限循环；分片时也只对第一个分片发差错，这些是协议设计上的细节。  
ping 通只说明 ICMP 往返可达，不保证某个 TCP/UDP 端口服务正常；端口是否开放，要看传输层有没有响应。

<br>

### 最小示例
```text
路由器找不到目标主机 → 回 ICMP 目标不可达给源主机
TTL 减到 0 → 回 ICMP 时间超时
目标主机禁 ICMP → ping 不通，但 TCP 80 可能仍然可用
```

结论：ICMP 是 IP 的报错助手，包没送到，至少告诉你为什么；但它不是“服务探活”的万能工具。

<br>

### 一句话本质
**ICMP 是 IP 的报错助手：包没送到，至少告诉你为什么。**

<br>

### 易错点
坑一：**ICMP 不是传输层协议**，它封装在 IP 里，属于网络层。  
坑二：**ICMP 本身也可能丢**，ping 不通不一定是网络完全断，也可能是 ICMP 被限速或禁用。  
坑三：**ping 通只说明 ICMP 往返可达，不保证某个 TCP/UDP 端口服务正常**。  
坑四：**ping 不通不代表一定断网**，有的主机或防火墙会禁 ICMP，但 TCP 端口仍然可用。  
坑五：**ICMP 差错报文不会对 ICMP 差错报文再发差错**，避免循环；别以为每个错误都会触发新的错误消息。

<br>

### 自测
ping 通目标主机，能说明目标主机的 80 端口一定开着吗？为什么 ICMP 差错报文不会对另一个 ICMP 差错报文再报错？

<br>

<br>

## 知识点 2：ping 的工作过程
***
### 问题
ping 怎么判断通不通、RTT 多少？ping 用的是 TCP 还是 UDP？请求和应答的 ICMP 类型分别是多少？跨网段时 MAC 怎么变？

<br>

### 方案
ping 使用 ICMP **回送请求（类型 8）** 和 **回送应答（类型 0）**。  
过程可以拆成几步：  
1. 源主机构造 ICMP 回送请求，带标识符和序号，数据区通常放发送时间；  
2. 交给 IP 层加上源/目标 IP；  
3. 查 ARP 得到下一跳 MAC，封装成帧发出；跨网段时目的 MAC 是网关 MAC，不是最终主机 MAC；  
4. 目标主机收到后，协议栈自动回 ICMP 回送应答，序号对应；  
5. 源主机收到后，用当前时间减发送时间算 RTT；超时没收到就认为不可达或丢包。  
注意：ping 的请求和应答走的是 IP 层，不经过 TCP/UDP；所以 ping 通不能替代端口探测。

<br>

### 最小示例
```text
ping 192.168.1.2
→ ICMP Echo Request type=8 seq=0
← ICMP Echo Reply type=0 seq=0
→ 计算 RTT

跨网段：
目标 IP = 8.8.8.8，但帧的目的 MAC 是网关 MAC。
```

结论：ping 就是发一个“你在吗”的 ICMP 请求，等一个“我在”的应答，顺便算往返时间。

<br>

### 一句话本质
**ping 就是发一个“你在吗”的 ICMP 请求，等一个“我在”的应答，顺便算往返时间。**

<br>

### 易错点
坑一：**ping 使用的是 ICMP，不是 TCP/UDP**；它不建立连接，也不探测端口。  
坑二：**有的主机会禁 ICMP**，所以 ping 不通不代表一定断网；要看 TCP 端口是否可达。  
坑三：**跨网段会经过路由器和 ARP 换 MAC**，ping 的 IP 不变，但帧的 MAC 每跳都换。  
坑四：**ping 显示的 `ttl=64` 是收到报文时的 TTL 剩余值**，不是发送方设置的初始 TTL；初始值常见 64、128、255。  
坑五：**ping 的 RTT 包含往返路径和中间设备处理时间**，不只是一段链路的延迟。

<br>

### 自测
ping 的请求和应答 ICMP 类型分别是多少？跨网段 ping 时，帧的目的 MAC 是目标主机还是网关？

<br>

<br>

## 完整可复现示例：用 C++ 发送 ICMP 回送请求
***
下面这个 C++ 程序用原始套接字发送一个 ICMP Echo Request，并接收 Echo Reply，打印类型、序号和 RTT。  
注意：原始套接字需要 root 权限；不同系统对 ICMP 原始套接字的限制不同。这个示例只演示最小流程，不处理校验和以外的复杂情况。

```cpp
// icmp_ping_demo.cpp
// 编译：g++ -std=c++17 -O2 icmp_ping_demo.cpp -o icmp_ping_demo
// 运行：sudo ./icmp_ping_demo 127.0.0.1
// 环境：Linux/POSIX

#include <arpa/inet.h>
#include <netinet/ip.h>
#include <netinet/ip_icmp.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <unistd.h>

#include <cstring>
#include <iostream>
#include <string>

// 计算 ICMP 校验和
uint16_t icmp_checksum(const void* data, int len) {
    const uint16_t* ptr = static_cast<const uint16_t*>(data);
    uint32_t sum = 0;
    while (len > 1) {
        sum += *ptr++;
        len -= 2;
    }
    if (len == 1) {
        sum += *reinterpret_cast<const uint8_t*>(ptr);
    }
    while (sum >> 16) {
        sum = (sum & 0xFFFF) + (sum >> 16);
    }
    return static_cast<uint16_t>(~sum);
}

int main(int argc, char* argv[]) {
    std::string target = (argc > 1) ? argv[1] : "127.0.0.1";

    int sock = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);
    if (sock < 0) {
        std::cerr << "socket failed，需要 root 权限\n";
        return 1;
    }

    sockaddr_in dst{};
    dst.sin_family = AF_INET;
    if (inet_pton(AF_INET, target.c_str(), &dst.sin_addr) != 1) {
        std::cerr << "invalid target\n";
        close(sock);
        return 1;
    }

    // 构造 ICMP Echo Request
    const int packet_size = sizeof(icmphdr) + 16;
    char packet[packet_size];
    std::memset(packet, 0, sizeof(packet));

    icmphdr* icmp = reinterpret_cast<icmphdr*>(packet);
    icmp->type = ICMP_ECHO;
    icmp->code = 0;
    icmp->un.echo.id = htons(getpid() & 0xFFFF);
    icmp->un.echo.sequence = htons(1);
    std::memset(packet + sizeof(icmphdr), 'A', 16);
    icmp->checksum = icmp_checksum(packet, packet_size);

    timeval send_time{};
    gettimeofday(&send_time, nullptr);

    if (sendto(sock, packet, packet_size, 0,
               reinterpret_cast<sockaddr*>(&dst), sizeof(dst)) < 0) {
        std::cerr << "sendto failed\n";
        close(sock);
        return 1;
    }

    std::cout << "sent ICMP Echo Request to " << target << "\n";

    // 接收回复
    char recv_buf[1024];
    sockaddr_in from{};
    socklen_t from_len = sizeof(from);

    ssize_t n = recvfrom(sock, recv_buf, sizeof(recv_buf), 0,
                         reinterpret_cast<sockaddr*>(&from), &from_len);
    if (n < 0) {
        std::cerr << "recvfrom failed\n";
        close(sock);
        return 1;
    }

    timeval recv_time{};
    gettimeofday(&recv_time, nullptr);

    // 跳过 IP 头
    iphdr* ip = reinterpret_cast<iphdr*>(recv_buf);
    int ip_header_len = ip->ihl * 4;
    icmphdr* reply = reinterpret_cast<icmphdr*>(recv_buf + ip_header_len);

    if (reply->type == ICMP_ECHOREPLY) {
        double rtt_ms = (recv_time.tv_sec - send_time.tv_sec) * 1000.0
                      + (recv_time.tv_usec - send_time.tv_usec) / 1000.0;
        std::cout << "got ICMP Echo Reply: type=" << static_cast<int>(reply->type)
                  << ", seq=" << ntohs(reply->un.echo.sequence)
                  << ", RTT=" << rtt_ms << " ms\n";
    } else {
        std::cout << "got ICMP type=" << static_cast<int>(reply->type)
                  << " (not echo reply)\n";
    }

    close(sock);
    return 0;
}
```

输出/现象/结论可能类似：

```text
sent ICMP Echo Request to 127.0.0.1
got ICMP Echo Reply: type=0, seq=1, RTT=0.05 ms
```

验证结果：`type=8` 是回送请求，`type=0` 是回送应答；`seq` 对应请求序号；RTT 是发送和接收时间差。  
注意：原始套接字需要 root 权限；不同系统对 `IPPROTO_ICMP` 原始套接字的权限限制不同；这个示例只处理 IPv4，没有处理超时、重传和多包统计，重点是看 ICMP 类型和 RTT 的计算方式。

<br>

<br>

## 自测三问
***
1. ICMP 的两大类报文是什么？ICMP 是传输层协议吗？  
2. ping 使用哪两种 ICMP 类型？跨网段时帧的目的 MAC 是目标主机还是网关？  
3. 为什么 ping 通不代表某个 TCP 端口一定可用？ping 不通又可能是什么原因？

<br>

<br>

## 一句话总结全章
***
**ICMP 是 IP 的报错与查询助手，ping 用类型 8 回送请求和类型 0 回送应答判断目标是否可达并计算 RTT；ping 通只说明 ICMP 可达，不保证具体服务端口正常。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>