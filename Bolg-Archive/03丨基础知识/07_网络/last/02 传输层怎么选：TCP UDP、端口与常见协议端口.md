# 02 传输层怎么选：TCP/UDP、端口与常见协议端口
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
网页和邮件不能丢数据，视频会议又更怕延迟。你可能会想：**TCP 可靠，那所有应用都用 TCP 不就行了？为什么视频会议还考虑 UDP？**

再往下想：TCP 和 UDP 的差别真的只是“可靠/不可靠”吗？DNS 明明说用 UDP，为什么区域传送又用 TCP？一台机器上 TCP 和 UDP 能用同一个端口号吗？

这一章咱们用一条问题链往前走：

1. 应用更怕丢，还是更怕慢？
2. TCP 和 UDP 各给出什么服务承诺？
3. 端口到底标识什么，为什么 TCP 和 UDP 可以同端口？
4. 常见协议为什么这样选？
5. 怎么动手验证，而不是只背结论？

<br>

<br>

## 优先级说明
***
这篇是传输层选型的地基，面试和项目都会反复用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- TCP 和 UDP 的核心区别：连接、可靠、按序、流量控制、拥塞控制、首部开销
- 端口的作用：IP 找主机，端口找进程，端口不是进程号
- TCP 和 UDP 的端口空间分开，同一端口号可以分别绑定
- DNS 为什么普通查询常用 UDP、区域传送用 TCP
- 知名端口只是默认约定，可以改

**了解即可（知道大概，不展开）：**
- QUIC 的帧格式和 0-RTT 细节
- FTP 主动/被动模式的具体协商过程
- RTP/RTCP 的协议细节

**主线**：应用怕丢还是怕慢 → TCP/UDP 给什么承诺 → 端口怎么找进程 → 具体协议按需求选。

<br>

<br>

## 知识点 1：TCP 和 UDP 不是“好协议/坏协议”，是两种服务承诺
***
### 问题

选 TCP 还是 UDP，到底看什么？TCP 可靠但不一定快，UDP 快但不一定可靠，怎么权衡？

**你可能会想**：TCP 可靠，那就无脑选 TCP；UDP 不可靠，能不用就不用。

但视频会议宁可丢几帧也不愿卡顿，DNS 查一个域名不想先握手三次。**“可靠”不是免费的，它要握手、要确认、要重传，这些都会带来延迟和开销。** 选型不是选“好”，是选“适合”。

<br>

### 核心概念与方案

先把服务承诺换成你熟悉的场景：

- **TCP 像挂号信。** 寄之前先确认对方在、能收；寄丢了会重寄；按顺序送到。代价是手续多、慢一点。
- **UDP 像明信片。** 直接扔进邮筒，不确认对方在不在，丢了不补，顺序也不保证。代价是可能丢，好处是快、省事。

落到协议上：

- **TCP**：面向连接、可靠、按序字节流，有流量控制和拥塞控制。适合不能丢数据的应用，比如网页、邮件、文件下载。
- **UDP**：无连接、尽最大努力交付、开销小、没有拥塞控制。适合实时性强、能容忍少量丢包，或者自己在应用层做可靠机制的应用。
- **加密不是 TCP 自带的**，要靠 TLS 等机制补。

**UDP 不等于“不能用”。** HTTP/3 就是基于 QUIC/UDP，在 UDP 上自己实现可靠和加密。可靠性没有消失，只是从 TCP 移到了上层。

<br>

### 最小实验：UDP 客户端不 connect 也能发

环境前提：Linux/macOS 或 Windows WSL，已安装 Python 3。下面只监听本机回环地址，不对外网开放。

先启动一个 UDP 回声服务，保存为 `udp_echo.py`：

````python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.bind(("127.0.0.1", 12000))
print("UDP echo listening on 127.0.0.1:12000")
data, addr = s.recvfrom(1024)
s.sendto(b"pong:" + data, addr)
s.close()
````

终端 1 运行：

````bash
python3 udp_echo.py
````

再写一个 UDP 客户端，保存为 `udp_client.py`：

````python
import socket

c = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
c.sendto(b"ping", ("127.0.0.1", 12000))
data, addr = c.recvfrom(1024)
print("client received:", data.decode(), "from", addr)
c.close()
````

