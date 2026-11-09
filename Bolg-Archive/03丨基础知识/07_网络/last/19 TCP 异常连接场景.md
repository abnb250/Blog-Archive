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
正常握手挥手你背得熟：SYN → SYN+ACK → ACK，FIN → ACK → FIN → ACK。但面试和工作常问的是异常：

- 压测时客户端 `connect` 一直超时，服务端却像没收到 SYN 一样，为什么？
- 一条已经 ESTABLISHED 的连接，突然收到一个同四元组的 SYN，内核会怎么处理？
- 进程 `kill -9` 和主机直接断电，对端看到的现象一样吗？
- 拔掉网线后，原来的连接会立即断开吗？

你可能会想：TCP 不是可靠的吗？怎么还有这么多“不知道对方死没死”的情况？

因为 **TCP 的可靠性建立在“能收到报文”的基础上**。对方进程崩溃、主机断电、网线被拔，这些情况下内核根本收不到任何报文，TCP 自己也不知道对方还活不活着。

这一章咱们用一条问题链往前走：

1. 服务端收到 SYN 却不回，是网络丢了还是服务端根本没处理？
2. 已建立连接收到旧 SYN，内核回什么？TIME_WAIT 收到 SYN 一定回 RST 吗？
3. 进程崩溃、主机断电、拔网线，对端看到的现象有什么不同？
4. Keepalive 为什么不能快速发现故障，线上为什么要自己做心跳？

<br>

<br>

## 优先级说明
***
这篇是 TCP 异常场景的进阶，面试常问，项目里排查连接超时、假活连接、TIME_WAIT 异常也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- SYN 被丢弃常见原因：半连接队列满、全连接队列满、历史参数误杀
- `tcp_syncookies` 是半连接队列的应急旁路，但不解决全连接队列满
- 全连接队列满默认丢 ACK，`tcp_abort_on_overflow=1` 才回 RST
- ESTABLISHED 收到旧 SYN 回 Challenge ACK，客户端发现不对回 RST
- TIME_WAIT 收到 SYN 不一定回 RST，合法 SYN 可以复用连接
- 进程崩溃内核会发 FIN；主机断电/拔网线内核不知道，只能靠重传或 Keepalive
- TCP Keepalive 默认 2 小时才启动，线上常需应用层心跳
- TCP Keepalive 和 HTTP Keep-Alive 不是一回事

**了解即可（知道大概，不展开）：**
- `tcp_tw_recycle` 在 Linux 4.12 后已移除
- `tcp_rfc1337` 对 TIME_WAIT 收到 RST 的影响
- PAWS 检查和时间戳的具体规则
- 半连接队列、全连接队列的内核实现细节

**主线**：SYN 可能因队列满被丢 → ESTABLISHED 收到旧 SYN 用 Challenge ACK 纠正 → TIME_WAIT 收到 SYN 看新旧是否合法 → 进程崩溃发 FIN，主机断电/拔网线只能靠重传或 Keepalive 兜底。

<br>

<br>

## 知识点 1：SYN 被丢弃——队列满了还是参数误杀
***
### 问题

服务端收到 SYN 却不回 SYN+ACK，为什么？是网络丢了，还是服务端根本没处理？半连接队列和全连接队列满了分别会怎样？

**你可能会想**：服务端没回，肯定是网络丢包了吧？

不一定。**服务端内核可能收到了 SYN，但因为队列满了，直接丢掉了，根本没走到回 SYN+ACK 那一步。** 客户端看到的就是“connect 超时”，和网络丢包的现象一模一样。

<br>

### 核心概念与方案

先把 SYN 队列换成你熟悉的场景：**服务端前台排队，半连接队列是“正在填表的人”，全连接队列是“填完表等叫号的人”。**

SYN 被丢弃，常见有三类原因：

