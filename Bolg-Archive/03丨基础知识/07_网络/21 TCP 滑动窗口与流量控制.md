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
如果 TCP 每发一个包都等 ACK，效率太低。**滑动窗口怎么让发送方连续发？接收方处理不过来怎么办？零窗口会死锁吗？Nagle 和延迟确认为什么会互相添堵？**  
先别急着背“发送窗口、接收窗口、rwnd、cwnd”这些词，先看两个具体场景：你发文件时，如果发一个包就停下来等 ACK，带宽大部分时间在空转；可如果不管不顾一直发，接收方缓存又可能被撑爆。再想一个场景：你写了个交互式程序，每次只发几个字节，结果发现延迟莫名其妙变大——很可能是 Nagle 在等更多数据，而延迟确认在等数据捎带 ACK，两边互相等。这一章咱们就按“怎么连续发、怎么不撑爆、怎么处理零窗口、怎么减少小包”这条线走。

<br>

<br>

## 知识点 1：发送窗口、接收窗口与流量控制
***
### 问题
怎么既提高发送效率又不撑爆接收方？发送方到底能连续发多少数据？接收方缓存满了怎么办？零窗口后发送方怎么知道什么时候能继续发？

<br>

### 方案
窗口表示**无需等待确认就能继续发送的数据最大值**。接收方在 TCP 头窗口字段告诉发送方自己还有多少缓存，这就是 `rwnd`（接收窗口）。  
发送方实际发送窗口受 `min(cwnd, rwnd)` 限制；不考虑拥塞控制时约等于接收窗口。`cwnd` 是拥塞窗口，保护网络；`rwnd` 是接收窗口，保护接收方。  
接收窗口为 0 时发送方停止发送，但不会永久停：它启动**持续定时器**，定期发**窗口探测**，防止窗口更新 ACK 丢失导致双方死锁。接收方窗口恢复后，发送方收到窗口更新再继续发。

<br>

### 最小示例
```text
接收方回 ACK 并带 rwnd=1000 → 发送方最多在途 1000 字节
rwnd=0 → 停止发送，定时窗口探测
收到窗口更新 rwnd=2000 → 继续发送
```

结论：流量控制就是接收方喊“我兜里还能装多少”，发送方按这个量发；零窗口不是永久停，要探测。

<br>

### 一句话本质
**流量控制就是接收方喊“我兜里还能装多少”，发送方按这个量发。**

<br>

### 易错点
坑一：**流量控制保护接收方，拥塞控制保护网络**，两者不是一回事；实际发送窗口取 `min(cwnd, rwnd)`。  
坑二：**窗口字段可扩大**，用窗口扩大因子把 16 位窗口放大，突破 64KB 限制。  
坑三：**窗口为 0 不是永久停**，发送方要窗口探测，否则窗口更新 ACK 丢了就死锁。  
坑四：**接收窗口是动态的**，应用读得慢，`rwnd` 会变小；应用读得快，窗口才会恢复。

<br>

### 自测
`rwnd=0` 后发送方怎么知道什么时候能继续发？如果窗口更新 ACK 丢了，双方会怎样？

<br>

<br>

## 知识点 2：糊涂窗口综合症、Nagle 与延迟确认
***
### 问题
每次只发几个字节，效率极低；接收方通告很小窗口，发送方就发很小数据，头部开销远大于数据。Nagle 和延迟确认一起用为什么更慢？交互式程序该怎么处理？

<br>

### 方案
**糊涂窗口综合症**：接收方通告很小窗口，发送方就发很小数据，头部开销远大于数据。解决分两边：  
- 接收方：不通告小窗口，小于 `min(MSS, 缓存/2)` 就通告 0；  
- 发送方：用 **Nagle 算法**，满足“数据够 MSS”或“收到之前 ACK”才发，减少小包。  
**延迟确认**：没有响应数据时，ACK 延迟一会儿，希望搭便车，减少空 ACK。  
问题在于：**Nagle + 延迟确认可能互相等待**——发送方等 ACK 才发更多，接收方等数据才捎带 ACK，导致延迟。交互式程序可关闭 Nagle（`TCP_NODELAY`）或关闭延迟确认（`TCP_QUICKACK`）。  
注意：`TCP_QUICKACK` 不是永久设置，可能需要每次读后重设；`TCP_NODELAY` 是套接字选项，设置一次通常持续生效。

<br>

### 最小示例
```text
Nagle：小数据先囤着，等够 MSS 或收到 ACK 再发
延迟确认：ACK 等一会儿，看有没有数据可捎带
两者一起：可能你等我、我等你
```

结论：Nagle 减少小包，延迟确认减少空 ACK；两者一起用可能互相等，交互式程序要小心。

<br>

### 一句话本质
**Nagle 减少小包，延迟确认减少空 ACK；两者一起用可能互相等，交互式程序要小心。**

