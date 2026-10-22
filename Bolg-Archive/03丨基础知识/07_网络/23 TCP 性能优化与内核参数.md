# 23 TCP 性能优化与内核参数：握手/挥手/数据传输优化、TFO、缓冲区、TIME_WAIT
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
跨机房传大文件，带宽明明有 1Gbps，`scp` 却只跑出几十 MB/s。

你可能会想：带宽不够？加带宽呗。

但加了带宽，吞吐可能还是上不去。**瓶颈可能不在带宽，而在 TCP 窗口和缓冲区。** 带宽时延积（BDP）是 100MB，发送缓冲区只有 16KB，窗口上不去，带宽根本用不满。

再想两个场景：

- 压测服务时，客户端 `connect` 一直超时，服务端 `ss -lnt` 看到 `Send-Q` 满了，是哪个队列的问题？
- 短连接服务跑一段时间，`ss -tan` 里一堆 `TIME_WAIT`，能不能直接调小？

这一章咱们用一条问题链往前走：

1. 建连和断连哪里会慢，半连接队列、全连接队列、TIME_WAIT 分别该看哪些参数？
2. 带宽很大但传不快，带宽时延积和缓冲区是什么关系？
3. 为什么程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能反而坏事？

<br>

<br>

## 优先级说明
***
这篇是 TCP 性能调优的入门，面试常问，项目里排查连接超时、吞吐上不去也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- 半连接队列受 `tcp_max_syn_backlog`、`somaxconn`、应用 `listen` 的 `backlog` 共同影响
- 全连接队列满默认丢 ACK，`tcp_abort_on_overflow=1` 才回 RST
- `tcp_tw_reuse` 只对客户端 `connect` 有效，服务端开它不会复用服务端主动关闭的 TIME_WAIT
- `tcp_tw_recycle` 在 NAT 下不安全，Linux 4.12 后已移除
- TIME_WAIT 不是越少越好，它是正常关闭流程的一部分
- 带宽时延积 BDP = 带宽 × RTT，缓冲区太小窗口上不去，带宽用不满
- `tcp_wmem`/`tcp_rmem` 单位是字节，`tcp_mem` 单位是页面
- 程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能关闭动态调节
- TFO 能在第二次以后连接省 1 RTT，但需双方支持

**了解即可（知道大概，不展开）：**
- `tcp_orphan_retries`、`tcp_fin_timeout` 的具体行为
- `tcp_moderate_rcvbuf` 的动态调节细节
- 不同内核版本的初始 `cwnd` 和默认参数差异
- 容器环境下 sysctl 的 namespace 隔离

**主线**：握手少重试、扩队列、防攻击 → 挥手控 FIN 重传、谨慎管理 TIME_WAIT → 数据传输按 BDP 调缓冲区 → 程序里别乱设 `SO_SNDBUF`/`SO_RCVBUF`。

<br>

<br>

## 知识点 1：三次握手与四次挥手优化——少重试、扩队列、管 TIME_WAIT
***
### 问题

建连和断连哪里会慢？客户端 SYN 重传、服务端 SYN+ACK 重传、半连接队列、全连接队列、TIME_WAIT，分别该看哪些参数？TFO 又省了哪一段？

**你可能会想**：`connect` 超时，肯定是网络丢包吧？

不一定。**服务端内核可能收到了 SYN，但半连接队列满了，直接丢掉了，根本没回 SYN+ACK。** 客户端看到的就是 `connect` 超时，和网络丢包的现象一模一样。

<br>

### 核心概念与方案

先把握手挥手换成你熟悉的场景：**握手是前台排队，半连接队列是“正在填表的人”，全连接队列是“填完表等叫号的人”；挥手是送客，TIME_WAIT 是客人走了但座位还要留一会儿。**

**三次握手**：

- 客户端 SYN 重传次数看 `tcp_syn_retries`；服务端 SYN+ACK 重传看 `tcp_synack_retries`。
- 半连接队列受 `tcp_max_syn_backlog`、`somaxconn`、应用 `listen` 的 `backlog` 共同影响；SYN 攻击可开 `tcp_syncookies=1` 应急。
- 全连接队列满默认丢 ACK，`tcp_abort_on_overflow=1` 回 RST；通常设 0 更利于突发流量，让客户端重试而不是直接失败。
- **TCP Fast Open（TFO）** 可在第二次以后连接绕过三次握手携带数据，减少 1 RTT，但需双方支持，且首次连接仍需正常握手。