- **半连接队列满**：SYN 攻击或突发 SYN 太多，内核丢弃新 SYN。开启 `tcp_syncookies=1` 可以绕过半连接队列，用 cookie 代替队列项，缓解 SYN Flood。
- **全连接队列满**：第三次 ACK 到达时 `accept` 队列满，默认丢弃 ACK；设置 `tcp_abort_on_overflow=1` 则回 RST。
- **`tcp_tw_recycle` + NAT**：旧参数按“对端 IP”做 PAWS 检查，NAT 后多个客户端同 IP，时间戳小的 SYN 可能被丢；Linux 4.12 后已取消该参数，现代内核不用再担心这一条。

排查时看内核统计：`netstat -s` 里 “SYNs to LISTEN sockets dropped” 和 “overflowed” 持续增长，说明队列溢出。

<br>

### 最小实验：只看队列溢出统计有没有增长

环境前提：Linux，已安装 `netstat`（`net-tools` 包）。下面只读统计，不改参数，不影响本机网络。

````bash
netstat -s 2>/dev/null | grep -i -E "SYNs to LISTEN sockets dropped|overflowed" || true
````

**这次只看一件事**：输出里有没有 `SYNs to LISTEN sockets dropped` 或 `overflowed` 的数字。

可能看到类似：

````text
    12345 SYNs to LISTEN sockets dropped
    678 times the listen queue of a socket overflowed
````

有数字，说明本机历史上出现过 SYN 队列溢出。**数值持续增长才说明当前有问题，只看一眼没用。** 如果输出为空，可能是 `netstat` 不可用或字段名不同，**需按环境核实**。

想观察增长，可以隔几秒再跑一次同样的命令，比较两次数字。**只看一件事：数字有没有变大。**

<br>

### 易错点与边界

坑一：**只增大 `tcp_max_syn_backlog` 不一定有效**，半连接队列还受 `somaxconn`、应用 `listen` 的 `backlog` 影响。

坑二：**`tcp_tw_recycle` 在 NAT 下不安全**，Linux 4.12 后已移除，现代内核不用再调这个参数。

坑三：**全连接队列满默认丢 ACK，不回 RST**，客户端可能一直重试；开 `tcp_abort_on_overflow` 才会回 RST。

坑四：**syncookies 不是万能**，它绕过半连接队列，但全连接队列满仍然会出问题。

<br>

### 一句话本质

**SYN 被丢，多半是队列满了或被历史参数误杀；syncookies 是半连接队列的应急旁路，但不解决全连接队列满。**

<br>

### 自测

半连接队列满了，开启 syncookies 后 SYN 还会被丢吗？全连接队列满和半连接队列满，客户端看到的现象一样吗？

<details><summary>答案</summary>开启 syncookies 后，半连接队列满时内核可以用 cookie 代替队列项，SYN 通常不会被丢，但这不是绝对的，还要看具体实现和负载。全连接队列满和半连接队列满，客户端看到的现象可能一样：connect 超时或重试，因为服务端都没完成握手。区别在内核统计和队列类型上。</details>

<br>

<br>

## 知识点 2：ESTABLISHED 收到 SYN、TIME_WAIT 收到 SYN——Challenge ACK 与复用
***
### 问题

ESTABLISHED 状态收到同四元组 SYN 怎么办？TIME_WAIT 收到 SYN 一定回 RST 吗？为什么有时能复用连接，有时又回 ACK 让客户端 RST？

**你可能会想**：都 ESTABLISHED 了还收到 SYN，肯定是攻击吧？直接丢不就行了？

不一定。**旧连接的 SYN 可能因为网络乱序，延迟到新连接建立后才到达。** 直接丢会让客户端一直等，Challenge ACK 是更聪明的做法：告诉对方“你发的 SYN 不对，我真正想要的是这个序号”。

<br>

### 核心概念与方案

先把这两种场景换成你熟悉的场景：**Challenge ACK 像“你拿错号了，我告诉你正确的号是多少”；TIME_WAIT 收到 SYN 像“旧业务还没彻底结束，新业务能不能开始，要看新旧是否合法”。**

