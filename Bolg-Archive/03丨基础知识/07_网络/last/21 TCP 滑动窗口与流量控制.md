# 21 TCP 滑动窗口与流量控制：窗口、零窗口、糊涂窗口、Nagle/延迟确认
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
如果 TCP 每发一个包就停下来等 ACK，带宽大部分时间在空转。

你可能会想：那就不等，一直发不就行了？

可一直发，接收方缓存又可能被撑爆。发得太快，对方处理不过来，包只能丢，丢了又要重传，反而更慢。

再想一个场景：你写了个交互式程序，每次只发几个字节，结果延迟莫名其妙变大。很可能是 **Nagle 在等更多数据**，而**延迟确认在等数据捎带 ACK**，两边互相等。

这一章咱们用一条问题链往前走：

1. 怎么既提高发送效率，又不撑爆接收方？
2. 发送方到底能连续发多少数据，窗口由谁决定？
3. 接收方缓存满了，窗口变成 0，发送方怎么知道什么时候能继续发？
4. 每次只发几个字节，Nagle 和延迟确认为什么会互相添堵？

<br>

<br>

## 优先级说明
***
这篇是 TCP 流量控制的核心，面试常问，项目里排查吞吐上不去、交互延迟高也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- 窗口表示无需等待确认就能继续发送的数据最大值
- `rwnd` 是接收窗口，保护接收方；`cwnd` 是拥塞窗口，保护网络
- 实际发送窗口取 `min(cwnd, rwnd)`
- 零窗口不是永久停，发送方要窗口探测，防止窗口更新 ACK 丢失导致死锁
- 糊涂窗口综合症：接收方不通告小窗口，发送方不发小包
- Nagle 减少小包，延迟确认减少空 ACK，两者一起用可能互相等
- 交互式程序常关 Nagle（`TCP_NODELAY`），或关延迟确认（`TCP_QUICKACK`）

**了解即可（知道大概，不展开）：**
- 窗口扩大因子的具体协商过程
- `TCP_QUICKACK` 每次读后重设的细节
- `SO_SNDBUF`/`SO_RCVBUF` 对动态调节的影响
- 持续定时器的具体超时时间

**主线**：窗口让发送方连续发 → `rwnd` 防止撑爆接收方 → 零窗口靠探测防死锁 → 小包太多用 Nagle 和延迟确认 → 两者一起用可能互相等，交互式程序要关一个。

<br>

<br>

## 知识点 1：发送窗口、接收窗口与流量控制——接收方喊“我还能装多少”
***
### 问题

怎么既提高发送效率又不撑爆接收方？发送方到底能连续发多少数据？接收方缓存满了怎么办？零窗口后发送方怎么知道什么时候能继续发？

**你可能会想**：窗口是不是发送方自己定的？我想发多少就发多少？

不是。**窗口是接收方告诉发送方的：“我还能装多少。”** 发送方按这个量发，发多了接收方装不下，只能丢。

<br>

### 核心概念与方案

先把窗口换成你熟悉的场景：**接收方喊“我兜里还能装多少”，发送方按这个量发。**

窗口表示**无需等待确认就能继续发送的数据最大值**。接收方在 TCP 头窗口字段告诉发送方自己还有多少缓存，这就是 `rwnd`（接收窗口）。

发送方实际发送窗口受 `min(cwnd, rwnd)` 限制；不考虑拥塞控制时约等于接收窗口。`cwnd` 是拥塞窗口，保护网络；`rwnd` 是接收窗口，保护接收方。

接收窗口为 0 时发送方停止发送，但不会永久停：它启动**持续定时器**，定期发**窗口探测**，防止窗口更新 ACK 丢失导致双方死锁。接收方窗口恢复后，发送方收到窗口更新再继续发。

<br>

### 最小实验：只看 rwnd=0 后发送方是不是停止发送并启动探测

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 且支持 C++17。下面只做本地模拟，不联网。

保存为 `window_probe.cpp`：

````cpp
#include <iostream>

