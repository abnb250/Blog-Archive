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
TCP 性能不只靠理论，还靠内核参数。**握手慢、accept 队列满、TIME_WAIT 多、缓冲区太小、带宽时延积不匹配，分别怎么优化？**  
先别急着背 `tcp_syn_retries`、`tcp_tw_reuse`、`tcp_wmem` 这些参数名，先看几个具体场景：你压测服务时，客户端 `connect` 一直超时，服务端 `ss -lnt` 看到 `Send-Q` 满了，是哪个队列的问题？短连接服务跑一段时间，`ss -tan` 里一堆 `TIME_WAIT`，能不能直接调小？跨机房传大文件，带宽明明很大，吞吐却上不去，是不是缓冲区没跟上？这一章咱们就按“握手挥手怎么优化、数据传输怎么调”这条线走。

<br>

<br>

## 知识点 1：三次握手与四次挥手优化
***
### 问题
建连和断连哪里会慢？客户端 SYN 重传、服务端 SYN+ACK 重传、半连接队列、全连接队列、TIME_WAIT，分别该看哪些参数？TFO 又省了哪一段？

<br>

### 方案
**三次握手**：  
- 客户端 SYN 重传次数看 `tcp_syn_retries`；服务端 SYN+ACK 重传看 `tcp_synack_retries`。  
- 半连接队列受 `tcp_max_syn_backlog`、`somaxconn`、应用 `listen` 的 `backlog` 共同影响；SYN 攻击可开 `tcp_syncookies=1` 应急。  
- 全连接队列满默认丢 ACK，`tcp_abort_on_overflow=1` 回 RST；通常设 0 更利于突发流量，让客户端重试而不是直接失败。  
- **TCP Fast Open（TFO）** 可在第二次以后连接绕过三次握手携带数据，减少 1 RTT，但需双方支持，且首次连接仍需正常握手。  
**四次挥手**：  
- 主动方 FIN 重传看 `tcp_orphan_retries`；`close` 后的 FIN_WAIT2 受 `tcp_fin_timeout` 限制。  
- TIME_WAIT 可用 `tcp_max_tw_buckets` 限制数量；客户端可用 `tcp_tw_reuse=1` + `tcp_timestamps=1` 复用超过 1 秒的 TIME_WAIT 端口。  
- `tcp_tw_recycle` 在 NAT 下不安全，Linux 4.12 后已取消，现代内核不用再调。  
一句话：握手优化是少重试、扩队列、防攻击、能 TFO；挥手优化是控 FIN 重传、管 TIME_WAIT、慎用暴力复用。

<br>

### 最小示例
```bash
sysctl net.ipv4.tcp_syn_retries net.ipv4.tcp_synack_retries
sysctl net.ipv4.tcp_tw_reuse net.ipv4.tcp_timestamps
ss -lnt
```

输出可能类似：

```text
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 5
net.ipv4.tcp_tw_reuse = 2
net.ipv4.tcp_timestamps = 1

State  Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 0      128    0.0.0.0:22         0.0.0.0:*
```

结论：`ss -lnt` 的 `Send-Q` 接近上限时，全连接队列可能满了；`tcp_tw_reuse=2` 表示只对客户端连接生效。

<br>

### 一句话本质
**握手优化是少重试、扩队列、防攻击、能 TFO；挥手优化是控 FIN 重传、管 TIME_WAIT、慎用暴力复用。**

<br>

### 易错点
坑一：**`tcp_tw_reuse` 只对客户端 `connect` 有效**，服务端开它不会复用服务端主动关闭的 TIME_WAIT。  
坑二：**`tcp_tw_recycle` 在 NAT 下会误杀 SYN**，Linux 4.12 后已移除，别再从旧文章里抄这个参数。  
坑三：**TIME_WAIT 不是越少越好**，它是正常关闭流程的一部分；盲目调小可能让旧报文干扰新连接。  
坑四：**全连接队列满默认丢 ACK，不回 RST**；开 `tcp_abort_on_overflow` 回 RST 会让客户端更快失败，不一定更好。  
坑五：**TFO 不是所有客户端和服务端都支持**，中间设备也可能干扰；不能把它当成万能加速。

<br>

### 自测
为什么 `tcp_tw_reuse` 默认关闭？只放行 21 端口为什么可能传不了文件——哦不对，那是 FTP；这里问：TIME_WAIT 很多时，为什么不能无脑调小 `tcp_max_tw_buckets`？

<br>

<br>

## 知识点 2：数据传输优化与缓冲区
***
### 问题
带宽很大但 TCP 传不快，为什么？带宽时延积和缓冲区是什么关系？`tcp_wmem`、`tcp_rmem`、`tcp_mem` 分别管什么？程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 为什么可能反而坏事？

<br>