**ESTABLISHED 收到 SYN**（通常是旧 SYN 乱序到达）：服务端回 **Challenge ACK**，携带正确序列号和确认号；客户端发现确认号不是自己期望的，回 RST，服务端释放连接。Challenge ACK 的作用是“你发的 SYN 不对，我告诉你我真正想要哪个序号”，而不是直接丢包让客户端干等。

**TIME_WAIT 收到 SYN**：看 SYN 是否合法。双方开时间戳时，合法 SYN 要求 SYN 的序列号大于期望下一个序列号，且时间戳更大；合法则复用四元组，跳过 2MSL 进入 SYN_RECV；非法则回第四次挥手 ACK，客户端发现不对回 RST。

**TIME_WAIT 收到 RST**：`tcp_rfc1337=0` 会提前释放；为 1 则丢弃 RST。

所以不是所有 TIME_WAIT 收到 SYN 都回 RST；合法 SYN 可以复用连接。

<br>

### 最小案例：用文字流程复现三种异常处理

不需要抓包，看下面这段流程就能理解内核的处理逻辑：

````text
Established 收到旧 SYN → Challenge ACK → 客户端发现 Ack 不对 → RST
TIME_WAIT 收到非法 SYN → 回上次 ACK → 客户端 RST
TIME_WAIT 收到合法 SYN → 复用四元组，进入 SYN_RECV
````

**这次只看一件事**：三种情况最后是建立新连接、回 RST，还是复用。

- Established 收到旧 SYN：不回 SYN+ACK，回 Challenge ACK，客户端发现不对回 RST。
- TIME_WAIT 收到非法 SYN：回上次 ACK，客户端 RST。
- TIME_WAIT 收到合法 SYN：复用四元组，进入 SYN_RECV。

**关键观察点**：Challenge ACK 不是 SYN+ACK，它不建立新连接，只是纠正序号。

<br>

### 易错点与边界

坑一：**不是所有 TIME_WAIT 收到 SYN 都回 RST**，合法 SYN 可以复用连接。

坑二：**Challenge ACK 不是 SYN+ACK**，它不建立新连接，只是告诉对方当前期望的序号。

坑三：**`tcp_rfc1337` 影响 TIME_WAIT 是否被 RST 提前结束**，默认 0 会提前释放，开 1 更严格。

坑四：**旧 SYN 乱序到达是常见现象**，不是攻击；Challenge ACK 让客户端自己纠正，避免服务端误建连。

<br>

### 一句话本质

**Challenge ACK 是“你发的 SYN 不对，我告诉你我真正想要哪个序号”；TIME_WAIT 收到 SYN 要看新旧是否合法，合法可以复用，非法回 ACK 让客户端 RST。**

<br>

### 自测

Established 收到 SYN 后为什么回 Challenge ACK 而不是直接丢？TIME_WAIT 收到 SYN 在什么情况下可以复用连接？

<details><summary>答案</summary>Challenge ACK 是为了告诉客户端“你发的 SYN 序号不对，我当前期望的是这个序号”，让客户端自己发现并回 RST，而不是直接丢包让客户端一直等。TIME_WAIT 收到 SYN 时，如果 SYN 合法（序列号大于期望下一个序列号，时间戳更大），可以复用四元组，跳过 2MSL 进入 SYN_RECV；非法则回上次 ACK，让客户端 RST。</details>

<br>

<br>

## 知识点 3：进程崩溃、主机宕机、拔网线——对端能看到什么
***
### 问题

一端异常后，另一端怎么感知？进程崩溃和主机断电，对端看到的现象一样吗？拔网线后连接会立即断开吗？Keepalive 到底靠不靠谱？

**你可能会想**：连接断了，TCP 应该能立刻知道吧？

不一定。**TCP 只有在“收到报文”或“发数据超时”时才知道出问题。** 如果对方进程崩溃、主机断电、网线被拔，而你这端又没有数据要发，TCP 可能一直以为连接还活着。

<br>

### 核心概念与方案

先把三种异常换成你熟悉的场景：**进程崩溃像店员下班前打了招呼；主机断电像店突然关门；拔网线像店还在但路断了。**