**四次挥手**：

- 主动方 FIN 重传看 `tcp_orphan_retries`；`close` 后的 FIN_WAIT2 受 `tcp_fin_timeout` 限制。
- TIME_WAIT 可用 `tcp_max_tw_buckets` 限制数量；客户端可用 `tcp_tw_reuse=1` + `tcp_timestamps=1` 复用超过 1 秒的 TIME_WAIT 端口。
- `tcp_tw_recycle` 在 NAT 下不安全，Linux 4.12 后已取消，现代内核不用再调。

一句话：**握手优化是少重试、扩队列、防攻击、能 TFO；挥手优化是控 FIN 重传、管 TIME_WAIT、慎用暴力复用。**

<br>

### 最小实验：只看 ss -lnt 的 Send-Q 是不是接近上限

环境前提：Linux；已安装 `ss`（`iproute2` 包）。下面只读参数和队列，不改配置。

````bash
sysctl net.ipv4.tcp_syn_retries net.ipv4.tcp_synack_retries 2>/dev/null || true
ss -lnt
````

**这次只看一件事**：`ss -lnt` 输出里 `Send-Q` 那一列的数字，和 `tcp_max_syn_backlog`/`somaxconn` 相比是不是接近上限。

可能看到类似：

````text
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 5

State  Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 0      128    0.0.0.0:22         0.0.0.0:*
````

`Send-Q` 是 128，说明这个监听套接字的全连接队列上限是 128。**如果 `Send-Q` 接近上限，且压测时连接超时，就要怀疑全连接队列满了。**

不同内核版本、不同发行版输出可能不同，`tcp_max_syn_backlog` 和 `somaxconn` 的默认值也不同，**需按环境核实**。如果 `sysctl` 报错，可能是参数不存在或容器环境隔离。

<br>

### 易错点与边界

坑一：**`tcp_tw_reuse` 只对客户端 `connect` 有效**，服务端开它不会复用服务端主动关闭的 TIME_WAIT。

坑二：**`tcp_tw_recycle` 在 NAT 下会误杀 SYN**，Linux 4.12 后已移除，别再从旧文章里抄这个参数。

坑三：**TIME_WAIT 不是越少越好**，它是正常关闭流程的一部分；盲目调小可能让旧报文干扰新连接。

坑四：**全连接队列满默认丢 ACK，不回 RST**；开 `tcp_abort_on_overflow` 回 RST 会让客户端更快失败，不一定更好。

坑五：**TFO 不是所有客户端和服务端都支持**，中间设备也可能干扰；不能把它当成万能加速。

<br>

### 一句话本质

**握手优化是少重试、扩队列、防攻击、能 TFO；挥手优化是控 FIN 重传、管 TIME_WAIT、慎用暴力复用；TIME_WAIT 不是越少越好，`tcp_tw_reuse` 只对客户端有效。**

<br>

### 自测

为什么 `tcp_tw_reuse` 只适合客户端？TIME_WAIT 很多时，为什么不能无脑调小 `tcp_max_tw_buckets`？

<details><summary>答案</summary>`tcp_tw_reuse` 只对客户端主动 `connect` 有效，因为客户端端口是临时分配的，复用旧端口风险相对可控；服务端端口是固定的，复用 TIME_WAIT 端口可能让旧连接的延迟报文串到新连接。`tcp_max_tw_buckets` 限制 TIME_WAIT 数量，调太小会让 TIME_WAIT 被提前清理，可能让旧报文干扰新连接，破坏 TIME_WAIT 防旧报文的作用。所以不能无脑调小，要结合业务和风险权衡。</details>

<br>

<br>

## 知识点 2：数据传输优化与缓冲区——带宽时延积决定窗口上限
***
### 问题

带宽很大但 TCP 传不快，为什么？带宽时延积和缓冲区是什么关系？`tcp_wmem`、`tcp_rmem`、`tcp_mem` 分别管什么？程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 为什么可能反而坏事？

