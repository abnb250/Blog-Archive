# 27 TCP 缺陷与 QUIC：队头阻塞、连接迁移、QUIC 可靠传输与 0-RTT
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
你打开一个 HTTP/2 网页，多个资源复用在一条 TCP 连接上。结果一个包丢了，所有资源都跟着等。

你可能会想：**HTTP/2 不是有多路复用吗？为什么一个丢包还能阻塞所有流？**

再想一个场景：你从 4G 切到 WiFi，IP 变了，原来的 TCP 连接直接断，得重新握手。**为什么换个网就得重连？**

TCP 已经这么成熟了，为什么还要在 UDP 上重新造一套可靠传输？

这一章咱们用一条问题链往前走：

1. TCP 哪里不够好，为什么说它“可靠但僵化”？
2. HTTP/2 多路复用为什么仍可能被一个丢包阻塞，根子在 HTTP/2 还是 TCP？
3. 4G 切 WiFi 为什么必须重连，四元组到底绑死了什么？
4. QUIC 基于 UDP，怎么做到可靠、安全、快、可迁移？

<br>

<br>

## 优先级说明
***
这篇是 TCP 缺陷与 QUIC 的核心，面试常问，理解 HTTP/3、现代网络协议也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- TCP 的四个典型缺陷：内核升级慢、握手延迟、全局队头阻塞、连接迁移成本高
- HTTP/2 的队头阻塞根子在 TCP 层，不是 HTTP/2 Stream 设计本身
- TCP 用四元组标识连接，IP 变化就要断开重连
- QUIC 在 UDP 之上实现可靠传输、TLS、多路复用和连接迁移
- QUIC 每条 Stream 独立，某流丢包只阻塞该流
- QUIC 用连接 ID 标识连接，不靠四元组，支持连接迁移
- QUIC 内含 TLS 1.3，首次 1 RTT，会话恢复可 0-RTT
- QUIC 重传包也换新 Packet Number，避免 TCP 重传歧义
- 0-RTT 有重放攻击风险，敏感请求慎用

**了解即可（知道大概，不展开）：**
- QUIC 的具体帧格式和包类型
- BBR 等拥塞控制算法在 QUIC 中的实现细节
- 连接迁移在 NAT 环境下的具体表现
- HTTP/3 的 QPACK 头部压缩

**主线**：TCP 可靠但僵化 → 内核升级慢、握手多、丢包全连接等、换网要重连 → QUIC 借 UDP 的壳，在应用层重造可靠传输、TLS、多路复用和连接迁移。

<br>

<br>

## 知识点 1：TCP 的四个缺陷——可靠但僵化
***
### 问题

TCP 哪里不够好？为什么说它“可靠但僵化”？HTTP/2 明明有多路复用，为什么一个丢包还能阻塞所有流？4G 切 WiFi 为什么必须重连？

**你可能会想**：TCP 这么成熟，应该没什么大问题吧？

可靠是真的，但僵化也是真的。**TCP 在内核里实现，升级要动操作系统；用四元组标识连接，IP 一变就断；是字节流按序交付，一个包丢了后面全得等。** 这些在早期不是大问题，但在现代高并发、移动网络场景下，就成了瓶颈。

<br>

### 核心概念与方案

先把 TCP 换成你熟悉的场景：**TCP 像一列老火车，轨道铺好了很难改，换条线路就得重新买票。**

TCP 的四个典型缺陷可以这样看：

- **升级困难**：TCP 在内核实现，升级要动操作系统，新特性推广慢；很多优化只能等内核版本，应用层干着急。
- **建立连接延迟**：TCP 三次握手 + TLS 握手，HTTPS 首次连接可能多个 RTT；TCP Fast Open 能缓解，但需双方支持，部署受限。
- **队头阻塞**：TCP 是字节流，必须按序交付；HTTP/2 多个 Stream 复用一条 TCP，丢一个包，TCP 就要等重传，后面所有 Stream 都读不了。
- **连接迁移成本高**：TCP 用四元组标识连接，IP 变化（4G 切 WiFi）就要断开重连，不能复用原连接。

一句话：**TCP 可靠但僵化：内核升级慢、握手多、丢包全连接等、换网要重连。**

<br>

### 最小实验：只看一个丢包为什么阻塞所有 Stream

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 且支持 C++17。下面只做本地模拟，不联网。

保存为 `hol_blocking.cpp`：

````cpp
#include <iostream>
#include <vector>