三种异常，内核知道的程度不同：

- **进程崩溃**：内核回收进程资源，会发 FIN，通常能完成四次挥手。对端看到正常的 FIN，连接正常关闭。
- **主机宕机/断电**：无法发 FIN；若没有数据要发且没开 TCP Keepalive，对端可能一直 ESTABLISHED，以为连接还活着。
- **拔网线**：不改变内核 TCP 状态；有数据发送时靠超时重传，重传超阈值后断开；没数据且没 Keepalive 则连接可能一直存在。

TCP Keepalive 默认 2 小时才启动，适合兜底；应用层可自己做心跳。HTTP Keep-Alive 是应用层长连接，不是 TCP Keepalive。

一句话：**进程崩溃内核会发 FIN；主机宕机/拔网线内核不知道，只能靠数据重传或 Keepalive 探测。**

<br>

### 最小实验：只看 Keepalive 默认参数是不是 2 小时起步

环境前提：Linux；已安装 `sysctl`。下面只读参数，不改配置。

````bash
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_intvl net.ipv4.tcp_keepalive_probes
````

**这次只看一件事**：`tcp_keepalive_time` 是不是 7200 秒（2 小时）。

可能看到类似：

````text
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
````

7200 秒就是 2 小时。**默认要空闲 2 小时才开始探测，75 秒一次，9 次失败才判死。** 这意味着对方主机断电后，最坏情况下你要等 2 小时 + 75×9 秒左右才能发现连接死了。

不同系统、不同内核版本默认值可能不同，**需按环境核实**。

<br>

### 易错点与边界

坑一：**ESTABLISHED 不代表对端一定活着**，可能只是没有数据要发，也没开 Keepalive。

坑二：**TCP Keepalive 和 HTTP Keep-Alive 不是一回事**；前者是 TCP 层探测，后者是 HTTP 层连接复用。

坑三：**Keepalive 要应用设置 `SO_KEEPALIVE` 才生效**，不是内核默认对所有连接都开。

坑四：**拔网线不会立即改变内核 TCP 状态**，没有数据发送时连接可能一直挂着。

坑五：**Keepalive 默认 2 小时才启动**，不能指望它快速发现故障；应用层心跳更及时。

<br>

### 一句话本质

**进程崩溃内核会发 FIN，对端能正常关闭；主机宕机/拔网线内核不知道，只能靠数据重传或 Keepalive 探测，而 Keepalive 默认 2 小时才启动，线上常需应用层心跳。**

<br>

### 自测

拔掉网线后，原本 TCP 连接一定立即断开吗？进程崩溃和主机断电，对端看到的 TCP 行为有什么关键区别？

<details><summary>答案</summary>拔网线后连接不一定立即断开。如果没有数据发送且没开 Keepalive，内核不知道网络已断，连接可能一直处于 ESTABLISHED。进程崩溃时内核会发 FIN，对端通常能正常完成四次挥手；主机断电无法发 FIN，对端只能靠数据重传超时或 Keepalive 探测来发现，默认可能要等很久。</details>

<br>

<br>

## 完整可复现示例：看队列统计、Keepalive 参数，并用 C++ 设置 SO_KEEPALIVE
***
你需要 Linux 或 WSL，已安装 `netstat`（`net-tools`）、`sysctl`、`g++`。下面每个步骤只看一件事，不展开无关细节。

````bash
set -e

echo "=== 1. 队列溢出统计 ==="
netstat -s 2>/dev/null | grep -i -E "SYNs to LISTEN sockets dropped|overflowed" || true

echo "=== 2. TCP Keepalive 默认参数 ==="
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_intvl net.ipv4.tcp_keepalive_probes
````

可能的输出：

````text
=== 1. 队列溢出统计 ===
    12345 SYNs to LISTEN sockets dropped
    678 times the listen queue of a socket overflowed

=== 2. TCP Keepalive 默认参数 ===
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
````