int main() {
    int rwnd = 1000;
    int in_flight = 0;
    int probe_count = 0;

    auto send = [&](int bytes) {
        if (rwnd == 0) {
            std::cout << "rwnd=0，停止发送，启动窗口探测\n";
            return;
        }
        int can_send = std::min(rwnd, bytes);
        in_flight += can_send;
        std::cout << "发送 " << can_send << " 字节，在途 " << in_flight << "\n";
    };

    send(500);
    send(500);
    rwnd = 0;
    send(100);

    std::cout << "开始窗口探测...\n";
    for (int i = 1; i <= 3; ++i) {
        ++probe_count;
        std::cout << "第 " << probe_count << " 次探测，仍 rwnd=0\n";
    }

    rwnd = 2000;
    std::cout << "收到窗口更新 rwnd=2000，继续发送\n";
    send(1000);
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 window_probe.cpp -o window_probe
./window_probe
````

**这次只看一件事**：`rwnd=0` 之后，程序是继续发数据，还是打印“停止发送，启动窗口探测”。

预期输出：

````text
发送 500 字节，在途 500
发送 500 字节，在途 1000
rwnd=0，停止发送，启动窗口探测
开始窗口探测...
第 1 次探测，仍 rwnd=0
第 2 次探测，仍 rwnd=0
第 3 次探测，仍 rwnd=0
收到窗口更新 rwnd=2000，继续发送
发送 1000 字节，在途 2000
````

**关键观察点**：`rwnd=0` 之后，有没有在收到窗口更新之前继续发数据。

没有。这就是流量控制：**接收方说 0，发送方就停；但停不是永久停，要探测。**

如果 `rwnd=0` 后程序还在发数据，说明模拟逻辑写错了。

<br>

### 易错点与边界

坑一：**流量控制保护接收方，拥塞控制保护网络**，两者不是一回事；实际发送窗口取 `min(cwnd, rwnd)`。

坑二：**窗口字段可扩大**，用窗口扩大因子把 16 位窗口放大，突破 64KB 限制。

坑三：**窗口为 0 不是永久停**，发送方要窗口探测，否则窗口更新 ACK 丢了就死锁。

坑四：**接收窗口是动态的**，应用读得慢，`rwnd` 会变小；应用读得快，窗口才会恢复。

<br>

### 一句话本质

**流量控制就是接收方喊“我兜里还能装多少”，发送方按这个量发；零窗口不是永久停，要探测，防止窗口更新 ACK 丢了双方死锁。**

<br>

### 自测

`rwnd=0` 后发送方怎么知道什么时候能继续发？如果窗口更新 ACK 丢了，双方会怎样？

<details><summary>答案</summary>发送方启动持续定时器，定期发窗口探测，接收方收到探测后如果窗口恢复，会回窗口更新。如果窗口更新 ACK 丢了，发送方靠探测继续问，不会永久停，所以不会死锁。没有探测，双方可能一个等窗口更新、一个等对方发数据，形成死锁。</details>

<br>

<br>

## 知识点 2：糊涂窗口综合症、Nagle 与延迟确认——小包太多，两边互相等
***
### 问题

每次只发几个字节，效率极低；接收方通告很小窗口，发送方就发很小数据，头部开销远大于数据。Nagle 和延迟确认一起用为什么更慢？交互式程序该怎么处理？

**你可能会想**：小包就小包呗，能发出去不就行了？

不行。**每个 TCP 头至少 20 字节，你只发 1 字节数据，头部开销是数据的 20 倍。** 网络里全是这种小包，带宽利用率极低。

<br>

### 核心概念与方案

先把这个问题换成你熟悉的场景：**你每次只寄一张纸，却要用一个巨大的箱子，箱子比纸还重。**

**糊涂窗口综合症**：接收方通告很小窗口，发送方就发很小数据，头部开销远大于数据。解决分两边：

- 接收方：不通告小窗口，小于 `min(MSS, 缓存/2)` 就通告 0。
- 发送方：用 **Nagle 算法**，满足“数据够 MSS”或“收到之前 ACK”才发，减少小包。

**延迟确认**：没有响应数据时，ACK 延迟一会儿，希望搭便车，减少空 ACK。

问题在于：**Nagle + 延迟确认可能互相等待**——发送方等 ACK 才发更多，接收方等数据才捎带 ACK，导致延迟。交互式程序可关闭 Nagle（`TCP_NODELAY`）或关闭延迟确认（`TCP_QUICKACK`）。

注意：`TCP_QUICKACK` 不是永久设置，可能需要每次读后重设；`TCP_NODELAY` 是套接字选项，设置一次通常持续生效。

<br>

### 最小实验：只看 TCP_NODELAY 设置后读回来是不是 1

环境前提：同上。下面只创建套接字、设置选项、读回确认，不联网。

保存为 `tcp_nodelay_demo.cpp`：

````cpp
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <sys/socket.h>
#include <unistd.h>

#include <iostream>

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    int nodelay = 1;
    if (setsockopt(sock, IPPROTO_TCP, TCP_NODELAY,
                   &nodelay, sizeof(nodelay)) < 0) {
        std::cerr << "setsockopt TCP_NODELAY failed\n";
        close(sock);
        return 1;
    }

    int current = 0;
    socklen_t len = sizeof(current);
    if (getsockopt(sock, IPPROTO_TCP, TCP_NODELAY,
                   &current, &len) == 0) {
        std::cout << "TCP_NODELAY = " << current << "\n";
    } else {
        std::cout << "getsockopt TCP_NODELAY failed\n";
    }

    close(sock);
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 tcp_nodelay_demo.cpp -o tcp_nodelay_demo
./tcp_nodelay_demo
````

**这次只看一件事**：输出里 `TCP_NODELAY` 是不是 1。

预期输出：

````text
TCP_NODELAY = 1
````

是 1，说明 Nagle 已关闭，小数据会立即发送。如果输出不是 1，先检查 `setsockopt` 是否失败、系统是否支持该选项。

**结论**：`TCP_NODELAY=1` 关闭 Nagle，适合交互式程序；但批量传输关不关影响不大，不要无脑关。

<br>

### 易错点与边界

坑一：**Nagle 默认开**，不是所有场景都适合关；批量传输关不关影响不大，交互式程序常关。

坑二：**telnet/ssh 等交互程序常关 Nagle**，避免小数据被囤住。

坑三：**`TCP_QUICKACK` 不是永久设置**，可能需要每次读后重设；不同系统支持不同。

坑四：**糊涂窗口综合症是双方的问题**，接收方不通告小窗口、发送方不发小包，两边都要配合。

坑五：**关 Nagle 不等于关延迟确认**，两者是不同层面的机制，要分开调。

<br>

### 一句话本质

**Nagle 减少小包，延迟确认减少空 ACK；两者一起用可能互相等，交互式程序要关一个，通常关 Nagle。**

<br>

### 自测

为什么 Nagle 和延迟确认混合使用可能增加时延？交互式程序通常怎么处理？

<details><summary>答案</summary>Nagle 会等数据够 MSS 或收到之前 ACK 才发，延迟确认会等一会儿看有没有数据可捎带 ACK。发送方等 ACK 才发更多，接收方等数据才捎带 ACK，两边互相等，导致延迟。交互式程序通常关闭 Nagle（`TCP_NODELAY`），让数据立即发送；也有场景关延迟确认（`TCP_QUICKACK`），但后者不是永久设置，可能需要每次读后重设。</details>

<br>

<br>

## 完整可复现示例：查看窗口参数 + 用 C++ 设置 TCP_NODELAY
***
你需要 Linux/macOS 或 WSL，已安装 `sysctl`、`g++`。下面每个步骤只看一件事，不展开无关细节。

````bash
set -e

echo "=== 1. 接收/发送缓冲区 ==="
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem net.ipv4.tcp_window_scaling

echo "=== 2. TCP_NODELAY 设置与读回 ==="
g++ -std=c++17 -O2 tcp_nodelay_demo.cpp -o tcp_nodelay_demo
./tcp_nodelay_demo
````

可能的输出：

````text
=== 1. 接收/发送缓冲区 ===
net.ipv4.tcp_rmem = 4096 87380 6291456
net.ipv4.tcp_wmem = 4096 16384 4194304
net.ipv4.tcp_window_scaling = 1

=== 2. TCP_NODELAY 设置与读回 ===
TCP_NODELAY = 1
````

**关键观察点**：第 1 步看 `tcp_window_scaling` 是不是 1；第 2 步看 `TCP_NODELAY` 是不是 1。

验证结果：缓冲区决定窗口上限；窗口扩大让窗口可超过 64KB；`TCP_NODELAY=1` 表示 Nagle 已关闭，小数据立即发送。

注意：不同系统、不同内核版本输出可能不同；`SO_SNDBUF`/`SO_RCVBUF` 显示的是内核调整后的值，程序里乱设可能关闭动态调节，反而影响性能；`TCP_QUICKACK` 需要每次读后重设，示例没有演示，避免给出不完整的永久设置假象，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| 吞吐上不去 | 窗口是否太小、是否频繁零窗口 | `ss -ti` 看 cwnd/rwnd；抓包看窗口字段 | 窗口小限制在途数据；零窗口说明接收方处理不过来 |
| 接收方处理慢，窗口变小 | 应用读得慢、缓冲区小 | 看 `rwnd` 变化；查应用读取速度 | 应用读得慢，接收缓存被占满，`rwnd` 变小甚至为 0 |
| 零窗口后连接卡住 | 窗口探测是否生效 | 抓包看窗口探测；看窗口更新 | 零窗口不是永久停，靠探测防死锁；更新 ACK 丢了也能靠探测恢复 |
| 交互式程序延迟高 | 是否开了 Nagle、是否延迟确认 | `ss -ti` 看选项；抓包看小包和 ACK 延迟 | Nagle 等更多数据，延迟确认等捎带 ACK，两边互相等 |
| 小包很多 | 是否该开 Nagle、是否应用发得太碎 | 抓包看包大小；查应用发送逻辑 | 应用发得太碎，Nagle 可以合并；但交互式程序可能反而要关 Nagle |
| `TCP_NODELAY` 设置无效 | 是否设置成功、是否在正确套接字上 | `getsockopt` 读回；查代码 | 设置失败或设在错误套接字上；读回确认最直接 |
| `TCP_QUICKACK` 不持久 | 是否需要每次读后重设 | 查代码；抓包看 ACK 延迟 | `TCP_QUICKACK` 不是永久设置，可能需要每次读后重设 |

<br>

<br>

## 常见误区
***
- **把流量控制和拥塞控制混为一谈**：流量控制保护接收方，拥塞控制保护网络，实际窗口取 `min(cwnd, rwnd)`。
- **以为零窗口会永久停**：发送方会窗口探测，防止窗口更新 ACK 丢失导致死锁。
- **以为窗口字段永远只有 64KB**：窗口扩大因子可以把窗口放大，突破 64KB。
- **以为 Nagle 默认关**：Nagle 默认开，交互式程序常关，批量传输关不关影响不大。
- **以为关 Nagle 就等于关延迟确认**：两者是不同层面的机制，要分开调。
- **以为 `TCP_QUICKACK` 设置一次就永久生效**：它可能需要每次读后重设。
- **无脑调大 `SO_SNDBUF`/`SO_RCVBUF`**：可能关闭动态调节，反而影响性能。

<br>

<br>

## 自测三问
***
1. 发送窗口、接收窗口、拥塞窗口是什么关系？实际发送窗口取谁的 `min`？
2. 零窗口时为什么需要窗口探测？如果窗口更新 ACK 丢了，双方会怎样？
3. Nagle 和延迟确认分别减少什么？为什么可能互相添堵？交互式程序通常怎么处理？

<details><summary>参考答案</summary>1. `rwnd` 是接收窗口，保护接收方；`cwnd` 是拥塞窗口，保护网络；实际发送窗口取 `min(cwnd, rwnd)`。2. 零窗口时发送方停止发送，但窗口更新 ACK 可能丢失，如果一直等可能死锁；窗口探测让发送方定期问接收方窗口是否恢复，避免死锁。3. Nagle 减少小包，延迟确认减少空 ACK；两者一起用，发送方等 ACK 才发更多，接收方等数据才捎带 ACK，可能互相等。交互式程序常关 Nagle（`TCP_NODELAY`），让数据立即发送；也有场景关延迟确认（`TCP_QUICKACK`），但后者不是永久设置。</details>

<br>

<br>

## 总结
***
**滑动窗口让 TCP 连续发送，流量控制用 `rwnd` 防止撑爆接收方，零窗口靠探测防死锁；Nagle 减少小包，延迟确认减少空 ACK，但两者混用可能增加交互延迟，交互式程序通常关 Nagle。**

<br>

<br>

## 参考资料
***
- RFC 9293：Transmission Control Protocol
- RFC 1122：Requirements for Internet Hosts — Communication Layers
- RFC 896：Congestion Control in IP/TCP Internetworks（Nagle 算法）
- RFC 7323：TCP Extensions for High Performance（窗口扩大、时间戳）
- `man 7 tcp`、`man 7 socket`
- 需按环境核实：`tcp_rmem`、`tcp_wmem`、`tcp_window_scaling` 默认值因内核版本而异；`TCP_QUICKACK` 的可用性和行为因系统而异；`ss -ti` 输出字段因系统而异；窗口扩大因子的协商取决于双方实现。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>