# 22 TCP 拥塞控制：cwnd、慢启动、拥塞避免、快速恢复、AIMD
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
上一章讲的流量控制，是接收方喊“我兜里还能装多少”，发送方按这个量发。

但你想过没有：**接收方说还能装，不代表中间的路由器还能扛。** 你发得太猛，中间路由器队列溢出，包照样丢。

你可能会想：TCP 又看不见路由器，它怎么知道网络堵了？

更具体一点：**为什么 TCP 刚开始要小心翼翼地涨，丢包后又突然刹车？同样是丢包，超时重传和收到 3 个重复 ACK，为什么反应不一样？**

这一章咱们用一条问题链往前走：

1. TCP 看不见路由器，靠什么判断拥塞？
2. 发送方到底能发多快，`cwnd` 和 `rwnd` 谁说了算？
3. 慢启动为什么叫“慢”但涨得很快，到 `ssthresh` 后为什么换策略？
4. 超时重传和快速重传，为什么一个回到 1，一个只减半？
5. AIMD 的“加法增大、乘法减小”到底在保证什么？

<br>

<br>

## 优先级说明
***
这篇是 TCP 拥塞控制的核心，面试必问，项目里排查吞吐上不去、重传多也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- `cwnd` 是拥塞窗口，保护网络；`rwnd` 是接收窗口，保护接收方；实际发送窗口取 `min(cwnd, rwnd)`
- 发送速率大致是 `cwnd / RTT`
- TCP 主要靠丢包判断拥塞：超时重传或 3 个重复 ACK
- 慢启动：起点低，按 RTT 轮次指数增长，到 `ssthresh` 切换
- 拥塞避免：每个 RTT 约加 1 MSS，线性增长
- 超时重传：`ssthresh = cwnd/2`，`cwnd = 1`，重回慢启动
- 快速恢复：`ssthresh = cwnd/2`，`cwnd = ssthresh + 3`，不回 1
- AIMD：加法增大、乘法减小，多流竞争时趋向公平

**了解即可（知道大概，不展开）：**
- Reno、CUBIC、BBR 等具体算法差异
- SACK 对拥塞恢复的影响
- 延迟确认对 cwnd 增长节奏的影响
- 不同内核版本的初始 `cwnd` 值

**主线**：流量控制管接收方，拥塞控制管网络 → `cwnd` 是踩油门的力度，丢包是刹车信号 → 慢启动探路，拥塞避免稳住 → 超时重罚回 1，快速恢复只减半 → AIMD 保证收敛和公平。

<br>

<br>

## 知识点 1：拥塞窗口与拥塞判断——cwnd 是油门，丢包是刹车
***
### 问题

TCP 看不见路由器，怎么判断拥塞？发送方到底能发多快？为什么说 `cwnd` 是“踩油门的力度”，丢包是“刹车信号”？

**你可能会想**：接收方说还能收，那就一直发呗，网络堵不堵关我什么事？

不行。**接收方说能收，不代表中间路由器能扛。** 你发得太猛，路由器队列溢出，包丢了，重传反而更慢。TCP 必须自己判断网络堵没堵。

<br>

### 核心概念与方案

先把 `cwnd` 换成你熟悉的场景：**`cwnd` 是踩油门的力度，丢包是刹车信号。**

发送方维护**拥塞窗口 `cwnd`**，实际发送窗口取 `min(cwnd, rwnd)`：

- `cwnd` 保护网络，是发送方自己维护的。
- `rwnd` 保护接收方，是接收方通告的。

发送速率大致是 `cwnd / RTT`。

TCP 主要靠**丢包**判断拥塞：**超时重传**或**收到 3 个重复 ACK**。没有拥塞就增大 `cwnd`，出现拥塞就减小 `cwnd`。

注意：丢包不一定是网络真的坏了，也可能是链路误码、无线干扰；但 TCP 通常把丢包当作拥塞信号来处理，这是它的设计假设。