**关键观察点**：第 1 步看有没有溢出统计，第 2 步看 `tcp_keepalive_time` 是不是 7200。

再用 C++ 给套接字打开 `SO_KEEPALIVE`，并读取当前参数。保存为 `keepalive_demo.cpp`：

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

    int keepalive = 1;
    if (setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE,
                   &keepalive, sizeof(keepalive)) < 0) {
        std::cerr << "setsockopt SO_KEEPALIVE failed\n";
        close(sock);
        return 1;
    }

    int idle = 0, intvl = 0, cnt = 0;
    socklen_t len = sizeof(idle);

    if (getsockopt(sock, IPPROTO_TCP, TCP_KEEPIDLE, &idle, &len) == 0) {
        std::cout << "TCP_KEEPIDLE = " << idle << "s\n";
    }

    len = sizeof(intvl);
    if (getsockopt(sock, IPPROTO_TCP, TCP_KEEPINTVL, &intvl, &len) == 0) {
        std::cout << "TCP_KEEPINTVL = " << intvl << "s\n";
    }

    len = sizeof(cnt);
    if (getsockopt(sock, IPPROTO_TCP, TCP_KEEPCNT, &cnt, &len) == 0) {
        std::cout << "TCP_KEEPCNT = " << cnt << "\n";
    }

    close(sock);
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 keepalive_demo.cpp -o keepalive_demo
./keepalive_demo
````

预期输出类似：

````text
TCP_KEEPIDLE = 7200s
TCP_KEEPINTVL = 75s
TCP_KEEPCNT = 9
````

**关键观察点**：`TCP_KEEPIDLE` 是不是 7200 秒。

验证结果：默认 Keepalive 很慢；线上常需应用层心跳；队列统计可判断 SYN/accept 队列是否溢出。`SO_KEEPALIVE` 只是打开开关，具体空闲时间、间隔、次数可以用 `TCP_KEEPIDLE`、`TCP_KEEPINTVL`、`TCP_KEEPCNT` 按连接调。

注意：不同系统、不同内核版本输出可能不同；`netstat -s` 字段名也可能有差异；修改 Keepalive 参数需要权限，示例只读不改，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| `connect` 一直超时，服务端像没收到 SYN | SYN 队列是否溢出、防火墙 | `netstat -s \| grep -i "SYNs to LISTEN"`；抓包看服务端有没有 SYN | 半连接队列满会丢 SYN；防火墙也可能拦 SYN |
| 客户端 `connect` 成功但很快被 RST | 全连接队列是否满、`tcp_abort_on_overflow` | `netstat -s \| grep overflowed`；查内核参数 | 全连接队列满默认丢 ACK，开 `tcp_abort_on_overflow=1` 会回 RST |
| ESTABLISHED 收到 SYN 后连接被重置 | 是否旧 SYN 乱序、Challenge ACK | 抓包看 Challenge ACK 和 RST | 旧 SYN 触发 Challenge ACK，客户端发现不对回 RST |
| TIME_WAIT 收到 SYN 后复用连接 | SYN 是否合法、时间戳 | 抓包看 SYN 序列号和时间戳 | 合法 SYN 可以复用四元组，跳过 2MSL；非法回 ACK 让客户端 RST |
| 对端进程崩溃，连接正常关闭 | 是否收到 FIN | `ss -tan` 看状态；抓包看 FIN | 进程崩溃内核会发 FIN，对端看到正常关闭 |
| 对端断电，连接一直 ESTABLISHED | 是否开了 Keepalive、是否发数据 | `ss -tan` 看状态；查 Keepalive 参数 | 没有 FIN，也没有数据重传，连接可能一直挂着 |
| 拔网线后连接没断 | 是否有数据发送、Keepalive | `ss -tan`；`sysctl net.ipv4.tcp_keepalive_time` | 没数据且没 Keepalive，内核不知道网络断了，连接可能一直存在 |
| Keepalive 发现故障很慢 | 默认参数 | `sysctl net.ipv4.tcp_keepalive_time` | 默认 2 小时才启动，线上常需应用层心跳 |
| 大量 CLOSE_WAIT | 应用是否忘记 close | `ss -tan \| grep CLOSE-WAIT` | 收到 FIN 后应用没 `close`，连接停在 CLOSE_WAIT |