### 方案
窗口上限受缓冲区影响。**带宽时延积 BDP = 带宽 × RTT**，表示网络中“飞行中”的字节数。发送缓冲区太小，窗口上不去，带宽用不满；太大又可能过载丢包。  
- `tcp_wmem` 配发送缓冲区 min/default/max；  
- `tcp_rmem` 配接收缓冲区；  
- `tcp_mem` 是 TCP 总内存页面数，单位是页面，不是字节。  
接收缓冲区动态调节需 `tcp_moderate_rcvbuf=1`；窗口扩大 `tcp_window_scaling=1` 可让窗口超过 64KB。  
程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能关闭动态调节，通常不建议；让内核按 BDP 和内存压力动态调通常更稳。  
一句话：缓冲区要围绕带宽时延积调，太小跑不满，太大易过载，交给内核动态调通常更稳。

<br>

### 最小示例
```text
带宽 100MB/s，RTT 10ms → BDP = 1MB
发送缓冲区远小于 1MB，窗口上不去，带宽用不满。
```

结论：高带宽高延迟网络需要更大的窗口和缓冲区，但不是越大越好，要结合 BDP 和内存成本。

<br>

### 一句话本质
**缓冲区要围绕带宽时延积调：太小跑不满，太大易过载，交给内核动态调通常更稳。**

<br>

### 易错点
坑一：**`tcp_wmem`/`tcp_rmem` 单位是字节，`tcp_mem` 单位是页面**，别把三者单位混了。  
坑二：**不是缓冲区越大越好**，太大可能增加排队延迟和内存压力，甚至掩盖拥塞。  
坑三：**带宽时延积是设置上限的重要参考**，不是唯一依据；还要看应用读写速度、并发连接数。  
坑四：**程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能关闭动态调节**，尤其在高并发服务里，可能让内核没法按压力自适应。  
坑五：**窗口扩大 `tcp_window_scaling=1` 是协商选项**，不是单方面开了就一定能用大窗口，还要对端支持。

<br>

### 自测
为什么高带宽高延迟网络需要更大的窗口和缓冲区？如果缓冲区远小于 BDP，会发生什么？

<br>

<br>

## 完整可复现示例：查看 TCP 优化参数
***
这个示例用 `sysctl` 和 `ss` 查看握手、挥手、缓冲区、监听队列相关参数。注意：不同内核版本、不同发行版输出可能不同；这里只读不改，避免影响本机网络。

```bash
set -e

echo "=== 1. 握手/挥手 ==="
sysctl net.ipv4.tcp_syn_retries \
       net.ipv4.tcp_synack_retries \
       net.ipv4.tcp_tw_reuse \
       net.ipv4.tcp_timestamps 2>/dev/null || true

echo "=== 2. 缓冲区 ==="
sysctl net.ipv4.tcp_wmem \
       net.ipv4.tcp_rmem \
       net.ipv4.tcp_mem 2>/dev/null || true

echo "=== 3. 监听队列 ==="
if command -v ss >/dev/null 2>&1; then
  ss -lnt | head
else
  netstat -lnt | head
fi
```

输出/现象/结论可能类似：

```text
=== 1. 握手/挥手 ===
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 5
net.ipv4.tcp_tw_reuse = 2
net.ipv4.tcp_timestamps = 1

=== 2. 缓冲区 ===
net.ipv4.tcp_wmem = 4096 16384 4194304
net.ipv4.tcp_rmem = 4096 87380 6291456
net.ipv4.tcp_mem = 94677 126238 189354

=== 3. 监听队列 ===
State  Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 0      128    0.0.0.0:22         0.0.0.0:*
```

验证结果：优化 TCP 要分握手、挥手、数据传输三阶段对症下药，不能只背一个参数。`ss -lnt` 的 `Recv-Q`/`Send-Q` 接近上限时，要怀疑队列溢出；`tcp_wmem`/`tcp_rmem` 是三元组，分别是最小、默认、最大值。  
注意：`sysctl` 某些参数在旧内核或容器环境里可能不存在，加了 `|| true` 避免脚本中断；容器里看到的可能是宿主机参数或被 namespace 隔离后的值。

<br>

<br>

## 自测三问
***
1. 半连接队列和全连接队列分别受哪些参数影响？`ss -lnt` 里 `Send-Q` 满了，更可能是哪个队列的问题？  
2. `tcp_tw_reuse` 为什么只适合客户端？TIME_WAIT 很多时，为什么不能无脑调小 `tcp_max_tw_buckets`？  
3. 带宽时延积和 TCP 缓冲区有什么关系？为什么程序里直接设 `SO_SNDBUF`/`SO_RCVBUF` 可能反而坏事？

<br>

<br>

## 一句话总结全章
***
**TCP 优化分三段：握手少重试扩队列防攻击，挥手控 FIN 重传并谨慎管理 TIME_WAIT，数据传输按带宽时延积调缓冲区；TFO 能省 RTT，但需双方支持。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>