<br>

### 最小实验：只看 cwnd 和 rwnd 谁更小，实际窗口就取谁

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 且支持 C++17。下面只做本地模拟，不联网。

保存为 `window_min.cpp`：

````cpp
#include <algorithm>
#include <iostream>

int main() {
    int cwnd = 10;   // 拥塞窗口，单位 MSS
    int rwnd = 4;    // 接收窗口，单位 MSS

    int effective = std::min(cwnd, rwnd);
    std::cout << "cwnd=" << cwnd << ", rwnd=" << rwnd
              << ", 实际发送窗口=" << effective << " MSS\n";

    rwnd = 20;
    effective = std::min(cwnd, rwnd);
    std::cout << "cwnd=" << cwnd << ", rwnd=" << rwnd
              << ", 实际发送窗口=" << effective << " MSS\n";

    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 window_min.cpp -o window_min
./window_min
````

**这次只看一件事**：实际发送窗口是不是取 `cwnd` 和 `rwnd` 中较小的那个。

预期输出：

````text
cwnd=10, rwnd=4, 实际发送窗口=4 MSS
cwnd=10, rwnd=20, 实际发送窗口=10 MSS
````

第一次 `rwnd` 更小，实际窗口是 4；第二次 `cwnd` 更小，实际窗口是 10。**实际发送窗口永远取两者较小值。**

**结论**：`cwnd` 再大，`rwnd=0` 也发不出去；`rwnd` 再大，`cwnd` 小也发不快。

<br>

### 易错点与边界

坑一：**实际发送窗口受 `cwnd` 和 `rwnd` 共同限制**，取两者较小值，不是只看 `cwnd`。

坑二：**丢包不一定是网络坏了**，但 TCP 通常把它当拥塞信号；无线误码场景下这可能误判。

坑三：**`cwnd` 是发送方维护的，不是接收方通告的**；接收方通告的是 `rwnd`。

坑四：**发送速率约等于 `cwnd/RTT`**，是近似值，实际还受 ACK 节奏、延迟确认等影响。

<br>

### 一句话本质

**cwnd 是 TCP 踩油门的力度，丢包是刹车信号；实际发送窗口取 `min(cwnd, rwnd)`，一个保护网络，一个保护接收方。**

<br>

### 自测

TCP 发送窗口由哪两个窗口共同决定？如果 `cwnd` 很大但 `rwnd=0`，发送方还能继续发吗？

<details><summary>答案</summary>发送窗口由 `cwnd` 和 `rwnd` 共同决定，实际取 `min(cwnd, rwnd)`。`cwnd` 保护网络，`rwnd` 保护接收方。如果 `rwnd=0`，即使 `cwnd` 很大，发送方也不能继续发，因为接收方缓存满了；发送方会启动窗口探测，等接收方窗口恢复。</details>

<br>

<br>

## 知识点 2：慢启动、拥塞避免、快速恢复——先探路，再稳住，丢包分轻重
***
### 问题

TCP 怎么从 1 个 MSS 探到网络容量，又怎么在丢包后恢复？为什么超时重传要回到 1，而 3 个重复 ACK 只减半？AIMD 又是什么意思？

**你可能会想**：丢包了就减半，不是挺简单吗？为什么还要分超时和快速重传？

因为**超时和 3 个重复 ACK 代表的严重程度不同**。超时说明 ACK 很久没回来，可能网络堵得很厉害；3 个重复 ACK 说明接收方还在收包，只是中间缺了一个，网络可能没那么糟。所以一个重罚回 1，一个只减半。

<br>

### 核心概念与方案

先把拥塞控制换成你熟悉的场景：**老司机开车，先慢慢试探，摸清路况再加速；前面出事了，轻点刹车还是重踩，看情况。**

TCP 拥塞控制分几个阶段：

- **慢启动**：初始 `cwnd` 小，每收到一个 ACK `cwnd` 加 1，效果是按 RTT 轮次指数增长（1→2→4→8）。到 `ssthresh` 后进入拥塞避免。叫“慢启动”是因为起点低，不是增长慢。
- **拥塞避免**：每个 RTT `cwnd` 约加 1 MSS，线性增长，小心试探。
- **超时重传**：认为严重拥塞，`ssthresh = cwnd/2`，`cwnd = 1`，重回慢启动。
- **快速重传/快速恢复**：收到 3 个重复 ACK，认为轻微丢包；`ssthresh = cwnd/2`，`cwnd = ssthresh + 3`，重传丢失包，之后进入拥塞避免，不必回到 1。

**AIMD**：加法增大（Additive Increase）、乘法减小（Multiplicative Decrease）。多流竞争时，守规矩的 TCP 流可以逐渐收敛到较公平的带宽分配。

<br>

### 最小实验：只看超时和快速重传后 cwnd 变化有什么不同

环境前提：同上。下面只做本地模拟，不联网。

保存为 `congestion_response.cpp`：

````cpp
#include <algorithm>
#include <iostream>

void timeout(int& cwnd, int& ssthresh) {
    ssthresh = std::max(cwnd / 2, 2);
    cwnd = 1;
    std::cout << "超时重传：ssthresh=" << ssthresh
              << ", cwnd=" << cwnd << "\n";
}

void fast_recovery(int& cwnd, int& ssthresh) {
    ssthresh = std::max(cwnd / 2, 2);
    cwnd = ssthresh + 3;
    std::cout << "快速恢复：ssthresh=" << ssthresh
              << ", cwnd=" << cwnd << "\n";
}

int main() {
    int cwnd = 16;
    int ssthresh = 32;

    std::cout << "初始 cwnd=" << cwnd << "\n";
    timeout(cwnd, ssthresh);
    std::cout << "超时后 cwnd=" << cwnd << "\n\n";

    cwnd = 16;
    ssthresh = 32;
    std::cout << "初始 cwnd=" << cwnd << "\n";
    fast_recovery(cwnd, ssthresh);
    std::cout << "快速恢复后 cwnd=" << cwnd << "\n";

    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 congestion_response.cpp -o congestion_response
./congestion_response
````

**这次只看一件事**：超时后 `cwnd` 是不是 1，快速恢复后 `cwnd` 是不是没有回到 1。

预期输出：

````text
初始 cwnd=16
超时重传：ssthresh=8, cwnd=1
超时后 cwnd=1

初始 cwnd=16
快速恢复：ssthresh=8, cwnd=11
快速恢复后 cwnd=11
````

**关键观察点**：超时后 `cwnd=1`，快速恢复后 `cwnd=11`。

两者都把 `ssthresh` 减半到 8，但超时直接回 1，快速恢复是 `8+3=11`。**超时是重罚，快速恢复是轻罚。**

如果两者 `cwnd` 变化一样，说明模拟逻辑写错了。

<br>

### 易错点与边界

坑一：**超时和 3 个重复 ACK 处理不同**；超时回 `cwnd=1`，快速恢复只减半，别混成一套。

坑二：**快速恢复不是“不减速”**，而是避免断崖式回 1，仍然把 `ssthresh` 减半。

坑三：**AIMD 公平性只在守规矩的 TCP 流之间成立**，UDP 或多连接会抢占带宽。

坑四：**慢启动的“慢”是起点低**，按 RTT 看是指数增长，不是线性慢慢加。

坑五：**`ssthresh` 是阈值，不是固定值**，每次拥塞事件都会更新。

<br>

### 一句话本质

**慢启动先探路，拥塞避免稳住，超时重罚回起点，快速恢复只减半；AIMD 用加法增大、乘法减小，在不让网络崩溃的前提下尽量多拉快跑。**

<br>

### 自测

超时重传和快速重传后，`cwnd` 变化有什么不同？为什么快速恢复不直接回到 1？

<details><summary>答案</summary>超时重传把 `ssthresh` 设为 `cwnd/2`，`cwnd` 回到 1，重回慢启动；快速恢复把 `ssthresh` 设为 `cwnd/2`，`cwnd` 设为 `ssthresh+3`，不回到 1。快速恢复不直接回 1，是因为 3 个重复 ACK 说明接收方还在收包，只是缺了一个，网络可能没那么严重；直接回 1 会断崖式降速，浪费带宽。</details>

<br>

<br>

## 完整可复现示例：用 C++ 模拟 AIMD 与快速恢复
***
你需要 Linux/macOS 或 WSL，已安装 `g++`。下面把慢启动、拥塞避免和一次快速恢复串起来，每个步骤只看一个观察点。

````bash
g++ -std=c++17 -O2 tcp_aimd.cpp -o tcp_aimd
./tcp_aimd
````

````cpp
#include <algorithm>
#include <iostream>

int main() {
    int cwnd = 1;
    int ssthresh = 8;

    for (int r = 1; r <= 11; ++r) {
        if (cwnd < ssthresh) {
            cwnd *= 2;
            std::cout << "R" << r << " 慢启动 cwnd=" << cwnd << "\n";
        } else {
            cwnd += 1;
            std::cout << "R" << r << " 拥塞避免 cwnd=" << cwnd << "\n";
        }

        if (r == 8) {
            ssthresh = std::max(cwnd / 2, 2);
            cwnd = ssthresh + 3;
            std::cout << "  3 个重复 ACK：ssthresh=" << ssthresh
                      << ", cwnd=" << cwnd << "\n";
        }
    }

    return 0;
}
````

预期输出：

````text
R1 慢启动 cwnd=2
R2 慢启动 cwnd=4
R3 慢启动 cwnd=8
R4 拥塞避免 cwnd=9
R5 拥塞避免 cwnd=10
R6 拥塞避免 cwnd=11
R7 拥塞避免 cwnd=12
R8 拥塞避免 cwnd=13
  3 个重复 ACK：ssthresh=6, cwnd=9
R9 拥塞避免 cwnd=10
R10 拥塞避免 cwnd=11
R11 拥塞避免 cwnd=12
````

**关键观察点**：慢启动阶段 `cwnd` 是不是按轮次翻倍；第 8 轮快速恢复后 `cwnd` 是不是没有回到 1。

验证结果：慢启动 1→2→4→8，到 `ssthresh=8` 后进入拥塞避免，线性增长；第 8 轮快速恢复，`ssthresh` 减半为 6，`cwnd` 变为 `6+3=9`，没有回到 1。

注意：这个示例是教学简化模型，不是完整 TCP 实现；真实 TCP 的 `cwnd` 按 ACK 增长，还受延迟确认、SACK、拥塞控制算法（Reno、CUBIC 等）影响，快速恢复的 `+3` 也是经典 Reno 的近似行为，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| 吞吐上不去，重传多 | 是否频繁丢包、cwnd 是否被压小 | `ss -ti` 看 cwnd、retrans；抓包看丢包 | 丢包会触发拥塞控制，cwnd 减小，吞吐下降 |
| 刚开始传输很慢，后来变快 | 是否处于慢启动 | `ss -ti` 看 cwnd 变化 | 慢启动起点低，按 RTT 指数增长，前期慢是正常的 |
| 大量超时重传 | 是否严重拥塞、RTO 是否过小 | 抓包看超时和重传；看 RTO | 超时会把 cwnd 回到 1，吞吐骤降；RTO 过小会误重传 |
| 重复 ACK 很多但没超时 | 是否触发快速恢复 | 抓包看重复 ACK；`ss -ti` 看 cwnd | 3 个重复 ACK 触发快速恢复，cwnd 减半但不回 1 |
| 多流抢带宽不公平 | 是否有 UDP 或多连接抢占 | 抓包看各流速率；看应用行为 | AIMD 公平性只在守规矩的 TCP 流之间成立，UDP 不遵守 |
| cwnd 一直很小 | 是否丢包持续、rwnd 是否受限 | `ss -ti` 看 cwnd/rwnd；抓包 | cwnd 受丢包影响，rwnd 受接收方缓存影响，两者取较小值 |
| 延迟高、吞吐低 | 是否缓冲膨胀、算法不适配 | 看 RTT 和 cwnd；查拥塞控制算法 | 缓冲膨胀会让 RTT 变大；不同算法在不同网络下表现不同 |

<br>

<br>

## 常见误区
***
- **把流量控制和拥塞控制混为一谈**：流量控制保护接收方，拥塞控制保护网络，实际窗口取 `min(cwnd, rwnd)`。
- **以为 cwnd 是接收方通告的**：`cwnd` 是发送方维护的，接收方通告的是 `rwnd`。
- **以为丢包一定是网络坏了**：无线误码也会丢包，但 TCP 通常把它当拥塞信号。
- **以为慢启动真的增长慢**：慢启动起点低，但按 RTT 轮次是指数增长。
- **以为超时和快速重传处理一样**：超时回 `cwnd=1`，快速恢复只减半。
- **以为快速恢复不减速**：它仍然把 `ssthresh` 减半，只是不回到 1。
- **以为 AIMD 对所有流量都公平**：UDP 和多连接不遵守 AIMD，会抢占带宽。

<br>

<br>

## 自测三问
***
1. `cwnd` 和 `rwnd` 分别限制什么？实际发送窗口取哪个值？
2. 慢启动为什么叫“慢”但实际增长很快？到 `ssthresh` 后为什么切换策略？
3. 超时重传和 3 个重复 ACK 的拥塞处理有什么不同？AIMD 的“加法增大、乘法减小”具体指什么？

<details><summary>参考答案</summary>1. `cwnd` 是拥塞窗口，保护网络；`rwnd` 是接收窗口，保护接收方；实际发送窗口取 `min(cwnd, rwnd)`。2. 慢启动起点低，但每收到一个 ACK `cwnd` 加 1，按 RTT 轮次近似指数增长，所以叫“慢”是指起点低，不是增长慢。到 `ssthresh` 后切换到拥塞避免，是因为已经探到接近网络容量，改为线性增长，避免过快把网络压垮。3. 超时重传把 `ssthresh` 设为 `cwnd/2`，`cwnd=1`，重回慢启动；3 个重复 ACK 触发快速恢复，`ssthresh=cwnd/2`，`cwnd=ssthresh+3`，不回到 1。AIMD 指加法增大（每个 RTT 约加 1 MSS）、乘法减小（拥塞时 `cwnd` 减半），多流竞争时趋向公平。</details>

<br>

<br>

## 总结
***
**TCP 拥塞控制像老司机：慢启动试探踩油门，到 `ssthresh` 后拥塞避免稳住，轻微丢包用快速恢复减半补救，严重超时才回到起点；`cwnd` 保护网络，`rwnd` 保护接收方，实际窗口取两者较小值，AIMD 在不让网络崩溃的前提下尽量多拉快跑。**

<br>

<br>

## 参考资料
***
- RFC 5681：TCP Congestion Control
- RFC 9293：Transmission Control Protocol
- RFC 6582：The NewReno Modification to TCP's Fast Recovery Algorithm
- RFC 8312：CUBIC for Fast Long-Distance Networks
- `man 7 tcp`、`man 7 tcp_congestion_control`
- 需按环境核实：不同内核版本的初始 `cwnd` 值、默认拥塞控制算法（Reno/CUBIC/BBR 等）可能不同；`ss -ti` 输出字段因系统而异；快速恢复的 `+3` 是经典 Reno 近似行为，不同算法实现有差异；AIMD 公平性只在守规矩的 TCP 流之间成立。
<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>