<br>

<br>

## 常见误区
***
- **以为服务端没回 SYN+ACK 就一定是网络丢包**：可能是服务端队列满，内核直接丢了 SYN。
- **以为 `tcp_syncookies` 能解决所有队列问题**：它只绕过半连接队列，全连接队列满仍然会出问题。
- **以为 ESTABLISHED 收到 SYN 直接丢**：内核回 Challenge ACK 纠正序号，不是直接丢。
- **以为 TIME_WAIT 收到 SYN 一定回 RST**：合法 SYN 可以复用连接，非法才回 ACK 让客户端 RST。
- **以为进程崩溃和主机断电对端现象一样**：进程崩溃会发 FIN，主机断电不会。
- **以为拔网线连接会立即断开**：没有数据且没 Keepalive，内核不知道网络断了。
- **把 TCP Keepalive 和 HTTP Keep-Alive 混了**：前者是 TCP 层探测，后者是 HTTP 层连接复用。
- **以为 Keepalive 能快速发现故障**：默认 2 小时才启动，线上常需应用层心跳。

<br>

<br>

## 自测三问
***
1. Established 收到旧 SYN 回什么？客户端随后做什么？TIME_WAIT 收到 SYN 一定回 RST 吗？
2. 进程崩溃和主机宕机在 TCP 层面有什么关键区别？拔网线后连接会立即断开吗？
3. TCP Keepalive 默认多久才启动？为什么线上常需应用层心跳？TCP Keepalive 和 HTTP Keep-Alive 有什么区别？

<details><summary>参考答案</summary>1. Established 收到旧 SYN 回 Challenge ACK，携带正确序号，客户端发现确认号不对回 RST。TIME_WAIT 收到 SYN 不一定回 RST，合法 SYN 可以复用四元组进入 SYN_RECV，非法才回上次 ACK 让客户端 RST。2. 进程崩溃内核会发 FIN，对端通常看到正常关闭；主机宕机无法发 FIN，对端只能靠数据重传超时或 Keepalive 发现，默认可能等很久。拔网线不会立即改变内核 TCP 状态，没数据且没 Keepalive 时连接可能一直存在。3. TCP Keepalive 默认 7200 秒（2 小时）才启动，75 秒一次，9 次失败才判死，发现故障很慢，所以线上常需应用层心跳。TCP Keepalive 是 TCP 层探测，HTTP Keep-Alive 是 HTTP 层连接复用，两者不是一回事。</details>

<br>

<br>

## 总结
***
**TCP 异常场景核心看状态和报文合法性：SYN 可能因队列满被丢；ESTABLISHED 收到旧 SYN 用 Challenge ACK 纠正；TIME_WAIT 收到 SYN 看新旧是否合法；进程崩溃会发 FIN，主机宕机/拔网线内核不知道，只能靠数据重传或 Keepalive 兜底，而 Keepalive 默认 2 小时才启动，线上常需应用层心跳。**

<br>

<br>

## 参考资料
***
- RFC 9293：Transmission Control Protocol
- RFC 1122：Requirements for Internet Hosts — Communication Layers
- RFC 5961：Improving TCP's Robustness to Blind In-Window Attacks（Challenge ACK 相关）
- RFC 1337：TIME-WAIT Assassination Hazards in TCP
- `man 7 tcp`、`man 7 socket`
- 需按环境核实：`tcp_tw_recycle` 在 Linux 4.12 后已移除；`netstat -s` 字段名和 `sysctl` 默认值因内核版本而异；`TCP_KEEPIDLE`、`TCP_KEEPINTVL`、`TCP_KEEPCNT` 的可用性和默认值需按系统核实；TIME_WAIT 收到 SYN 的复用条件依赖时间戳和具体实现。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>