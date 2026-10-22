# 20 TCP 重传与可靠传输：超时重传、RTO、快速重传、SACK/D-SACK
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
你发了一个 TCP 包，对方一直没回 ACK。

你可能会想：**等多久重传？**

等太短，网络只是抖了一下你就重发，反而加剧拥塞；等太长，真丢包了又反应慢。

再想一个场景：你连发 1、2、3、4、5，结果 2 丢了，接收方一直回 ACK1。你收到三个 ACK1，能不能不等超时就直接补发 2？补发 2 之后，3、4、5 要不要也重发？

这一章咱们用一条问题链往前走：

1. 等多久没 ACK 才重传，RTO 是固定值吗？
2. 网络抖动时 RTO 为什么要拉长，超时后为什么还要翻倍？
3. 收到重复 ACK 说明什么，为什么收到 3 个就快速重传？
4. SACK 和 D-SACK 又解决什么问题，为什么能少重传？

<br>

<br>

## 优先级说明
***
这篇是 TCP 重传机制的核心，面试常问，项目里排查重传、吞吐下降也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- RTO 应略大于 RTT，但 RTT 会波动，所以 RTO 要动态计算
- 常见公式：`RTO = EstimatedRTT + 4 * DevRTT`，`4 * DevRTT` 是给波动留余量
- 超时重传后 RTO 指数退避，避免网络差时疯狂重发
- 连续收到 3 个重复 ACK 触发快速重传，不等超时
- SACK 让接收方告诉发送方已收到哪些区间，发送方只补缺失段
- D-SACK 报告重复收到的区间，帮助判断是数据丢了还是 ACK 丢了
- SACK/D-SACK 需要双方通过 TCP 选项协商支持

**了解即可（知道大概，不展开）：**
- `tcp_retries2` 的具体行为和内核版本差异
- RTO 上限的具体取值
- SACK 选项的块数量和格式
- D-SACK 在拥塞控制中的具体应用

**主线**：超时重传兜底 → RTO 动态适应网络并指数退避 → 重复 ACK 早发现丢包 → 快速重传减少等待 → SACK 精确补缺 → D-SACK 识别重复原因。

<br>

<br>

## 知识点 1：超时重传与 RTO——智能闹钟，网络稳就准，网络抖就宽容
***
### 问题

等多久没 ACK 才重传？RTO 是固定值吗？为什么网络抖动时 RTO 要拉长？超时重传后，下次超时时间为什么还要翻倍？

**你可能会想**：RTO 设个固定值不就行了？比如 1 秒没 ACK 就重传。

不行。**网络稳的时候 RTT 可能只有几十毫秒，固定 1 秒太慢；网络抖的时候 RTT 可能突然涨到几百毫秒，固定 1 秒又会误重传。** RTO 必须跟着 RTT 和波动动态调整。

<br>

### 核心概念与方案

先把 RTO 换成你熟悉的场景：**RTO 是智能闹钟，网络稳就准，网络抖就宽容。**

TCP 给发送数据设定时器，超时没 ACK 就重传。RTO 应略大于 RTT，但 RTT 会波动，所以 RTO 要动态计算：

- 采样 RTT，算平滑值 `EstimatedRTT`。
- 算波动 `DevRTT`。
- 常见公式：`RTO = EstimatedRTT + 4 * DevRTT`。

加 `4 * DevRTT` 是为了给波动留余量：网络稳时 RTO 接近 RTT，网络抖时 RTO 自动拉长，避免误重传。

超时重传后，下次超时时间通常**加倍（指数退避）**，避免网络差时疯狂重发。连接建立后的数据重传最大次数受 `tcp_retries2` 影响，默认 15，但不是简单重传 15 次，还受计算出的 timeout 限制。

<br>

### 最小实验：只看 RTO 在稳定和抖动两种网络下是不是不同

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 且支持 C++17。下面只做本地模拟，不联网。

保存为 `rto_calc.cpp`：

````cpp
#include <algorithm>
#include <cmath>
#include <iomanip>
#include <iostream>

struct RtoState {
    double estimated_rtt = 0.0;
    double dev_rtt = 0.0;
    bool initialized = false;
};