终端 2 运行：

````bash
python3 udp_client.py
````

**这次只看一件事**：客户端代码里有没有调用 `connect`。

预期输出类似：

````text
client received: pong:ping from ('127.0.0.1', 12000)
````

客户端没有 `connect`，直接 `sendto` 就能发、能收。**UDP 不需要先建立连接。** 如果没收到回复，先检查服务端是否在运行、端口是否一致。

对比一下：TCP 客户端必须先 `connect` 成功才能 `send`，否则会报错。这个差别就是“面向连接”和“无连接”的最直观体现。

<br>

### 易错点与边界

坑一：**UDP 不是“坏协议”**，它只是把可靠性交给应用自己决定；QUIC/HTTP3 就是基于 UDP 做可靠传输的例子。

坑二：**TCP 可靠但不保证实时性**，拥塞时延迟也可能变大；低延迟场景不能只看“可靠”两个字。

坑三：**“可靠”不等于“安全”**，需要加密和身份验证时得上 TLS 等机制。

坑四：**UDP 的 `connect` 不等于 TCP 握手**。在编程接口里，UDP 的 `connect` 主要用来绑定默认对端、接收 ICMP 错误等，没有三次握手，**具体行为需按操作系统和编程接口核实**。

<br>

### 一句话本质

**TCP 像挂号信，UDP 像明信片；怕丢选挂号，怕慢常选明信片，重要内容再自己加锁。**

<br>

### 自测

在线会议和下载文件，如果只能二选一，哪个更可能优先选 UDP？为什么？

<details><summary>答案</summary>在线会议更可能优先选 UDP。它更怕延迟和卡顿，能容忍少量丢包；下载文件不能丢数据，通常选 TCP，或者选基于 UDP 但自己实现可靠传输的协议（如 QUIC）。</details>

<br>

<br>

## 知识点 2：端口是进程的“门牌号”，不是进程号
***
### 问题

一台主机同时跑浏览器、邮件客户端、Web 服务器，数据到了主机后怎么交给正确进程？为什么 TCP 和 UDP 可以同时用同一个端口号？

**你可能会想**：端口就是进程号吧？看到 8000 端口，就以为是 8000 号进程。

其实不是。端口和进程号是两套东西。这个问题不搞清楚，后面排查端口占用、理解 `ss` 输出都会卡。

<br>

### 核心概念与方案

先把场景换成你熟悉的：**IP 是小区地址，端口是门牌号，进程是住户。**

- IP 地址找到主机：数据送到哪台机器。
- 端口号找到主机上的进程：数据交给哪个应用。
- 传输层协议区分 TCP/UDP：用什么方式传。

更准确地说，一次通信常用“源 IP、源端口、目的 IP、目的端口、传输层协议”来区分不同的连接或数据流。服务器常使用知名端口：HTTP 80、HTTPS 443、DNS 53、SMTP 25/587、POP3 110/995、IMAP 143/993、FTP 控制 21。客户端端口通常由系统临时分配。

**端口不是进程号。** 端口是传输层用来标识应用进程的编号；进程号是操作系统内部标识。一个进程可以监听多个端口，一个端口也可能由多个连接共用。

<br>

### 最小实验：TCP 和 UDP 同时绑定同一端口

下面这段代码验证：同一 IP 上，TCP 12345 和 UDP 12345 可以同时绑定。保存为 `bind_both.py`：

````python
import socket

tcp = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
tcp.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
tcp.bind(("127.0.0.1", 12345))
tcp.listen(1)

udp = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
udp.bind(("127.0.0.1", 12345))

print("TCP and UDP both bound to 127.0.0.1:12345")
tcp.close()
udp.close()
````

运行：

````bash
python3 bind_both.py
````

**这次只看一件事**：程序有没有打印成功，还是报 `Address already in use`。

预期输出：

````text
TCP and UDP both bound to 127.0.0.1:12345
````

如果报 `Address already in use`，先检查 12345 端口是不是已经被别的程序占了，换个端口再试。**具体行为需按环境核实**，比如 `SO_REUSEADDR` 在不同系统上的语义有差异。

