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
ping 只能告诉你通不通，**怎么知道数据包经过哪些路由器？怎么知道路径上最小 MTU 是多少？**  
先别急着背“TTL、ICMP 超时、DF 位”这些词，先看几个具体场景：你 ping 一个目标能通，但中间到底走了哪几跳？为什么有时大包发不出去，小包却正常？你怀疑某条链路 MTU 不是 1500，怎么让路由器主动告诉你“我这儿只能过 1400”？这一章咱们就按“用 TTL 逐跳探路、用不分片探 MTU”这条线走。

<br>

<br>

## 知识点 1：用 TTL 追踪路径
***
### 问题
traceroute 怎么拿到沿途路由器 IP？为什么故意把 TTL 设得很小？怎么判断已经到达目标主机，而不是还在中间某跳？

<br>

### 方案
traceroute 的核心是**故意设置小 TTL，从 1 开始递增**。TTL 每经过一个路由器减 1，减到 0 路由器就丢弃该包，并回一个 **ICMP 时间超时**。这样 TTL=1 拿到第一跳，TTL=2 拿到第二跳，逐跳推进。  
Linux 的 traceroute 默认常用 **UDP 高端口**（大于 3000）发探测包；到达目标主机后，目标回 **ICMP 端口不可达**，表示路径追踪结束。Windows 的 `tracert` 默认用 ICMP Echo，行为类似但探测报文不同。  
所以 traceroute 不是“一次性看到整条路径”，而是靠一串“故意牺牲”的包，让每台路由器各报一次超时，拼出路径。

<br>

### 最小示例
```text
TTL=1 → 路由器1 回 ICMP 超时
TTL=2 → 路由器2 回 ICMP 超时
TTL=3 → 路由器3 回 ICMP 超时
...
目标主机回 ICMP 端口不可达 → 结束
```

结论：每一跳都是靠 TTL 减到 0 触发 ICMP 超时暴露出来的；目标主机用端口不可达告诉你“别再探了”。

<br>

### 一句话本质
**traceroute 用 TTL 逐跳“故意牺牲”，让每台路由器报一次超时，从而画出路径。**

<br>

### 易错点
坑一：**有的路由器不回 ICMP**，所以中间会显示 `*`，不代表那一跳不存在，可能只是被禁用或限速。  
坑二：**Windows 的 `tracert` 默认用 ICMP，Linux traceroute 常用 UDP**，两者探测方式不同，结果可能略有差异。  
坑三：**路径可能不对称**，去程和回程经过的路由器不一定相同，traceroute 只反映去程（或探测方向）的路径。  
坑四：**目标主机可能不回端口不可达**，比如防火墙拦截，导致 traceroute 一直探到最大跳数也没结束。  
坑五：**TTL 超时是路由器回的，端口不可达是目标主机回的**，两者别混。

<br>

### 自测
traceroute 怎么知道已经到达目标主机？如果中间某跳显示 `*`，说明什么？

<br>

<br>

## 知识点 2：路径 MTU 发现
***
### 问题
怎么知道路径上最小 MTU？为什么大包不分片就过不去？路由器遇到超 MTU 又不能分片的包，会怎么通知发送方？

<br>

### 方案
路径 MTU 发现（PMTUD）的思路是：发送方把 IP 首部的**分片禁止位（DF）设为 1**，发较大包。途中路由器遇到超过 MTU 且不能分片的包，会丢弃并回 **ICMP 目标不可达，代码 4：需要分片但设置了不分片位**，同时可携带该链路的 MTU。发送方逐步减小包大小，最终找到能通过的 MTU。  
关键点：**路由器不是默默丢弃**，而是用 ICMP 代码 4 告诉发送方“我这儿 MTU 只有这么大，你别发这么大还不让分片”。  
IPv6 中间路由器默认不分片，更依赖 PMTUD；如果 ICMP 被防火墙拦截，可能出现“PMTUD 黑洞”：小包能过，大包一直失败。

<br>