**你可能会想**：带宽 1Gbps，缓冲区设大一点不就行了？

不是越大越好。**缓冲区要围绕带宽时延积调：太小窗口上不去，带宽用不满；太大又可能增加排队延迟和内存压力，甚至掩盖拥塞。**

<br>

### 核心概念与方案

先把缓冲区换成你熟悉的场景：**带宽时延积是“路上同时在跑多少货”，缓冲区是“仓库能暂存多少货”。仓库太小，路上跑不满；仓库太大，货堆着，延迟反而高。**

**带宽时延积 BDP = 带宽 × RTT**，表示网络中“飞行中”的字节数。发送缓冲区太小，窗口上不去，带宽用不满；太大又可能过载丢包。

- `tcp_wmem` 配发送缓冲区 min/default/max，单位字节。
- `tcp_rmem` 配接收缓冲区，单位字节。
- `tcp_mem` 是 TCP 总内存页面数，单位是页面，不是字节。

接收缓冲区动态调节需 `tcp_moderate_rcvbuf=1`；窗口扩大 `tcp_window_scaling=1` 可让窗口超过 64KB。

程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能关闭动态调节，通常不建议；让内核按 BDP 和内存压力动态调通常更稳。

<br>

### 最小实验：只看 BDP 和缓冲区谁更大

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 且支持 C++17。下面只做本地计算，不联网。

保存为 `bdp.cpp`：

````cpp
#include <iomanip>
#include <iostream>