<br>

### 易错点
坑一：**Nagle 默认开**，不是所有场景都适合关；批量传输关不关影响不大，交互式程序常关。  
坑二：**telnet/ssh 等交互程序常关 Nagle**，避免小数据被囤住。  
坑三：**`TCP_QUICKACK` 不是永久设置**，可能需要每次读后重设；不同系统支持不同。  
坑四：**糊涂窗口综合症是双方的问题**，接收方不通告小窗口、发送方不发小包，两边都要配合。  
坑五：**关 Nagle 不等于关延迟确认**，两者是不同层面的机制，要分开调。

<br>

### 自测
为什么 Nagle 和延迟确认混合使用可能增加时延？交互式程序通常怎么处理？

<br>

<br>

## 完整可复现示例：查看窗口参数并用 C++ 设置 TCP_NODELAY
***
这个示例分两部分：先用 `sysctl` 看系统窗口和缓冲区参数；再用 C++ 写一个最小程序，给 TCP 套接字打开 `TCP_NODELAY`，关闭 Nagle，并读取当前值确认。  
注意：改系统参数需要权限；这里只读不改，避免影响本机网络。

```bash
set -e

echo "=== 接收缓冲区 ==="
sysctl net.ipv4.tcp_rmem

echo "=== 发送缓冲区 ==="
sysctl net.ipv4.tcp_wmem

echo "=== 窗口扩大 ==="
sysctl net.ipv4.tcp_window_scaling
```

输出可能类似：

```text
=== 接收缓冲区 ===
net.ipv4.tcp_rmem = 4096 87380 6291456

=== 发送缓冲区 ===
net.ipv4.tcp_wmem = 4096 16384 4194304

=== 窗口扩大 ===
net.ipv4.tcp_window_scaling = 1
```

C++ 程序：打开 `TCP_NODELAY` 并读取确认。

```cpp
// tcp_nodelay_demo.cpp
// 编译：g++ -std=c++17 -O2 tcp_nodelay_demo.cpp -o tcp_nodelay_demo
// 运行：./tcp_nodelay_demo
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

    // 关闭 Nagle：小数据立即发送
    int nodelay = 1;
    if (setsockopt(sock, IPPROTO_TCP, TCP_NODELAY,
                   &nodelay, sizeof(nodelay)) < 0) {
        std::cerr << "setsockopt TCP_NODELAY failed\n";
        close(sock);
        return 1;
    }

    // 读回确认
    int current = 0;
    socklen_t len = sizeof(current);
    if (getsockopt(sock, IPPROTO_TCP, TCP_NODELAY,
                   &current, &len) == 0) {
        std::cout << "TCP_NODELAY = " << current << "\n";
    } else {
        std::cout << "getsockopt TCP_NODELAY failed\n";
    }

    // 读取发送/接收缓冲区大小
    int sndbuf = 0, rcvbuf = 0;
    len = sizeof(sndbuf);
    if (getsockopt(sock, SOL_SOCKET, SO_SNDBUF, &sndbuf, &len) == 0) {
        std::cout << "SO_SNDBUF = " << sndbuf << " bytes\n";
    }

    len = sizeof(rcvbuf);
    if (getsockopt(sock, SOL_SOCKET, SO_RCVBUF, &rcvbuf, &len) == 0) {
        std::cout << "SO_RCVBUF = " << rcvbuf << " bytes\n";
    }

    close(sock);
    return 0;
}
```

输出可能类似：

```text
TCP_NODELAY = 1
SO_SNDBUF = 16384 bytes
SO_RCVBUF = 131072 bytes
```

验证结果：缓冲区决定窗口上限；窗口扩大让窗口可超过 64KB；`TCP_NODELAY=1` 表示 Nagle 已关闭，小数据立即发送。  
注意：不同系统、不同内核版本输出可能不同；`SO_SNDBUF`/`SO_RCVBUF` 显示的是内核调整后的值，程序里乱设 `SO_SNDBUF/SO_RCVBUF` 可能关闭动态调节，反而影响性能。`TCP_QUICKACK` 需要每次读后重设，这里没有演示，避免给出不完整的永久设置假象。

<br>

<br>

## 自测三问
***
1. 发送窗口、接收窗口、拥塞窗口是什么关系？实际发送窗口取谁的 `min`？  
2. 零窗口时为什么需要窗口探测？如果窗口更新 ACK 丢了，双方会怎样？  
3. Nagle 和延迟确认分别减少什么？为什么可能互相添堵？交互式程序通常怎么处理？

<br>

<br>

## 一句话总结全章
***
**滑动窗口让 TCP 连续发送，流量控制用 rwnd 防止撑爆接收方，零窗口靠探测防死锁；Nagle 减少小包，延迟确认减少空 ACK，但两者混用可能增加交互延迟。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>