结论：**TCP 和 UDP 的端口号空间是分开的，同一端口号可以分别给 TCP 和 UDP 使用。**

<br>

### 易错点与边界

坑一：**DNS 不只走 UDP**。普通查询主要用 UDP 53，区域传送用 TCP 53；响应过大或被截断时也可能改用 TCP，**需按环境核实**。

坑二：**知名端口可以改**，排查问题时不能只看默认端口。看到 8080 不一定是 HTTP，看到 80 也不保证一定是 Web 服务。

坑三：**客户端端口是临时的**，服务端回信要用对方实际端口，不能假设客户端一定用某个固定端口。

坑四：**端口不是进程号**，用 `ss -p` 看进程信息可能需要权限；没有权限时看不到进程名，但仍能看到端口。

<br>

### 一句话本质

**IP 找到主机，端口找到进程，协议决定怎么传；端口不是进程号，TCP 和 UDP 的端口空间还彼此独立。**

<br>

### 自测

同一台电脑上浏览器访问 HTTPS 网站，本地浏览器端口和服务器 443 端口分别起什么作用？

<details><summary>答案</summary>服务器 443 端口是 HTTPS 服务监听的知名端口，用来让客户端找到服务器上的 Web 进程；本地浏览器端口通常是系统临时分配的源端口，用来标识这台主机上的浏览器连接，方便响应回来时交给正确进程。</details>

<br>

<br>

## 知识点 3：常见协议为什么这么选传输层
***
### 问题

HTTP、DNS、SMTP、视频会议，为什么有的用 TCP，有的用 UDP？DNS 不是“只用 UDP”吗，为什么又说区域传送用 TCP？

**你可能会想**：DNS 用 UDP 是因为 UDP 快，那干脆所有 DNS 都 UDP 不就行了？

但区域传送一次要传大量记录，丢一条就可能不完整；普通查询就查一个域名，握手反而拖慢。**同一个协议，不同场景可以选不同传输层。**

<br>

### 核心概念与方案

咱们按“应用更怕丢还是更怕慢”来看：

- **HTTP/1.1、HTTP/2**：通常 TCP，因为网页不能丢数据。
- **HTTPS**：TCP + TLS，可靠且加密。
- **HTTP/3**：QUIC/UDP，在 UDP 上实现可靠、加密和多路复用。
- **DNS 普通查询**：通常 UDP 53，查询短、希望快；区域传送用 TCP 53，因为数据量大且要可靠。
- **SMTP/POP3/IMAP**：TCP，邮件不能丢。
- **视频会议/直播**：常 UDP/RTP，低延迟优先，能容忍少量丢包。
- **FTP**：TCP，控制连接 21；主动模式数据连接常用 20，被动模式是服务器随机端口。

<br>

### 最小实验：DNS 默认查询和强制 TCP 查询都能工作

环境前提：已安装 `dig`（常见于 `dnsutils` 或 `bind-utils` 包）。如果没装，可用 `getent hosts` 观察默认解析，但无法强制 TCP。

先看默认查询：

````bash
dig example.com A +short
````

再用 `+tcp` 强制走 TCP：

````bash
dig +tcp example.com A +short
````

**这次只看一件事**：两次命令是不是都返回了 A 记录。

预期输出类似：

````text
93.184.216.34
93.184.216.34
````

两次都能解析，说明 DNS 协议本身支持 UDP 和 TCP 两种传输层。默认查询通常走 UDP 53，`+tcp` 强制走 TCP 53。**具体 IP 会随站点和网络环境变化，需按环境核实**；看不到输出时先检查网络和 `dig` 是否可用。

想亲眼看到传输层差异，可以在有权限时抓包：

````bash
# 需要权限，按环境核实
sudo tcpdump -ni any port 53 -c 5
````

然后在另一个终端分别执行默认查询和 `+tcp` 查询。没有抓包权限也不影响结论：**DNS 默认选择 UDP 是为了短查询快，区域传送和截断等场景才需要 TCP 的可靠传输。**

<br>

### 易错点与边界

坑一：**HTTP/3 不是“UDP 不可靠所以不能用”**，QUIC 在 UDP 上补了可靠性、加密和连接管理。