int main() {
    double bandwidth_mbps = 1000.0;   // 1 Gbps
    double rtt_ms = 10.0;             // 10 ms
    double buf_kb = 16.0;             // 发送缓冲区 16 KB

    double bandwidth_bytes_per_sec = bandwidth_mbps * 1e6 / 8.0;
    double rtt_sec = rtt_ms / 1000.0;
    double bdp_bytes = bandwidth_bytes_per_sec * rtt_sec;
    double buf_bytes = buf_kb * 1024.0;

    std::cout << std::fixed << std::setprecision(2);
    std::cout << "BDP = " << bdp_bytes / 1024.0 << " KB\n";
    std::cout << "发送缓冲区 = " << buf_kb << " KB\n";

    if (buf_bytes < bdp_bytes) {
        std::cout << "缓冲区小于 BDP，窗口上不去，带宽用不满\n";
    } else {
        std::cout << "缓冲区不小于 BDP，窗口有机会跑满\n";
    }

    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 bdp.cpp -o bdp
./bdp
````

**这次只看一件事**：缓冲区是不是小于 BDP。

预期输出：

````text
BDP = 1220.70 KB
发送缓冲区 = 16.00 KB
缓冲区小于 BDP，窗口上不去，带宽用不满
````

BDP 约 1.2MB，发送缓冲区只有 16KB，**窗口上不去，带宽根本用不满**。

如果把缓冲区改成 2MB，再跑一次：

````text
BDP = 1220.70 KB
发送缓冲区 = 2048.00 KB
缓冲区不小于 BDP，窗口有机会跑满
````

**结论**：高带宽高延迟网络需要更大的窗口和缓冲区，但不是越大越好，要结合 BDP 和内存成本。

<br>

### 易错点与边界

坑一：**`tcp_wmem`/`tcp_rmem` 单位是字节，`tcp_mem` 单位是页面**，别把三者单位混了。

坑二：**不是缓冲区越大越好**，太大可能增加排队延迟和内存压力，甚至掩盖拥塞。

坑三：**带宽时延积是设置上限的重要参考**，不是唯一依据；还要看应用读写速度、并发连接数。

坑四：**程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能关闭动态调节**，尤其在高并发服务里，可能让内核没法按压力自适应。

坑五：**窗口扩大 `tcp_window_scaling=1` 是协商选项**，不是单方面开了就一定能用大窗口，还要对端支持。

<br>

### 一句话本质

**缓冲区要围绕带宽时延积调：太小跑不满，太大易过载；`tcp_wmem`/`tcp_rmem` 单位是字节，`tcp_mem` 单位是页面；程序里别乱设 `SO_SNDBUF`/`SO_RCVBUF`，交给内核动态调通常更稳。**

<br>

### 自测

为什么高带宽高延迟网络需要更大的窗口和缓冲区？如果缓冲区远小于 BDP，会发生什么？

<details><summary>答案</summary>高带宽高延迟网络下，BDP = 带宽 × RTT 很大，表示网络中“飞行中”的字节数很多。发送缓冲区太小，窗口上不去，发送方不能连续发足够多的数据，带宽用不满。缓冲区远小于 BDP 时，发送方大部分时间在等 ACK，链路空闲，吞吐上不去。但缓冲区也不是越大越好，太大可能增加排队延迟和内存压力，要结合 BDP 和内存成本权衡。</details>

<br>

<br>

## 完整可复现示例：查看 TCP 优化参数并计算 BDP
***
你需要 Linux 或 WSL，已安装 `sysctl`、`ss`、`g++`。下面每个步骤只看一件事，不展开无关细节。

````bash
set -e

echo "=== 1. 握手/挥手参数 ==="
sysctl net.ipv4.tcp_syn_retries net.ipv4.tcp_synack_retries \
       net.ipv4.tcp_tw_reuse net.ipv4.tcp_timestamps 2>/dev/null || true

echo "=== 2. 缓冲区参数 ==="
sysctl net.ipv4.tcp_wmem net.ipv4.tcp_rmem net.ipv4.tcp_mem 2>/dev/null || true

echo "=== 3. 监听队列 ==="
ss -lnt | head || true

echo "=== 4. BDP 与缓冲区对比 ==="
g++ -std=c++17 -O2 bdp.cpp -o bdp
./bdp
````

可能的输出：

````text
=== 1. 握手/挥手参数 ===
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 5
net.ipv4.tcp_tw_reuse = 2
net.ipv4.tcp_timestamps = 1

=== 2. 缓冲区参数 ===
net.ipv4.tcp_wmem = 4096 16384 4194304
net.ipv4.tcp_rmem = 4096 87380 6291456
net.ipv4.tcp_mem = 94677 126238 189354

=== 3. 监听队列 ===
State  Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 0      128    0.0.0.0:22         0.0.0.0:*

=== 4. BDP 与缓冲区对比 ===
BDP = 1220.70 KB
发送缓冲区 = 16.00 KB
缓冲区小于 BDP，窗口上不去，带宽用不满
````

**关键观察点**：第 3 步看 `Send-Q` 是不是接近上限；第 4 步看缓冲区是不是小于 BDP。

验证结果：优化 TCP 要分握手、挥手、数据传输三阶段对症下药，不能只背一个参数。`ss -lnt` 的 `Recv-Q`/`Send-Q` 接近上限时，要怀疑队列溢出；`tcp_wmem`/`tcp_rmem` 是三元组，分别是最小、默认、最大值；缓冲区小于 BDP 时，高带宽高延迟网络的吞吐会被窗口限制。

注意：`sysctl` 某些参数在旧内核或容器环境里可能不存在，加了 `|| true` 避免脚本中断；容器里看到的可能是宿主机参数或被 namespace 隔离后的值，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| `connect` 超时，服务端像没收到 SYN | 半连接队列、防火墙 | `netstat -s \| grep -i "SYNs to LISTEN"`；`ss -lnt` | 半连接队列满会丢 SYN；防火墙也可能拦 SYN |
| 客户端 `connect` 成功但很快被 RST | 全连接队列、`tcp_abort_on_overflow` | `netstat -s \| grep overflowed`；`ss -lnt` 看 `Send-Q` | 全连接队列满默认丢 ACK，开 `tcp_abort_on_overflow=1` 会回 RST |
| 大量 `TIME_WAIT` | 是否短连接多、主动关闭方集中 | `ss -tan \| grep TIME-WAIT \| wc -l` | 主动关闭方多、短连接多时 TIME_WAIT 会多；不一定是 bug |
| TIME_WAIT 端口无法复用 | 是否客户端、`tcp_tw_reuse` | `sysctl net.ipv4.tcp_tw_reuse`；`ss -tan` | `tcp_tw_reuse` 只对客户端 `connect` 有效，服务端开它没用 |
| 带宽很大但吞吐上不去 | 缓冲区是否小于 BDP | `sysctl net.ipv4.tcp_wmem`；计算 BDP | 缓冲区太小，窗口上不去，带宽用不满 |
| 延迟高、吞吐低 | 是否缓冲膨胀 | 看 RTT 和 cwnd；查拥塞控制算法 | 缓冲区太大可能增加排队延迟；不同算法在不同网络下表现不同 |
| 程序设了 `SO_SNDBUF` 反而慢 | 是否关闭动态调节 | 查代码；`ss -ti` 看窗口 | 直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能关闭内核动态调节 |
| TFO 没生效 | 双方是否支持、中间设备 | 抓包看 TFO 选项；查内核参数 | TFO 需要双方支持，中间设备也可能干扰 |

<br>

<br>

## 常见误区
***
- **以为 `connect` 超时一定是网络丢包**：可能是服务端半连接队列满，内核直接丢 SYN。
- **以为 `tcp_tw_reuse` 对服务端也有效**：它只对客户端 `connect` 有效。
- **以为 TIME_WAIT 越少越好**：TIME_WAIT 是正常关闭流程的一部分，盲目调小可能让旧报文干扰新连接。
- **以为缓冲区越大越好**：太大可能增加排队延迟和内存压力，要围绕 BDP 调。
- **把 `tcp_wmem`/`tcp_rmem` 和 `tcp_mem` 单位混了**：前两者是字节，后者是页面。
- **程序里无脑设 `SO_SNDBUF`/`SO_RCVBUF`**：可能关闭动态调节，反而影响性能。
- **以为 TFO 是万能加速**：需要双方支持，中间设备也可能干扰。

<br>

<br>

## 自测三问
***
1. 半连接队列和全连接队列分别受哪些参数影响？`ss -lnt` 里 `Send-Q` 满了，更可能是哪个队列的问题？
2. `tcp_tw_reuse` 为什么只适合客户端？TIME_WAIT 很多时，为什么不能无脑调小 `tcp_max_tw_buckets`？
3. 带宽时延积和 TCP 缓冲区有什么关系？为什么程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能反而坏事？

<details><summary>参考答案</summary>1. 半连接队列受 `tcp_max_syn_backlog`、`somaxconn`、应用 `listen` 的 `backlog` 共同影响；全连接队列主要受 `somaxconn` 和应用 `listen` 的 `backlog` 影响。`ss -lnt` 的 `Send-Q` 接近上限，更可能是全连接队列满了，因为 `Send-Q` 显示的是监听套接字已建立但未被 `accept` 取走的连接数上限。2. `tcp_tw_reuse` 只对客户端 `connect` 有效，因为客户端端口临时分配，复用旧端口风险相对可控；服务端端口固定，复用 TIME_WAIT 端口可能让旧连接延迟报文串到新连接。`tcp_max_tw_buckets` 调太小会提前清理 TIME_WAIT，破坏防旧报文的作用，所以不能无脑调小。3. BDP = 带宽 × RTT，表示网络中“飞行中”的字节数；缓冲区太小，窗口上不去，带宽用不满。程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能关闭内核动态调节，让内核没法按 BDP 和内存压力自适应，高并发时反而影响性能。</details>

<br>

<br>

## 总结
***
**TCP 优化分三段：握手少重试扩队列防攻击，挥手控 FIN 重传并谨慎管理 TIME_WAIT，数据传输按带宽时延积调缓冲区；`tcp_tw_reuse` 只对客户端有效，TIME_WAIT 不是越少越好，缓冲区不是越大越好，程序里别乱设 `SO_SNDBUF`/`SO_RCVBUF`。**

<br>

<br>

## 参考资料
***
- RFC 9293：Transmission Control Protocol
- RFC 7413：TCP Fast Open
- RFC 7323：TCP Extensions for High Performance
- `man 7 tcp`、`man 7 socket`、`man 2 listen`
- 需按环境核实：`tcp_syn_retries`、`tcp_synack_retries`、`tcp_tw_reuse`、`tcp_wmem`、`tcp_rmem`、`tcp_mem` 的默认值因内核版本和发行版而异；`tcp_tw_recycle` 在 Linux 4.12 后已移除；容器环境下 sysctl 可能被 namespace 隔离；`ss -lnt` 输出字段因系统而异；TFO 支持取决于客户端、服务端和中间设备。
<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>