void update(RtoState& s, double sample_rtt) {
    if (!s.initialized) {
        s.estimated_rtt = sample_rtt;
        s.dev_rtt = sample_rtt / 2.0;
        s.initialized = true;
        return;
    }
    const double alpha = 0.125;
    const double beta = 0.25;

    s.dev_rtt = (1 - beta) * s.dev_rtt + beta * std::fabs(sample_rtt - s.estimated_rtt);
    s.estimated_rtt = (1 - alpha) * s.estimated_rtt + alpha * sample_rtt;
}

double rto(const RtoState& s) {
    return s.estimated_rtt + 4.0 * s.dev_rtt;
}

int main() {
    std::cout << std::fixed << std::setprecision(3);

    RtoState stable;
    for (double rtt : {0.100, 0.102, 0.098, 0.101, 0.099}) {
        update(stable, rtt);
        std::cout << "稳定网络 sample=" << rtt
                  << "s EstimatedRTT=" << stable.estimated_rtt
                  << "s DevRTT=" << stable.dev_rtt
                  << "s RTO=" << rto(stable) << "s\n";
    }

    std::cout << "\n";

    RtoState jitter;
    for (double rtt : {0.100, 0.400, 0.120, 0.500, 0.150}) {
        update(jitter, rtt);
        std::cout << "抖动网络 sample=" << rtt
                  << "s EstimatedRTT=" << jitter.estimated_rtt
                  << "s DevRTT=" << jitter.dev_rtt
                  << "s RTO=" << rto(jitter) << "s\n";
    }

    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 rto_calc.cpp -o rto_calc
./rto_calc
````

**这次只看一件事**：抖动网络的 RTO 是不是明显大于稳定网络。

预期输出类似：

````text
稳定网络 sample=0.100s EstimatedRTT=0.100s DevRTT=0.050s RTO=0.300s
稳定网络 sample=0.102s EstimatedRTT=0.100s DevRTT=0.038s RTO=0.252s
...
抖动网络 sample=0.100s EstimatedRTT=0.100s DevRTT=0.050s RTO=0.300s
抖动网络 sample=0.400s EstimatedRTT=0.138s DevRTT=0.103s RTO=0.550s
抖动网络 sample=0.120s EstimatedRTT=0.135s DevRTT=0.082s RTO=0.463s
抖动网络 sample=0.500s EstimatedRTT=0.181s DevRTT=0.120s RTO=0.661s
抖动网络 sample=0.150s EstimatedRTT=0.177s DevRTT=0.098s RTO=0.569s
````

**关键观察点**：抖动网络的 RTO 明显更大，稳定网络 RTO 接近 RTT 加一点余量。

**结论**：RTO 不是固定值，网络稳时小、抖时大；`4 * DevRTT` 是给波动留余量，避免把延迟误判成丢包。

如果稳定网络 RTO 也很大，说明 RTT 采样或公式写错了。

<br>

### 易错点与边界

坑一：**RTO 太小会无谓重传加剧拥塞**，把本来只是延迟的包当成丢了。

坑二：**RTO 太大丢包反应慢**，恢复时间长。

坑三：**`tcp_retries2` 不是绝对次数**，受 timeout 约束；不同内核版本行为也可能有差异。

坑四：**指数退避不是“每次都翻倍到无限”**，有上限，具体受内核实现限制。

<br>

### 一句话本质

**RTO 是智能闹钟：网络稳就准，网络抖就宽容；超时后还指数退避，避免网络差时疯狂重发。**

<br>

### 自测

为什么 RTO 要加 `4 * DevRTT`？如果去掉这一项，网络抖动时会发生什么？

<details><summary>答案</summary>`4 * DevRTT` 是给 RTT 波动留余量。网络抖动时 RTT 可能突然变大，如果 RTO 只等于 EstimatedRTT，就会把本来只是延迟的包误判成丢包，触发无谓重传，反而加剧拥塞。加上 `4 * DevRTT` 后，网络抖动越大，RTO 越宽容，减少误重传。</details>

<br>

<br>

## 知识点 2：快速重传——听到三次缺货就补发，不等超时
***
### 问题

等超时太慢，能不能早点重传？收到重复 ACK 说明什么？为什么收到 3 个重复 ACK 就快速重传，而不是 1 个？

**你可能会想**：收到一个重复 ACK 就重传，不是更快吗？

不行。**网络乱序也会导致重复 ACK。** 如果收到一个就重传，可能把本来没丢的包重发一遍。等 3 个重复 ACK，是为了降低把乱序误判成丢包的概率。

<br>

### 核心概念与方案

先把快速重传换成你熟悉的场景：**接收方一直在说“我还缺 2 号”，说了三次，发送方就认为 2 号真丢了，立刻补发，不等超时。**

接收方收到乱序包会重复发 ACK，告诉发送方“我还在等那个缺失的字节”。发送方连续收到 **3 个重复 ACK**，就认为中间包丢了，立刻**快速重传**，不等超时。

但快速重传有个问题：它只知道“最早缺的那个包”可能丢了，不知道后面还有没有丢。于是有 SACK。

<br>

### 最小实验：只看重复 ACK 到第几个才触发快速重传

环境前提：同上。下面只做本地模拟，不联网。

保存为 `fast_retransmit.cpp`：

````cpp
#include <iostream>

int main() {
    const int dup_threshold = 3;
    int dup_acks = 0;

    auto on_dup_ack = [&](int ack_num) {
        ++dup_acks;
        std::cout << "收到重复 ACK " << ack_num
                  << "，累计 " << dup_acks << " 个\n";

        if (dup_acks == dup_threshold) {
            std::cout << "触发快速重传，补发包 " << ack_num << "\n";
        }
    };

    on_dup_ack(1);
    on_dup_ack(1);
    on_dup_ack(1);

    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 fast_retransmit.cpp -o fast_retransmit
./fast_retransmit
````

**这次只看一件事**：第几个重复 ACK 之后才打印“触发快速重传”。

预期输出：

````text
收到重复 ACK 1，累计 1 个
收到重复 ACK 1，累计 2 个
收到重复 ACK 1，累计 3 个
触发快速重传，补发包 1
````

**关键观察点**：前两个重复 ACK 没有触发重传，第三个才触发。

**结论**：快速重传不是收到一个重复 ACK 就重传，要够 3 个，降低乱序误判。

如果把阈值改成 1，第一个重复 ACK 就会触发重传，网络乱序时容易误重传。

<br>

### 易错点与边界

坑一：**3 个重复 ACK 是常见触发条件**，但具体实现可能不同；不是收到一个重复 ACK 就重传。

坑二：**快速重传不是“一收到重复 ACK 就重传”**，要够 3 个，避免把乱序误判成丢包。

坑三：**快速重传只补最早缺失的包**，后面有没有丢它不知道，要靠 SACK 或后续 ACK 判断。

坑四：**重复 ACK 不一定都是丢包**，网络乱序也会产生；3 个阈值是折中。

<br>

### 一句话本质

**快速重传是“听到三次缺货就补发”，不等超时；3 个重复 ACK 是为了降低乱序误判，不是越多越好也不是越少越好。**

<br>

### 自测

快速重传的触发条件是什么？为什么是 3 个重复 ACK，而不是 1 个？

<details><summary>答案</summary>发送方连续收到 3 个重复 ACK 时触发快速重传。用 3 个而不是 1 个，是因为网络乱序也会产生重复 ACK，收到一个就重传容易把乱序误判成丢包；3 个是折中，既能较早发现丢包，又降低误判概率。</details>

<br>

<br>

## 知识点 3：SACK 与 D-SACK——精确补缺，识别重复
***
### 问题

快速重传只知道最早缺的包，后面还有没有丢怎么办？SACK 和 D-SACK 又分别解决什么问题？

**你可能会想**：快速重传补了最早的包，后面的包如果也丢了，等超时再补不就行了？

可以，但慢。**SACK 让接收方在 ACK 里告诉发送方“我已经收到哪些区间”，发送方只重传缺失段，不用把后面全重发，也不用等超时。**

<br>

### 核心概念与方案

先把 SACK 换成你熟悉的场景：**接收方拿着一张收货清单，告诉发送方“1 到了、3 到 5 到了，就缺 2”。**

- **SACK（选择性确认）**：接收方在 ACK 里告诉发送方“我已经收到哪些区间”，发送方只重传缺失段，不用把后面全重发。
- **D-SACK（Duplicate SACK）**：用 SACK 告诉发送方“哪些数据重复收到了”，可以用来判断是数据丢了、ACK 丢了，还是网络延迟/复制，帮助发送方区分拥塞和误判。

SACK/D-SACK 需要双方通过 TCP 选项协商支持，不是所有连接都开。

<br>

### 最小实验：只看普通 SACK 和 D-SACK 报告的区间有什么不同

环境前提：同上。下面只做本地模拟，不联网。

保存为 `sack_dsack.cpp`：

````cpp
#include <iostream>
#include <vector>

int main() {
    // 接收方视角：收到 1，缺 2，收到 3~5
    std::cout << "普通 SACK：\n";
    std::cout << "  ACK=2（还在等 2）\n";
    std::cout << "  SACK 区间：3~5 已收到\n";
    std::cout << "  发送方动作：只重传 2\n\n";

    // D-SACK 场景：发送方重传 2，接收方发现 2 之前已经收到过
    std::cout << "D-SACK：\n";
    std::cout << "  ACK=6（6 之前都到了）\n";
    std::cout << "  D-SACK 区间：2~2 重复收到\n";
    std::cout << "  发送方判断：2 不是丢了，可能是 ACK 丢了或网络延迟\n";

    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 sack_dsack.cpp -o sack_dsack
./sack_dsack
````

**这次只看一件事**：普通 SACK 报告的是“缺什么、已收到什么”，D-SACK 报告的是“什么重复收到了”。

预期输出：

````text
普通 SACK：
  ACK=2（还在等 2）
  SACK 区间：3~5 已收到
  发送方动作：只重传 2

D-SACK：
  ACK=6（6 之前都到了）
  D-SACK 区间：2~2 重复收到
  发送方判断：2 不是丢了，可能是 ACK 丢了或网络延迟
````

**关键观察点**：普通 SACK 的区间是“已收到但乱序”的区间，D-SACK 的区间是“重复收到”的区间。

**结论**：SACK 让发送方只补缺失段，少重传；D-SACK 让发送方知道有东西重复到了，帮助判断是数据丢了还是 ACK 丢了。

<br>

### 易错点与边界

坑一：**SACK 需双方支持**，不是所有连接都开；协商阶段要通过 TCP 选项确认。

坑二：**D-SACK 能区分 ACK 丢包和网络延迟，不是普通 SACK**；它报告的是“重复收到的区间”。

坑三：**SACK 不是“重传所有未确认的包”**，它让发送方只补缺失段。

坑四：**SACK 选项块数有限**，接收方不可能把所有区间都塞进一个 ACK，具体格式需按实现核实。

<br>

### 一句话本质

**SACK 是“告诉我还缺哪一段，别全重发”，D-SACK 是“告诉你有东西重复到了”；一个减少重传，一个帮助判断重复原因。**

<br>

### 自测

SACK 解决了快速重传的什么问题？D-SACK 又比普通 SACK 多告诉发送方什么信息？

<details><summary>答案</summary>快速重传只知道最早缺失的包可能丢了，不知道后面还有没有丢；SACK 让接收方告诉发送方已收到哪些区间，发送方只重传缺失段，不用把后面全重发。D-SACK 比普通 SACK 多告诉发送方“哪些数据重复收到了”，帮助判断是数据丢了、ACK 丢了，还是网络延迟/复制，从而区分拥塞和误判。</details>

<br>

<br>

## 完整可复现示例：RTO 计算 + 指数退避 + 快速重传 + SACK 一次串起来
***
你需要 Linux/macOS 或 WSL，已安装 `g++`。下面把三个模拟串起来，每个步骤只看一个观察点。

````bash
set -e

echo "=== 1. RTO 动态计算 ==="
g++ -std=c++17 -O2 rto_calc.cpp -o rto_calc
./rto_calc

echo "=== 2. 快速重传阈值 ==="
g++ -std=c++17 -O2 fast_retransmit.cpp -o fast_retransmit
./fast_retransmit

echo "=== 3. SACK 与 D-SACK 报告区间 ==="
g++ -std=c++17 -O2 sack_dsack.cpp -o sack_dsack
./sack_dsack
````

可能的输出/现象：

````text
=== 1. RTO 动态计算 ===
稳定网络 RTO 接近 RTT 加一点余量
抖动网络 RTO 明显更大

=== 2. 快速重传阈值 ===
前两个重复 ACK 不重传，第三个触发

=== 3. SACK 与 D-SACK ===
SACK 报告已收到 3~5，缺 2
D-SACK 报告 2~2 重复收到
````

**关键观察点**：第 1 步看抖动网络 RTO 是否更大；第 2 步看第几个重复 ACK 触发重传；第 3 步看 SACK 和 D-SACK 区间含义是否不同。

验证结果：RTO 动态适应网络，快速重传等 3 个重复 ACK，SACK 精确补缺，D-SACK 识别重复。

注意：这些是教学模拟，真实 TCP 还要处理定时器、窗口、拥塞控制、SACK 选项格式等；`tcp_retries2` 和 RTO 上限的具体行为需按内核版本核实。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| 抓包看到大量重传 | 是超时重传还是快速重传 | 看时间间隔和重复 ACK | 超时重传反应慢；快速重传由重复 ACK 触发，更快 |
| 吞吐上不去，重传多 | 是否频繁丢包、RTO 是否过小 | `ss -ti` 看 RTO、重传计数；抓包看重传 | RTO 过小会误重传，丢包多会限制吞吐 |
| 重复 ACK 很多 | 是否丢包触发快速重传 | 看同一 ACK 重复次数 | 重复 ACK 常是接收方在提示“我缺这个包” |
| SACK 选项没出现 | 双方是否支持 SACK | 抓包看 TCP 选项 | SACK 需要双方协商支持，不是所有连接都开 |
| D-SACK 出现 | 是否有重复数据到达 | 抓包看 D-SACK 区间 | D-SACK 报告重复收到的区间，帮助判断 ACK 丢失或网络延迟 |
| 网络抖动时重传暴增 | RTO 是否太激进 | 看 RTO 值和 RTT 波动 | RTO 应加 `4 * DevRTT` 留余量，避免把延迟误判成丢包 |
| 大量重传但抓包没看到丢包 | 是否 ACK 丢失、是否 D-SACK | 抓包看双向 ACK；看 D-SACK | 可能是 ACK 丢了导致发送方重传，D-SACK 能帮助识别 |

<br>

<br>

## 常见误区
***
- **以为 RTO 是固定值**：RTO 跟着 RTT 和波动动态调整，网络抖时会拉长。
- **以为超时后 RTO 不变**：超时后通常指数退避，避免网络差时疯狂重发。
- **以为收到一个重复 ACK 就快速重传**：通常要 3 个，降低乱序误判。
- **以为快速重传能补所有丢失的包**：它只知道最早缺失的包，后面要靠 SACK 或后续 ACK。
- **以为 SACK 是“重传所有未确认包”**：SACK 让发送方只补缺失段。
- **把 D-SACK 当普通 SACK**：D-SACK 报告的是重复收到的区间，帮助判断重复原因。
- **以为 SACK 默认开启**：需要双方通过 TCP 选项协商支持。

<br>

<br>

## 自测三问
***
1. RTO 和 RTT 是什么关系？为什么 RTO 要加 `4 * DevRTT`？超时后为什么要指数退避？
2. 快速重传的触发条件是什么？为什么是 3 个重复 ACK，而不是 1 个？
3. SACK 和 D-SACK 分别解决什么问题？它们为什么需要双方支持？

<details><summary>参考答案</summary>1. RTO 应略大于 RTT，但 RTT 会波动，所以用 `EstimatedRTT + 4 * DevRTT` 动态计算；`4 * DevRTT` 给波动留余量，避免把延迟误判成丢包。超时后指数退避，避免网络差时疯狂重发，加剧拥塞。2. 发送方连续收到 3 个重复 ACK 时触发快速重传。用 3 个而不是 1 个，是因为网络乱序也会产生重复 ACK，收到一个就重传容易误判；3 个是折中。3. SACK 让接收方告诉发送方已收到哪些区间，发送方只重传缺失段，减少多余重传；D-SACK 报告重复收到的区间，帮助判断是数据丢了、ACK 丢了还是网络延迟。它们需要双方通过 TCP 选项协商支持，不是所有连接都开。</details>

<br>

<br>

## 总结
***
**TCP 可靠传输靠超时重传兜底、动态 RTO 适应网络并指数退避、快速重传减少等待、SACK 精确补缺、D-SACK 识别重复；目标是用尽量少的重传把字节流可靠送达。**

<br>

<br>

## 参考资料
***
- RFC 9293：Transmission Control Protocol
- RFC 2018：TCP Selective Acknowledgment Options
- RFC 2883：An Extension to the Selective Acknowledgement (SACK) Option for TCP（D-SACK）
- RFC 5681：TCP Congestion Control
- RFC 6298：Computing TCP's Retransmission Timer
- 需按环境核实：`tcp_retries2` 的行为和默认值因内核版本而异；RTO 上限和指数退避的具体实现需按内核核实；SACK 选项块数和格式需按实现核实；`ss -ti` 输出字段因系统而异。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>