### 最小示例
```text
发 1500 字节不分片包 → 某跳 MTU 1400 → 回 ICMP 代码 4，MTU=1400
发送方减小到 1400 → 通过
```

结论：PMTUD 就是“不许分片，撞到谁太小就让谁报 MTU，再慢慢调小”。

<br>

### 一句话本质
**路径 MTU 发现就是“不许分片，撞到谁太小就让谁报 MTU，再慢慢调小”。**

<br>

### 易错点
坑一：**ICMP 被防火墙拦截会导致 PMTUD 失败**，发送方一直不知道包为什么被丢，形成黑洞。  
坑二：**分片禁止位为 1 时路由器不能分片**，只能丢弃并回 ICMP；所以 DF 位是 PMTUD 的前提。  
坑三：**路径 MTU 可能随路由变化**，不是探到一次就永久固定；网络路径变了要重新发现。  
坑四：**ICMP 代码 4 是 IPv4 的说法**，IPv6 用 Packet Too Big 消息，机制类似但报文不同。  
坑五：**PMTUD 依赖 ICMP 反馈**，如果中间设备把 ICMP 全禁了，只能靠应用层猜或保守设置 MSS。

<br>

### 自测
ICMP 目标不可达代码 4 表示什么？如果 ICMP 被防火墙拦截，PMTUD 会怎样？

<br>

<br>

## 完整可复现示例：运行 traceroute 并观察路径
***
这个示例用系统自带的 `traceroute`（或 `tracert`）观察逐跳路径，重点看每一跳的 IP、延迟和 `*` 的含义。  
注意：不同系统、不同网络环境下输出不同；没有 `traceroute` 可以改用 `tracert`，或者用 `mtr`。

```bash
set -e

echo "=== traceroute 示例 ==="
if command -v traceroute >/dev/null 2>&1; then
  traceroute -n -m 15 example.com 2>/dev/null || true
elif command -v tracert >/dev/null 2>&1; then
  tracert -d -h 15 example.com 2>/dev/null || true
else
  echo "traceroute/tracert 不可用，可安装后重试"
fi
```

输出/现象/结论可能类似：

```text
traceroute to example.com (93.184.216.34), 15 hops max
 1  192.168.1.1  1.234 ms  1.100 ms  1.050 ms
 2  * * *
 3  10.10.0.1  5.678 ms  5.432 ms  5.210 ms
 ...
15  * * *
```

验证结果：每一行是一跳；`*` 表示该跳未回 ICMP 超时，可能是路由器禁 ICMP 或限速；延迟列是该跳往返时间。  
注意：`-n` 表示不解析域名，`-m 15` 限制最大跳数；不同系统参数名可能不同。路径 MTU 发现不直接显示在 traceroute 输出里，它依赖 ICMP 代码 4 的反馈，通常用 `ping -M do -s <size>` 或 `tracepath` 观察。  
如果想验证 PMTUD，可以在 Linux 上试：

```bash
# 发送不分片的大包，观察是否收到 ICMP 代码 4
ping -M do -s 1472 -c 1 example.com
```

`-s 1472` 是 ICMP 数据部分大小，加上 8 字节 ICMP 头和 20 字节 IP 头正好 1500；如果路径 MTU 更小，可能看到“Frag needed and DF set”之类提示。

<br>

<br>

## 自测三问
***
1. traceroute 利用 ICMP 哪类报文逐跳发现路由器？为什么中间会出现 `*`？  
2. 怎么判断 UDP 探测包到达了目标主机？Windows `tracert` 和 Linux `traceroute` 默认探测方式有什么不同？  
3. 路径 MTU 发现依赖哪个 ICMP 代码？如果 ICMP 被防火墙拦截，会出现什么问题？

<br>

<br>

## 一句话总结全章
***
**traceroute 用递增 TTL 让每跳路由器回 ICMP 超时，目标回端口不可达结束；路径 MTU 发现用“不分片”让路由器回 ICMP 代码 4，逐步调小包大小找到可通行 MTU。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>