坑二：**DNS 不是永远 UDP**，区域传送用 TCP，普通查询也可能因响应过大、TC 截断标志而改用 TCP。

坑三：**FTP 的数据连接端口不总是 20**，主动模式服务器常用 20 连客户端，被动模式是服务器随机端口，具体看协商结果。

坑四：**协议选型不是永久不变**，HTTP/3 就是把可靠传输从 TCP 移到 QUIC/UDP 上的例子；但“可靠”这个需求没有消失，只是实现位置变了。

<br>

### 一句话本质

**协议选 TCP 还是 UDP，不是看谁“高级”，而是看应用更怕丢还是更怕慢；可靠性也可以在上层补。**

<br>

### 自测

为什么 DNS 普通查询常用 UDP，而区域传送常用 TCP？

<details><summary>答案</summary>普通查询报文短，UDP 开销小、不用握手，追求快；区域传送数据量大，需要可靠、有序、完整的传输，所以用 TCP。普通查询响应过大或被截断时也可能转 TCP。</details>

<br>

<br>

## 完整可复现示例：端口、TCP/UDP 同端口、DNS 传输层一次串起来
***
下面把前面的实验串起来。你需要 Linux/macOS 或 WSL，已安装 Python 3；`dig` 和 `ss` 可选，没有时按注释降级。

````bash
set -e

echo "=== 1. 查看本机常见服务端口 ==="
if command -v ss >/dev/null 2>&1; then
  ss -tulnp 2>/dev/null | grep -E ':(53|80|443|25|110|143|21)\b' || true
else
  echo "ss 不可用，可改用 netstat -tulnp"
fi

echo "=== 2. TCP 和 UDP 同时绑定 12345 ==="
python3 - <<'PY'
import socket

tcp = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
tcp.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
tcp.bind(("127.0.0.1", 12345))
tcp.listen(1)

udp = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
udp.bind(("127.0.0.1", 12345))
print("TCP and UDP both bound to 127.0.0.1:12345")
tcp.close()
udp.close()
PY

echo "=== 3. DNS 默认查询与强制 TCP 查询 ==="
if command -v dig >/dev/null 2>&1; then
  echo "--- 默认（通常 UDP 53） ---"
  dig example.com A +short
  echo "--- 强制 TCP ---"
  dig +tcp example.com A +short
else
  echo "dig 不可用，可用 getent hosts example.com 观察默认解析"
  getent hosts example.com || true
fi
````

可能的输出/现象：

````text
=== 1. 查看本机常见服务端口 ===
（有服务时显示 LISTEN 行；没有则可能为空）

=== 2. TCP 和 UDP 同时绑定 12345 ===
TCP and UDP both bound to 127.0.0.1:12345

=== 3. DNS 默认查询与强制 TCP 查询 ===
--- 默认（通常 UDP 53） ---
93.184.216.34
--- 强制 TCP ---
93.184.216.34
````

**关键观察点**：第 1 步看有没有 `LISTEN` 行；第 2 步看有没有打印成功；第 3 步看两次查询是不是都返回了地址。

验证结果：端口帮助定位本机进程；TCP 和 UDP 的端口空间分开，同一端口号可以分别绑定；DNS 默认查询和强制 TCP 查询都能工作，传输层选择取决于场景和策略。

