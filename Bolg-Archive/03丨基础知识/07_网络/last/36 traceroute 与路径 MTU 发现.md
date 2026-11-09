# 36 traceroute 与路径 MTU 发现：用 TTL 和“不分片”探路
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
`ping` 只能告诉你通不通，但你想知道更多：

- 数据包从你电脑到目标服务器，中间到底走了哪几台路由器？
- 为什么有时大包发不出去，小包却正常？
- 你怀疑某条链路 MTU 不是 1500，怎么让路由器主动告诉你“我这儿只能过 1400”？

你可能会想：**ping 都能测通不通了，为什么还需要 traceroute？**

因为 ping 只回答“能不能到”，不回答“怎么到”。**traceroute 用 TTL 逐跳探路，路径 MTU 发现用“不分片”逼路由器报出瓶颈 MTU。**

这一章咱们用一条问题链往前走：

1. traceroute 怎么拿到沿途路由器 IP，为什么故意把 TTL 设得很小？
2. 怎么判断已经到达目标主机，而不是还在中间某跳？
3. 路径 MTU 发现怎么知道路径上最小 MTU 是多少？
4. 路由器遇到超 MTU 又不能分片的包，会怎么通知发送方？

<br>

<br>

## 优先级说明
***
这篇是网络层排障的核心，面试常问，项目里排查“路径不通”“大包黑洞”也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- traceroute 核心是故意设置小 TTL，从 1 开始递增
- TTL 减到 0，路由器回 ICMP 时间超时，暴露该跳 IP
- Linux traceroute 常用 UDP 高端口，目标回 ICMP 端口不可达表示结束
- Windows tracert 默认用 ICMP Echo，探测方式不同
- 中间显示 `*` 可能是路由器不回 ICMP，不代表那一跳不存在
- 路径不对称：去程和回程可能走不同路由器
- 路径 MTU 发现用 DF 位（不分片），超 MTU 路由器回 ICMP 代码 4
- ICMP 被防火墙拦截会导致 PMTUD 黑洞：小包能过，大包一直失败
- IPv6 中间路由器默认不分片，更依赖 PMTUD

**了解即可（知道大概，不展开）：**
- MPLS 标签对 TTL 的影响
- ICMP 限速对 traceroute 结果的影响
- `tracepath` 和 `mtr` 的用法差异
- IPv6 Packet Too Big 消息的具体格式

**主线**：TTL 逐跳递减 → TTL=0 路由器回超时 → 拼出路径 → DF 位不分片 → 超 MTU 路由器回 ICMP 代码 4 → 逐步调小找到可通行 MTU。

<br>

<br>

## 知识点 1：用 TTL 逐跳探路——每跳路由器各报一次超时
***
### 问题

traceroute 怎么拿到沿途路由器 IP？为什么故意把 TTL 设得很小？怎么判断已经到达目标主机，而不是还在中间某跳？

**你可能会想**：traceroute 是不是一次就能看到整条路径？

不是。**traceroute 靠一串“故意牺牲”的包，让每台路由器各报一次超时，拼出路径。** TTL=1 拿到第一跳，TTL=2 拿到第二跳，逐跳推进。

<br>

### 核心概念与方案

先把 TTL 换成你熟悉的场景：**TTL 像一张“最多还能走几站”的车票，每经过一个路由器就撕掉一格；撕到 0，路由器就把包拦下，并告诉你“我拦的”。**

traceroute 的核心是**故意设置小 TTL，从 1 开始递增**。TTL 每经过一个路由器减 1，减到 0 路由器就丢弃该包，并回一个 **ICMP 时间超时**。这样 TTL=1 拿到第一跳，TTL=2 拿到第二跳，逐跳推进。

Linux 的 traceroute 默认常用 **UDP 高端口**（大于 3000）发探测包；到达目标主机后，目标回 **ICMP 端口不可达**，表示路径追踪结束。Windows 的 `tracert` 默认用 ICMP Echo，行为类似但探测报文不同。

所以 traceroute 不是“一次性看到整条路径”，而是靠一串“故意牺牲”的包，让每台路由器各报一次超时，拼出路径。

<br>

### 最小实验：用 C++ 模拟 TTL 逐跳递减，只看每一跳谁回超时

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 且支持 C++17。下面只做逻辑模拟，不联网。

保存为 `ttl_trace.cpp`：

````cpp
#include <iostream>