int main() {
    // HTTP/2 多个 Stream 复用一条 TCP 字节流
    std::vector<int> tcp_packets = {1, 2, 3, 4, 5};
    int lost = 2;

    std::cout << "TCP 字节流: ";
    for (int p : tcp_packets) std::cout << p << " ";
    std::cout << "\n包 " << lost << " 丢失\n\n";

    std::cout << "HTTP/2 over TCP：\n";
    for (int p : tcp_packets) {
        if (p < lost) {
            std::cout << "  Stream 数据 " << p << " 可交付\n";
        } else if (p == lost) {
            std::cout << "  包 " << p << " 丢失，TCP 等待重传\n";
        } else {
            std::cout << "  包 " << p << " 已到，但 TCP 按序交付，仍阻塞\n";
        }
    }

    std::cout << "\n结论：TCP 是字节流，必须按序交付；一个包丢了，后面所有 Stream 都得等。\n";
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 hol_blocking.cpp -o hol_blocking
./hol_blocking
````

**这次只看一件事**：包 2 丢失后，包 3、4、5 是不是也被阻塞了。

预期输出：

````text
TCP 字节流: 1 2 3 4 5 
包 2 丢失

HTTP/2 over TCP：
  Stream 数据 1 可交付
  包 2 丢失，TCP 等待重传
  包 3 已到，但 TCP 按序交付，仍阻塞
  包 4 已到，但 TCP 按序交付，仍阻塞
  包 5 已到，但 TCP 按序交付，仍阻塞

结论：TCP 是字节流，必须按序交付；一个包丢了，后面所有 Stream 都得等。
````

**关键观察点**：包 3、4、5 虽然到了，但因为包 2 没到，TCP 不交付，应用层读不到。

**结论**：HTTP/2 的 Stream 多路复用是在应用层分的，但底层只有一条 TCP 字节流；TCP 不按序交付，所有 Stream 都得等。**根子在 TCP，不在 HTTP/2 Stream 设计。**

如果包 3、4、5 能独立交付，说明模拟逻辑写错了。

<br>

### 易错点与边界

坑一：**HTTP/2 的队头阻塞在 TCP 层，不是 HTTP/2 Stream 设计本身**；Stream 已经分开了，但 TCP 不分开。

坑二：**TCP Fast Open 能省 RTT，但部署受限**，不是所有客户端、服务端和中间设备都支持。

坑三：**TCP 的可靠是按字节流按序交付**，不是按消息、按流交付；这正是它阻塞所有 Stream 的根源。

坑四：**连接迁移不是“换个 IP 自动续上”**，TCP 四元组变了，内核就认为是另一条连接。

<br>

### 一句话本质

**TCP 可靠但僵化：内核升级慢、握手多、丢包全连接等、换网要重连；HTTP/2 的队头阻塞根子在 TCP 按序交付字节流。**

<br>

### 自测

HTTP/2 多路复用为什么仍可能被一个丢包阻塞？这个阻塞发生在 HTTP/2 层还是 TCP 层？

<details><summary>答案</summary>HTTP/2 的多个 Stream 复用一条 TCP 连接，但 TCP 是字节流，必须按序交付。一个包丢了，TCP 要等重传，后面所有 Stream 的数据即使到了也读不了。所以阻塞发生在 TCP 层，不是 HTTP/2 Stream 设计本身；Stream 已经在应用层分开了，但底层 TCP 不分开。</details>

<br>

<br>

## 知识点 2：QUIC 怎么解决——借 UDP 的壳，在应用层重造运输层
***
### 问题

QUIC 基于 UDP，怎么做到可靠、安全、快、可迁移？UDP 不是不可靠吗？为什么重传包还要换新编号？0-RTT 又有什么风险？

**你可能会想**：UDP 不可靠，QUIC 基于 UDP，那它怎么保证可靠？

QUIC 不是“UDP 直接可靠”，而是**在 UDP 之上自己实现可靠传输、加密和拥塞控制**。UDP 只是承载，可靠性由 QUIC 补上。

<br>

### 核心概念与方案

先把 QUIC 换成你熟悉的场景：**QUIC 借 UDP 的壳，在应用层重造可靠传输、TLS、多路复用和连接迁移，绕开 TCP 的内核僵化。**

QUIC 在应用层基于 UDP 重新实现运输层能力：

- **可靠传输**：Packet Number 严格递增，重传包也换新编号，避免 TCP 重传歧义；Stream ID + Offset 保证流内有序。
- **无队头阻塞**：每条 Stream 独立滑动窗口，某流丢包只阻塞该流，其他流完整就交付。
- **更快握手**：QUIC 内含 TLS 1.3，首次 1 RTT 同时完成连接和密钥协商；会话恢复可 0-RTT。
- **连接迁移**：用连接 ID 标识连接，不靠四元组，IP 变化可复用原连接。
- **流量控制**：Stream 级和 Connection 级窗口；拥塞控制可在应用层快速迭代，如 BBR。

一句话：**QUIC 借 UDP 的壳，在应用层重造可靠传输、TLS、多路复用和连接迁移，绕开 TCP 的内核僵化。**

<br>

### 最小实验：只看 QUIC 丢包时是不是只阻塞单流

环境前提：同上。下面只做本地模拟，不联网。

保存为 `quic_streams.cpp`：

````cpp
#include <iostream>
#include <vector>

int main() {
    // QUIC 每条 Stream 独立
    std::vector<int> stream_a = {1, 2, 3};
    std::vector<int> stream_b = {4, 5, 6};
    std::vector<int> stream_c = {7, 8, 9};

    int lost_a = 2;

    std::cout << "Stream A: ";
    for (int p : stream_a) std::cout << p << " ";
    std::cout << "\nStream A 包 " << lost_a << " 丢失\n\n";

    std::cout << "QUIC：\n";
    for (int p : stream_a) {
        if (p < lost_a) {
            std::cout << "  Stream A 数据 " << p << " 可交付\n";
        } else if (p == lost_a) {
            std::cout << "  Stream A 包 " << p << " 丢失，只阻塞 Stream A\n";
        } else {
            std::cout << "  Stream A 包 " << p << " 等到重传后才交付\n";
        }
    }
    for (int p : stream_b) {
        std::cout << "  Stream B 数据 " << p << " 可交付\n";
    }
    for (int p : stream_c) {
        std::cout << "  Stream C 数据 " << p << " 可交付\n";
    }

    std::cout << "\n结论：QUIC 每条流独立，A 丢包只阻塞 A，B/C 完整就交付。\n";
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 quic_streams.cpp -o quic_streams
./quic_streams
````

**这次只看一件事**：Stream A 丢包时，Stream B 和 C 是不是还能交付。

预期输出：

````text
Stream A: 1 2 3 
Stream A 包 2 丢失

QUIC：
  Stream A 数据 1 可交付
  Stream A 包 2 丢失，只阻塞 Stream A
  Stream A 包 3 等到重传后才交付
  Stream B 数据 4 可交付
  Stream B 数据 5 可交付
  Stream B 数据 6 可交付
  Stream C 数据 7 可交付
  Stream C 数据 8 可交付
  Stream C 数据 9 可交付

结论：QUIC 每条流独立，A 丢包只阻塞 A，B/C 完整就交付。
````

**关键观察点**：Stream A 丢包，Stream B 和 C 是不是还能正常交付。

是。**QUIC 解决的是流与流之间的队头阻塞，不是单流内部**；单流内丢包仍要等重传。

<br>

### 易错点与边界

坑一：**QUIC 不是“UDP 直接可靠”**，而是在 UDP 之上实现可靠传输、加密和拥塞控制。

坑二：**0-RTT 有重放攻击风险**，敏感请求慎用；0-RTT 数据可能被攻击者重放。

坑三：**单流内仍要等丢包重传**，QUIC 解决的是流与流之间的队头阻塞，不是单流内部。

坑四：**Packet Number 重传也递增，不是复用旧编号**；这样 ACK 和 RTT 采样不会把原包和重传包搞混。

坑五：**连接迁移靠连接 ID，不是靠 IP**；但迁移能否成功还取决于网络和实现支持。

<br>

### 一句话本质

**QUIC 借 UDP 的壳，在应用层重造可靠传输、TLS、多路复用和连接迁移；每条流独立，某流丢包只阻塞该流，连接靠 ID 不靠四元组。**

<br>

### 自测

QUIC 的 Packet Number 为什么重传也要递增？QUIC 首次 1-RTT、再次 0-RTT 分别靠什么？

<details><summary>答案</summary>QUIC 的 Packet Number 重传也递增，是为了避免 TCP 重传歧义：ACK 和 RTT 采样不会把原包和重传包搞混，发送方可以更精确地判断哪个包被确认了。QUIC 首次连接靠内含的 TLS 1.3，1 RTT 同时完成传输握手和密钥协商；会话恢复靠之前的会话密钥或 PSK，可 0-RTT 发送数据，但 0-RTT 有重放风险，敏感请求慎用。</details>

<br>

<br>

## 完整可复现示例：浏览器看 HTTP/3
***
这个示例不需要写代码，用浏览器开发者工具观察站点是否走 HTTP/3（QUIC）。

操作步骤：

1. Chrome 打开 `chrome://flags/#enable-quic`，确认 Experimental QUIC protocol 为 Enabled；如果已经是 Default/Enabled，可跳过。
2. 访问 `https://cloudflare-quic.com` 或 `https://www.google.com`。
3. F12 → Network → 右键表头勾选 Protocol。
4. 刷新页面，看 Protocol 列是否显示 `h3`。

输出/现象/结论可能类似：

````text
Protocol 显示 h3 表示走 HTTP/3（QUIC）；
显示 h2 表示 HTTP/2 over TCP；
显示 http/1.1 表示 HTTP/1.1。
````

**关键观察点**：Protocol 列有没有 `h3`。

验证结果：QUIC 已用于 HTTP/3；是否生效取决于浏览器、服务器和网络支持。

注意：不同 Chrome 版本、不同网络环境下结果可能不同；有些网络会限制 UDP，导致 HTTP/3 回退到 HTTP/2。看不到 `h3` 不代表 QUIC 不存在，可能只是当前链路不支持，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| HTTP/2 网页一个资源慢，其他资源也卡 | TCP 队头阻塞 | 抓包看丢包和重传；看是否所有 Stream 一起等 | TCP 是字节流，丢一个包后面全等；HTTP/2 Stream 无法绕过 TCP 按序交付 |
| 4G 切 WiFi 后连接断 | TCP 四元组变化 | 抓包看源 IP 是否变化；看是否重新握手 | TCP 用四元组标识连接，IP 变了就认为是新连接，必须重连 |
| HTTP/3 没生效，回退到 h2 | UDP 是否被限制、浏览器/服务器支持 | 查浏览器 flags；抓包看 UDP 443 | 部分网络限制 UDP，HTTP/3 会回退到 HTTP/2 |
| 0-RTT 请求被重放 | 是否对敏感请求开了 0-RTT | 查服务器配置和业务日志 | 0-RTT 有重放风险，敏感操作应禁用或加防重放 |
| QUIC 单流内仍卡 | 是否单流丢包 | 抓包看该流重传 | QUIC 解决流间队头阻塞，单流内丢包仍要等重传 |
| 连接迁移失败 | 网络是否支持、实现是否兼容 | 看连接 ID 是否变化；查客户端/服务端日志 | 连接迁移靠连接 ID，但网络和实现支持都会影响 |
| TCP Fast Open 没生效 | 双方和中间设备是否支持 | 抓包看 TFO 选项；查内核参数 | TFO 需双方支持，中间设备也可能干扰 |

<br>

<br>

## 常见误区
***
- **以为 HTTP/2 多路复用能完全消除队头阻塞**：HTTP/2 的 Stream 在应用层分开，但底层 TCP 仍是字节流，丢包会阻塞所有 Stream。
- **以为 TCP 连接迁移是自动的**：TCP 用四元组标识连接，IP 变了就断，不能复用。
- **以为 QUIC 就是“UDP 直接可靠”**：QUIC 在 UDP 之上自己实现可靠传输、加密和拥塞控制。
- **以为 QUIC 完全没有队头阻塞**：流与流之间没有全局阻塞，单流内丢包仍要等重传。
- **以为 QUIC 重传包复用旧编号**：Packet Number 重传也递增，避免 ACK 和 RTT 采样歧义。
- **以为 0-RTT 越多越好**：0-RTT 有重放风险，敏感请求慎用。
- **以为连接迁移一定能成功**：靠连接 ID，但网络和实现支持都会影响。

<br>

<br>

## 自测三问
***
1. TCP 的队头阻塞和 HTTP/2 的队头阻塞是什么关系？为什么说根子在 TCP？
2. QUIC 如何做到连接迁移？它为什么不靠四元组？
3. QUIC 首次 1-RTT、再次 0-RTT 分别靠什么？0-RTT 的风险是什么？

<details><summary>参考答案</summary>1. HTTP/2 的多个 Stream 复用一条 TCP 连接，但 TCP 是字节流，必须按序交付。一个包丢了，TCP 要等重传，后面所有 Stream 都读不了。所以 HTTP/2 的队头阻塞根子在 TCP 层，不是 Stream 设计本身。2. QUIC 用连接 ID 标识连接，不靠源/目的 IP 和端口四元组；IP 变化时连接 ID 不变，可以复用原连接。3. QUIC 首次连接靠内含的 TLS 1.3，1 RTT 同时完成传输握手和密钥协商；会话恢复靠之前的会话密钥或 PSK，可 0-RTT 发送数据。0-RTT 的风险是重放攻击，敏感请求应禁用或加防重放。</details>

<br>

<br>

## 总结
***
**TCP 的可靠换来内核僵化、握手延迟、全局队头阻塞和连接迁移成本；QUIC 在 UDP 之上用递增 Packet Number、独立 Stream、内置 TLS1.3、连接 ID 和可迭代拥塞控制，重造了一个更快更灵活的运输层；单流内仍要等重传，0-RTT 有重放风险。**

<br>

<br>

## 参考资料
***
- RFC 9000：QUIC: A UDP-Based Multiplexed and Secure Transport
- RFC 9001：Using TLS to Secure QUIC
- RFC 9002：QUIC Loss Detection and Congestion Control
- RFC 9114：HTTP/3
- RFC 7413：TCP Fast Open
- 需按环境核实：浏览器对 HTTP/3 的支持和 flags 设置因版本而异；部分网络限制 UDP 443 会导致 HTTP/3 回退；QUIC 连接迁移能否成功取决于网络和实现；0-RTT 的重放风险需结合业务评估。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>