注意：`dig` 得到的 A 记录会随站点配置和网络环境变化；`ss -p` 显示进程信息可能需要权限；`dig +tcp` 强制 TCP 的行为需按 `dig` 版本核实。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| `curl: (7) Failed to connect` | 服务是否启动、端口是否对、监听地址 | `ss -ltnp \| grep ':端口'`；`curl -v` | 连接被拒绝常见于没进程监听、端口写错，或服务只监听 `127.0.0.1` 而你在别的机器访问 |
| `ping` 通但端口连不上 | 端口、服务、协议 | `curl -v`；`nc -vz 主机 端口`（若已安装） | ICMP 通只说明网络层可达，不代表 TCP/UDP 端口有服务 |
| 端口被占用 | 谁在监听 | `ss -ltnp`；`lsof -i :端口`（若已安装） | 换端口或停掉占用进程；`-p` 权限不足时可能看不到进程名 |
| TCP 和 UDP 同端口，一个能用另一个不能用 | 是否绑错协议 | 分别用 `ss -ltnp` 和 `ss -lunp` 查看 | TCP 和 UDP 端口空间分开，要确认服务实际监听的是 TCP 还是 UDP |
| DNS 解析慢或失败 | 先分清解析还是连接 | `getent hosts 域名`；`dig 域名`；`dig +tcp 域名` | 解析失败和 HTTP 服务是两层问题；普通查询 UDP 53，区域传送/截断转 TCP 53 |
| UDP 服务“没反应” | 是否有进程监听 UDP、防火墙、客户端是否发对地址 | `ss -lunp`；抓包看是否有 UDP 报文 | UDP 无连接，没有 TCP 那样的握手失败提示，容易表现为“发出去没回” |
| 看到 `HTTP/2` 或 `HTTP/3` 输出 | 应用层协议版本，不代表传输层一定是 TCP 或 UDP | `curl -v`；查 curl 是否支持 HTTP/3 | HTTP/2 通常跑在 TCP 上；HTTP/3 跑在 QUIC/UDP 上，**需按环境核实** |

<br>

<br>

## 常见误区
***
- **把 UDP 当成“不可靠所以不能用”**：QUIC/HTTP3 就在 UDP 上实现可靠传输。
- **以为 DNS 只用 UDP**：普通查询主要 UDP/53，区域传送用 TCP/53，截断等场景可能转 TCP。
- **把端口当进程号**：端口是传输层标识，进程号是操作系统内部标识。
- **以为 TCP 和 UDP 不能同端口**：端口空间按协议分开，同一端口号可以分别绑定 TCP 和 UDP。
- **以为知名端口不能改**：80、443、53 只是默认约定，服务可以配置到其他端口。
- **用 `ping` 通证明服务正常**：`ping` 通只说明网络层可达，不代表端口和应用层服务可用。
- **以为 TCP 一定比 UDP 慢**：要看场景；TCP 有握手和拥塞控制，但 UDP 上自己实现可靠机制也可能带来开销。

<br>

<br>

## 自测三问
***
1. TCP 和 UDP 的核心差别是什么？为什么 UDP 还能用于 HTTP/3？
2. 端口的作用是什么？它和进程号有什么区别？为什么 TCP 和 UDP 可以同端口？
3. 为什么 DNS 普通查询常用 UDP，而区域传送常用 TCP？

<details><summary>参考答案</summary>1. TCP 面向连接、可靠、按序字节流，有流量控制和拥塞控制；UDP 无连接、尽最大努力交付、开销小。HTTP/3 用 QUIC 在 UDP 上实现可靠、加密和多路复用，所以 UDP 只是承载，可靠性由 QUIC 补上。2. 端口是传输层用来标识应用进程的编号，IP 找主机、端口找进程；进程号是操作系统内部标识。TCP 和 UDP 的端口号空间按协议分开，所以同一端口号可以分别给 TCP 和 UDP 使用。3. 普通 DNS 查询报文短，UDP 不用握手、开销小，追求快；区域传送数据量大，需要可靠有序完整，所以用 TCP。普通查询响应过大或被截断时也可能转 TCP。</details>

<br>

<br>

## 总结
***
**传输层选择就是权衡：怕丢选 TCP，怕慢常选 UDP；IP 找主机、端口找进程、协议定服务质量，HTTP/DNS/SMTP/视频会议各自按需求选传输层，可靠性也可以在上层补。**

<br>

<br>

## 参考资料
***
- RFC 768：User Datagram Protocol
- RFC 9293：Transmission Control Protocol
- RFC 1035：Domain Names - Implementation and Specification
- RFC 9114：HTTP/3；RFC 9000：QUIC
- RFC 5321：Simple Mail Transfer Protocol；RFC 1939：POP3；RFC 3501：IMAP4rev1
- RFC 959：File Transfer Protocol
- 需按环境核实：`ss`、`dig`、`tcpdump`、`nc` 的可用性和权限要求；`dig +tcp` 的行为；HTTP/3 支持情况取决于 curl 版本和编译选项；UDP `connect` 在不同操作系统/编程接口下的具体语义；`SO_REUSEADDR` 在不同系统上的行为差异。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>