int main() {
    const int max_hops = 5;
    int ttl = 1;

    for (int hop = 1; hop <= max_hops; ++hop) {
        std::cout << "发送探测包 TTL=" << ttl << " -> ";
        std::cout << "第 " << hop << " 跳路由器收到，TTL 减到 0，回 ICMP 超时\n";
        ttl++;
    }

    std::cout << "TTL=" << ttl << " 的探测包到达目标主机，回 ICMP 端口不可达，结束\n";
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 ttl_trace.cpp -o ttl_trace
./ttl_trace
````

**这次只看一件事**：每一跳是不是 TTL 递增 1，TTL 减到 0 时谁回超时。

预期输出：

````text
发送探测包 TTL=1 -> 第 1 跳路由器收到，TTL 减到 0，回 ICMP 超时
发送探测包 TTL=2 -> 第 2 跳路由器收到，TTL 减到 0，回 ICMP 超时
发送探测包 TTL=3 -> 第 3 跳路由器收到，TTL 减到 0，回 ICMP 超时
发送探测包 TTL=4 -> 第 4 跳路由器收到，TTL 减到 0，回 ICMP 超时
发送探测包 TTL=5 -> 第 5 跳路由器收到，TTL 减到 0，回 ICMP 超时
TTL=6 的探测包到达目标主机，回 ICMP 端口不可达，结束
````

**关键观察点**：每一跳的 TTL 是不是从 1 开始递增，TTL=0 时是不是路由器回超时。

**结论**：traceroute 就是靠 TTL 逐跳“牺牲”，让每台路由器暴露自己。

如果 TTL 没有递增或跳数不对，先检查循环变量和 TTL 初始值。

<br>

### 易错点与边界

坑一：**有的路由器不回 ICMP**，所以中间会显示 `*`，不代表那一跳不存在，可能只是被禁用或限速。

坑二：**Windows 的 `tracert` 默认用 ICMP，Linux traceroute 常用 UDP**，两者探测方式不同，结果可能略有差异。

坑三：**路径可能不对称**，去程和回程经过的路由器不一定相同，traceroute 只反映去程（或探测方向）的路径。

坑四：**目标主机可能不回端口不可达**，比如防火墙拦截，导致 traceroute 一直探到最大跳数也没结束。

坑五：**TTL 超时是路由器回的，端口不可达是目标主机回的**，两者别混。

<br>

### 一句话本质

**traceroute 用 TTL 逐跳“故意牺牲”，让每台路由器报一次超时，从而画出路径；目标主机用端口不可达告诉你“别再探了”。**

<br>

### 自测

traceroute 怎么知道已经到达目标主机？如果中间某跳显示 `*`，说明什么？

<details><summary>答案</summary>Linux traceroute 默认用 UDP 探测，目标主机收到后回 ICMP 端口不可达，表示路径追踪结束；Windows tracert 用 ICMP Echo，目标回 Echo Reply 表示到达。中间显示 `*` 表示该跳没有回 ICMP 超时，可能是路由器禁 ICMP、限速，或该跳不回探测包，不代表那一跳不存在。</details>

<br>

<br>

## 知识点 2：路径 MTU 发现——用“不分片”逼路由器报出瓶颈 MTU
***
### 问题

怎么知道路径上最小 MTU？为什么大包不分片就过不去？路由器遇到超 MTU 又不能分片的包，会怎么通知发送方？

**你可能会想**：MTU 不是链路的固定属性吗？我按 1500 发不就行了？

不一定。**路径上每段链路的 MTU 可能不同**，中间某段只有 1400，你发 1500 还不让分片，路由器只能丢弃。**路径 MTU 发现就是通过“撞墙”让路由器告诉你哪段太小。**

<br>

### 核心概念与方案

先把 PMTUD 换成你熟悉的场景：**PMTUD 就是“不许分片，撞到谁太小就让谁报 MTU，再慢慢调小”。**

路径 MTU 发现（PMTUD）的思路是：发送方把 IP 首部的**分片禁止位（DF）设为 1**，发较大包。途中路由器遇到超过 MTU 且不能分片的包，会丢弃并回 **ICMP 目标不可达，代码 4：需要分片但设置了不分片位**，同时可携带该链路的 MTU。发送方逐步减小包大小，最终找到能通过的 MTU。

关键点：**路由器不是默默丢弃**，而是用 ICMP 代码 4 告诉发送方“我这儿 MTU 只有这么大，你别发这么大还不让分片”。

IPv6 中间路由器默认不分片，更依赖 PMTUD；如果 ICMP 被防火墙拦截，可能出现“PMTUD 黑洞”：小包能过，大包一直失败。

<br>

### 最小实验：只看不同路径 MTU 下，最大不分片包大小怎么变

环境前提：同上。下面只做本地计算，不联网。

保存为 `pmtud_calc.cpp`：

````cpp
#include <iostream>

int main() {
    int ip_header = 20;
    int icmp_header = 8;

    for (int path_mtu : {1500, 1400, 1280}) {
        int max_icmp_payload = path_mtu - ip_header - icmp_header;
        std::cout << "路径 MTU = " << path_mtu
                  << "，最大 ICMP 数据 = " << max_icmp_payload
                  << " 字节，总包大小 = " << path_mtu << "\n";
    }

    std::cout << "\n发送超过路径 MTU 且不分片的包，路由器会回 ICMP 代码 4\n";
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 pmtud_calc.cpp -o pmtud_calc
./pmtud_calc
````

**这次只看一件事**：路径 MTU 从 1500 降到 1400、1280 时，最大 ICMP 数据是不是跟着变小。

预期输出：

````text
路径 MTU = 1500，最大 ICMP 数据 = 1472 字节，总包大小 = 1500
路径 MTU = 1400，最大 ICMP 数据 = 1372 字节，总包大小 = 1400
路径 MTU = 1280，最大 ICMP 数据 = 1252 字节，总包大小 = 1280

发送超过路径 MTU 且不分片的包，路由器会回 ICMP 代码 4
````

**关键观察点**：路径 MTU 越小，能通过的 ICMP 数据也越小。

**结论**：PMTUD 就是不断试探，撞到小 MTU 就调小，直到找到能通过的大小。

如果最大 ICMP 数据没有随 MTU 变化，先检查是否减去了 IP 头和 ICMP 头。

<br>

### 易错点与边界

坑一：**ICMP 被防火墙拦截会导致 PMTUD 失败**，发送方一直不知道包为什么被丢，形成黑洞。

坑二：**分片禁止位为 1 时路由器不能分片**，只能丢弃并回 ICMP；所以 DF 位是 PMTUD 的前提。

坑三：**路径 MTU 可能随路由变化**，不是探到一次就永久固定；网络路径变了要重新发现。

坑四：**ICMP 代码 4 是 IPv4 的说法**，IPv6 用 Packet Too Big 消息，机制类似但报文不同。

坑五：**PMTUD 依赖 ICMP 反馈**，如果中间设备把 ICMP 全禁了，只能靠应用层猜或保守设置 MSS。

<br>

### 一句话本质

**路径 MTU 发现就是“不许分片，撞到谁太小就让谁报 MTU，再慢慢调小”；ICMP 代码 4 是路由器告诉你“我这儿只能过这么大”。**

<br>

### 自测

ICMP 目标不可达代码 4 表示什么？如果 ICMP 被防火墙拦截，PMTUD 会怎样？

<details><summary>答案</summary>ICMP 目标不可达代码 4 表示“需要分片但设置了不分片位”，路由器遇到超 MTU 且不能分片的包时回这个错误，通常还会携带该链路的 MTU。如果 ICMP 被防火墙拦截，发送方收不到代码 4，不知道包为什么被丢，会一直发大包失败，形成 PMTUD 黑洞：小包能过，大包一直失败。</details>

<br>

<br>

## 完整可复现示例：用 traceroute 看路径，用 ping -M do 验证 PMTUD
***
你需要 Linux 或 WSL，已安装 `traceroute`（或 `tracert`）、`ping`。抓包需要 `sudo`。下面每个步骤只看一件事，不展开无关字段。

````bash
set -e

echo "=== 1. traceroute：看逐跳路径 ==="
if command -v traceroute >/dev/null 2>&1; then
  traceroute -n -m 10 example.com 2>/dev/null || true
else
  echo "traceroute 不可用，可改用 tracert 或 mtr"
fi

echo "=== 2. PMTUD：发不分片的大包，看是否被拒 ==="
ping -M do -s 1472 -c 1 example.com 2>&1 | head -n 5 || true
````

可能的输出：

````text
=== 1. traceroute ===
traceroute to example.com (93.184.216.34), 10 hops max
 1  192.168.1.1  1.234 ms  1.100 ms  1.050 ms
 2  * * *
 3  10.10.0.1  5.678 ms  5.432 ms  5.210 ms
 ...

=== 2. PMTUD ===
PING example.com (93.184.216.34) 1472(1500) bytes of data.
1480 bytes from ...: icmp_seq=1 ttl=... time=...
````

或者如果路径 MTU 小于 1500，可能看到：

````text
ping: local error: message too long, mtu=1400
````

**关键观察点**：第 1 步看每一跳有没有 IP 和延迟，`*` 表示该跳没回 ICMP；第 2 步看 1472 字节不分片包能不能通过。

验证结果：traceroute 逐跳显示路径；`ping -M do -s 1472` 发的是总长 1500 的不分片包，如果路径 MTU 更小，会收到“message too long”或 ICMP 代码 4 提示。

注意：`-M do` 和 `-s` 的参数因系统而异；`example.com` 可能不响应 ICMP，结果可能不完整；抓包能看到 ICMP 代码 4，但需要权限，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| traceroute 中间全是 `*` | 是否禁 ICMP、是否限速 | `traceroute -n`；抓包看 ICMP | 路由器可能不回 ICMP 超时，不代表那一跳不存在 |
| traceroute 到不了目标 | 目标是否禁 ICMP、防火墙 | `ping` 目标；`traceroute` 最大跳数 | 目标可能不回端口不可达，或防火墙拦截探测包 |
| 去程和回程路径不同 | 路径不对称 | 双向 traceroute；`mtr` | 去程和回程可能走不同路由器，traceroute 只反映探测方向 |
| 大包不通、小包正常 | 路径 MTU、PMTUD | `ping -M do -s 大小`；抓包看 ICMP 代码 4 | 路径 MTU 小于包大小，不分片时路由器回 ICMP 代码 4 |
| 看到 `message too long` | 本地 MTU 或路径 MTU | `ip addr` 看 MTU；`ping -M do` | 本地网卡 MTU 或路径 MTU 小于包大小 |
| PMTUD 黑洞 | ICMP 是否被拦截 | 抓包看有没有 ICMP 代码 4；查防火墙 | ICMP 被拦，发送方收不到 MTU 反馈，大包一直失败 |
| traceroute 很慢 | ICMP 限速、DNS 反查 | `traceroute -n` 不解析域名；看延迟 | 路由器可能限速 ICMP，或 DNS 反查拖慢 |
| IPv6 大包不通 | Packet Too Big 是否被拦 | `ping6 -M do -s 大小`；抓包看 ICMPv6 | IPv6 中间路由器默认不分片，更依赖 PMTUD |

<br>

<br>

## 常见误区
***
- **以为 traceroute 一次就能看到整条路径**：它是靠递增 TTL 逐跳探测，拼出来的。
- **以为中间 `*` 表示那一跳不存在**：可能是路由器禁 ICMP 或限速。
- **以为去程和回程路径一定相同**：路径可能不对称，traceroute 只反映探测方向。
- **把 TTL 超时和端口不可达混了**：超时是路由器回的，端口不可达是目标主机回的。
- **以为 MTU 是全程固定的**：路径上每段链路 MTU 可能不同，要 PMTUD 探测。
- **以为路由器遇到大包会默默丢弃**：DF 位为 1 时，路由器会回 ICMP 代码 4。
- **以为 PMTUD 一定成功**：ICMP 被防火墙拦截会形成黑洞，小包能过，大包一直失败。
- **把 IPv4 的 ICMP 代码 4 套到 IPv6**：IPv6 用 Packet Too Big，机制类似但报文不同。

<br>

<br>

## 自测三问
***
1. traceroute 利用 ICMP 哪类报文逐跳发现路由器？为什么中间会出现 `*`？
2. 怎么判断 UDP 探测包到达了目标主机？Windows `tracert` 和 Linux `traceroute` 默认探测方式有什么不同？
3. 路径 MTU 发现依赖哪个 ICMP 代码？如果 ICMP 被防火墙拦截，会出现什么问题？

<details><summary>参考答案</summary>1. traceroute 利用 ICMP 时间超时报文逐跳发现路由器：TTL 减到 0，路由器回 ICMP 超时，暴露该跳 IP。中间出现 `*` 表示该跳没有回 ICMP 超时，可能是路由器禁 ICMP、限速，或该跳不回探测包，不代表那一跳不存在。2. Linux traceroute 默认用 UDP 高端口探测，目标主机回 ICMP 端口不可达表示到达；Windows tracert 默认用 ICMP Echo，目标回 Echo Reply 表示到达。3. 路径 MTU 发现依赖 ICMP 目标不可达代码 4（需要分片但设置了不分片位），路由器遇到超 MTU 且不能分片的包时回这个错误。如果 ICMP 被防火墙拦截，发送方收不到代码 4，不知道包为什么被丢，会形成 PMTUD 黑洞：小包能过，大包一直失败。</details>

<br>

<br>

## 总结
***
**traceroute 用递增 TTL 让每跳路由器回 ICMP 超时，目标回端口不可达结束；路径 MTU 发现用“不分片”让路由器回 ICMP 代码 4，逐步调小包大小找到可通行 MTU；中间 `*` 不一定是故障，ICMP 被拦会导致 PMTUD 黑洞。**

<br>

<br>

## 参考资料
***
- RFC 792：Internet Control Message Protocol
- RFC 1191：Path MTU Discovery
- RFC 1981：Path MTU Discovery for IP version 6
- RFC 1812：Requirements for IP Version 4 Routers
- `man 8 traceroute`、`man 8 ping`、`man 8 tcpdump`
- 需按环境核实：`traceroute`、`tracert`、`ping -M do` 的参数和可用性因系统而异；中间路由器是否回 ICMP 取决于配置和限速策略；PMTUD 依赖 ICMP 不被防火墙拦截；IPv6 的 Packet Too Big 与 IPv4 的 ICMP 代码 4 报文不同。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>