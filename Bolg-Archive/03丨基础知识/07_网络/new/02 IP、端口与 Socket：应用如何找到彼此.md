# 02 IP、端口与 Socket：应用如何找到彼此
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 场景问题

***

你在本地写了一个服务，用 `python server.py` 跑起来，自己 `curl 127.0.0.1:12345` 一切正常。部署到测试环境后，同事说“连不上”，他 `telnet 10.0.0.5 12345` 卡了半天，最后报 `Connection timed out`。你第一反应是“端口没开放”或者“防火墙挡了”。这个方向可能对，但不够——**“连不上”至少有三种截然不同的底层原因**：端口根本没在监听，通常表现为 `Connection refused`；监听了但绑错了 IP，表现为本机通、远程不通；或者 SYN 包被中间设备静默丢弃，表现为 `Connection timed out`。如果分不清这三者，排查会一直在“重启服务”和“关防火墙”之间原地打转。

<br>

<br>

## 核心概念

***

### IP 与端口：一台机器和一个服务

<br>

一台主机在网络上需要一个 IP 地址，别人才能把数据包送到它面前。但一台机器上跑着许多服务——Web 服务器、SSH、数据库、你刚写的测试服务——数据包到了主机之后，还需要一个“门牌号”来区分到底交给谁。**端口（Port）就是这个门牌号**，它是一个 16 位整数，范围 0–65535。

IANA 把端口划分为三段：**系统端口（System Ports，0–1023）**、**用户端口（User Ports，1024–49151）** 和**动态/私有端口（Dynamic/Private Ports，49152–65535）**。你的测试服务用 12345，落在用户端口区间，普通用户权限即可绑定；如果非要用 80，通常需要 root 或额外能力，因为系统端口一般保留给需要特权的进程。**IP 定位“哪台机器”，端口定位“机器上的哪个服务”**，两者合在一起，才能把数据交给正确的进程。

<br>

### Socket：应用和内核之间的“接口合同”

<br>

应用本身不直接处理 TCP/IP 数据包。你调用 `socket()`，实际上是在向操作系统内核申请一个**通信端点（endpoint）**。这个端点在内核里有状态、有缓冲区、有队列。Socket API 就是你（应用）和内核之间的合同：你告诉内核“我要在哪个 IP、哪个端口上等”或者“我要连到哪个 IP、哪个端口”，内核负责后面的握手、收发和排队。

一个 TCP 连接在内核视角里由**四元组**唯一标识：`(源 IP, 源端口, 目的 IP, 目的端口)`。这就是为什么同一个客户端可以同时打开多个到同一服务器的连接——源端口不同，四元组就不同。客户端调用 `connect()` 时一般不需要自己指定源端口，内核会从临时端口范围（Linux 上通常由 `ip_local_port_range` 控制）里选一个未被占用的端口；只要源端口不同，即使目的 IP 和目的端口相同，四元组仍然唯一。

<br>

### 服务端的两个内核队列：listen 之后、accept 之前

<br>

这是初学者最容易忽略、但面试和排障都绕不开的细节。服务端调用 `listen(fd, backlog)` 之后，内核会为这个监听 Socket 维护**两个队列**：

- **半连接队列（SYN queue）**：收到 SYN、回了 SYN-ACK，但三次握手还没完成的连接请求。它存的不是 SYN 包本身，而是**收到了 SYN、尚未收到最终 ACK 的连接请求状态**；长度与内核参数 `tcp_max_syn_backlog` 强相关，但并非直接等于该值，内核还会施加下限约束（如 `max(64, tcp_max_syn_backlog)`）。排障时把它视为“半连接容量的主要旋钮”即可，不要把它当作精确的队列长度读数。
- **全连接队列（accept queue）**：三次握手已经完成、但应用还没调用 `accept()` 取走的连接。长度受 `listen()` 传入的 `backlog` 和内核 `somaxconn` 共同约束，具体默认值和上限以当前主流实现为准，不同版本需查证。

**`accept()` 不从网络“拉取”连接，而是从全连接队列里取走一个已经就绪的连接**。如果你写了 `listen` 但从不 `accept`，握手照样能在内核里完成，连接会堆在全连接队列里；队列满了，新来的连接就可能被丢弃或拒绝——表现出来就是“客户端连不上”，但你的进程看起来“明明在跑”。这里需要特别说明 Linux 的默认行为：`tcp_abort_on_overflow=0` 时，全连接队列满后内核**静默丢弃**后续 ACK，客户端表现为 `connect()` 超时，而不是立即收到 RST；只有把 `tcp_abort_on_overflow` 设为 1，内核才会向客户端发 RST，客户端才可能看到 `Connection reset by peer`。因此“全连接队列满”和“防火墙丢 SYN”在客户端侧都可能是 `timed out`，不能仅凭超时排除队列问题。这里的三次握手只作为时序背景：客户端发 SYN，服务端回 SYN-ACK，客户端再回 ACK；内核完成握手后把连接放进全连接队列。**TCP 状态机的细节留到第 9 篇，本篇不展开。**

<br>

### 绑 127.0.0.1 还是 0.0.0.0：一个绑定决定谁能够到你

<br>

`bind()` 时指定的 IP 决定了监听的范围。`127.0.0.1` 是**回环地址（loopback）**，绑定它意味着只有本机内部能访问，流量走的是本机回环接口，不经过对外的物理网卡；在普通主机上，这等价于“只有本机内部能访问”。`0.0.0.0` 是一个特殊地址，表示“本机所有 IPv4 接口”，绑定它，同一局域网或公网能路由到这台机器的客户端都可以连进来。**很多“本地 curl 通、远程 telnet 不通”的事故，根源就是服务端 `bind` 了 `127.0.0.1`，而不是 `0.0.0.0`。**

这里有一个常见误解，也是面试追问陷阱：**`ss` 里看到 `LISTEN`，不代表远程一定能连。** 正确回答思路是先看 `Local Address`：如果是 `127.0.0.1:12345`，那只监听回环；如果是 `0.0.0.0:12345` 或 `*:12345`，才表示监听所有接口。然后再谈防火墙、安全组和路由。另一个追问是“服务端不调用 `accept()`，客户端能 `connect()` 成功吗？”——**通常能完成内核层面的三次握手并进入全连接队列**，但应用不取走，队列满后新连接会受影响；默认情况下是静默丢弃并让客户端超时，具体行为受 `tcp_abort_on_overflow` 等参数影响，需结合当前内核参数查证。

<br>

<br>

## 动手实验

***

> 以下命令在 Linux 上验证；`ss` 属于 iproute2 工具集。Windows/macOS 读者可以把 `ss -tlnp` 暂时替换为 `netstat -an | grep 12345`（队列字段的语义会不同，但监听地址和状态仍可参考）。如果你的系统只有 `python3`，把下文中的 `python` 替换为 `python3`。

### 实验一：跑一个极简 TCP echo 服务，用 ss 观察监听与连接

<br>

准备两个 Python 文件。服务端：

```python
# echo_server.py
import socket

HOST = "0.0.0.0"   # 先绑所有接口；后面再改成 127.0.0.1 对比
PORT = 12345

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server:
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((HOST, PORT))
    server.listen(1)          # 先保持 backlog=1；后续“启示二”会解释它为什么在高并发下可能不够
    print(f"listening on {HOST}:{PORT}")
    while True:
        conn, addr = server.accept()
        with conn:
            print(f"accepted from {addr}")
            data = conn.recv(1024)
            if data:
                conn.sendall(b"echo: " + data)
```

客户端：

```python
# echo_client.py
import socket

with socket.create_connection(("127.0.0.1", 12345), timeout=5) as s:
    s.sendall(b"hello")
    print(s.recv(1024).decode())
```

先启动服务端：

```bash
python3 echo_server.py
```

再在另一个终端运行客户端：

```bash
python3 echo_client.py
```

> `echo: hello`

然后在服务端运行期间，开第三个终端观察：

```bash
ss -tnp | grep 12345
```

> `LISTEN  0  1  0.0.0.0:12345  0.0.0.0:*  users:(("python",pid=...,fd=3))`
>
> `ESTAB   0  0  127.0.0.1:12345  127.0.0.1:54321  users:(("python",pid=...,fd=4))`

**看到什么说明什么**：第一行 `LISTEN` 状态、本地地址 `0.0.0.0:12345`，说明内核正在所有 IPv4 接口的 12345 端口上等待 SYN。在 Linux 的 `ss` 中，`LISTEN` 行的 `Recv-Q` 是**当前全连接队列里的连接数**，`Send-Q` 是**这个队列的容量上限**（由 `backlog` 与 `somaxconn` 取 min 决定）。第二行 `ESTAB` 说明这个连接的三次握手已经完成，已经是一个内核层面的已建立连接。它可能正在全连接队列里等 `accept()`，也可能已经被 `accept()` 取走；从 `ss` 这一行本身通常无法区分这两种情况。如果你在 `accept()` 之前就看到了 `ESTAB`，不是异常——握手是内核完成的，不需要应用参与。`ESTAB` 行的 `Recv-Q/Send-Q` 则是收发缓冲区中未被应用取走/未被确认的字节数。两套语义不要混用。

**可选：观察全连接队列堆积**。先保持服务端运行，然后开几个终端，各运行下面这条命令，并让进程挂住不发送数据：

```bash
python3 -c 'import socket; s=socket.create_connection(("127.0.0.1",12345)); input("按回车断开...")'
```

由于服务端 `accept()` 第一个连接后会阻塞在 `recv()`，后面的连接会完成握手但留在全连接队列里。此时在另一个终端运行：

```bash
ss -lnt | grep 12345
```

观察 `LISTEN` 行的 `Recv-Q` 是否接近 `Send-Q`。如果继续发起连接，客户端可能表现为 `timed out`，这就是“队列满 + 默认静默丢弃”的典型现象。还可以用下面这条命令查看内核累计的溢出计数：

```bash
netstat -s | grep -i overflowed
```

**常见失败 1**：`ss -tlnp | grep 12345` 什么都没有。服务端可能没启动，或者绑定失败退出了。端口被占用时常见报错是 `OSError: [Errno 98] Address already in use`。先用 `ss -tlnp | grep 12345` 找占用进程，再决定换端口还是停进程。

**常见失败 2**：客户端报 `Connection refused`。这通常说明目标主机可达，但目标端口上没有进程在 `LISTEN`——内核收到 SYN 后直接回了 RST。少数情况下也可能是中间设备或监听方主动发送了 RST。先确认服务端真的在 `LISTEN`，再看 `Local Address` 是否覆盖你访问的地址。

**常见失败 3**：客户端一直卡到 `timed out`。这更像 SYN 被静默丢弃、路由不可达、中间防火墙拦截，或者全连接队列溢出后的默认静默丢弃，而不是简单的“端口没开”。这时要先看监听地址，再查网络路径和中间设备，同时用 `ss -lnt` 看全连接队列是否接近上限。

<br>

### 实验二：把绑定改成 127.0.0.1，观察监听范围变化

<br>

把服务端第一行改成：

```python
HOST = "127.0.0.1"
```

重启服务端，再次观察：

```bash
ss -tlnp | grep 12345
```

> `LISTEN  0  1  127.0.0.1:12345  0.0.0.0:*  users:(("python",pid=...,fd=3))`

**看到什么说明什么**：`Local Address` 从 `0.0.0.0:12345` 变成了 `127.0.0.1:12345`，说明服务只监听回环接口。此时本机 `python3 echo_client.py` 仍然能通，但其他机器通过本机非回环 IP 访问时通常连不上。**这正是“本地通、远程不通”的典型原因之一。** 修复方式不是盲目关防火墙，而是确认业务是否真的需要对外提供，再决定绑 `0.0.0.0` 还是指定具体网卡 IP，并配合安全组/防火墙最小化放行。

<br>

<br>

## 开发启示

***

### 启示一：先看监听地址，再谈防火墙

<br>

**现象**：本地 `curl 127.0.0.1:12345` 通，同事 `telnet 10.0.0.5 12345` 超时。**底层原因**：服务端可能 `bind` 在 `127.0.0.1`，只监听回环；也可能监听正确但中间防火墙静默丢 SYN；还可能是全连接队列满后默认静默丢弃。**可操作动作**：先在服务端跑 `ss -tlnp | grep 12345`，看 `Local Address` 是 `127.0.0.1` 还是 `0.0.0.0`；再看客户端报错是 `refused` 还是 `timed out`。**面试答法**：先看 `ss` 的 `Local Address` 和客户端报错：`refused` 优先查监听，`timed out` 且监听覆盖正确时才把中间设备和全连接队列溢出纳入怀疑范围。不要在没有这两步之前就把“防火墙”当成第一答案。

<br>

### 启示二：服务在跑不等于连接能被取走

<br>

**现象**：进程活着，但高峰期偶发连接超时或拒绝。**底层原因**：`listen` 之后连接先进入内核队列，应用 `accept()` 太慢或 `backlog` 太小，全连接队列可能堆满。Linux 默认 `tcp_abort_on_overflow=0`，队列满时内核静默丢弃后续 ACK，客户端通常表现为超时；设为 1 才会发 RST。**可操作动作**：用 `ss -lnt` 观察监听 Socket 的 `Recv-Q/Send-Q`，检查代码里 `accept()` 循环是否有阻塞操作；合理设置 `backlog`，并了解 `somaxconn` 的上限，具体默认值需查证；必要时用 `netstat -s | grep -i overflowed` 查看溢出计数。**面试答法**：`accept()` 是从全连接队列取连接，不是从网络拉连接；不 `accept` 也可能完成握手，但队列满后新连接会受影响，默认表现为客户端超时。

<br>

### 启示三：端口冲突不要只会“重启”

<br>

**现象**：启动报 `Address already in use`。**底层原因**：该端口已有监听 Socket，或者地址/端口组合被占用；有时还和连接状态、复用选项有关。**可操作动作**：用 `ss -tlnp | grep 端口` 找 PID 和进程名，确认是旧进程、同机其他服务还是配置重复；开发环境可换端口，生产环境要按发布流程处理。**面试答法**：先定位占用者。如果 `EADDRINUSE` 是因为旧连接处于 `TIME_WAIT`、需要快速重启，`SO_REUSEADDR` 是合适的；如果是有另一个进程正在 `LISTEN`，`SO_REUSEADDR` 帮不上忙，不要把它当成“端口复用”的万能开关。真正用于多进程共享监听端口的 `SO_REUSEPORT` 语义与平台相关，本篇不展开。

<br>

<br>

## 小结与下一篇预告

***

开头那个“本地通、远程不通”的问题，现在可以闭环回答了：**先看服务端 `ss -tlnp` 的 `Local Address`，确认是 `127.0.0.1` 还是 `0.0.0.0`；再看客户端是 `refused` 还是 `timed out`；最后才查防火墙、安全组和路由。** 如果绑定错了，改绑定；如果没监听，查进程；如果是超时且监听正确，除了中间设备丢 SYN，也要想到**全连接队列溢出时的静默丢弃**——用 `ss -lnt` 看 `Recv-Q` 是否接近 `Send-Q`，或查 `netstat -s | grep -i overflowed`。这样“timed out”的闭环才和“启示二”对得上。

下一篇讲 **第 3 篇：HTTP 基础：请求、响应、状态码与 Header**，我们会用 `curl` 观察一次 HTTP 请求长什么样。

自测问题：

1. TCP 连接的四元组是哪四个字段？为什么源端口不同就能区分不同连接？
<details>
<summary>[点击展开]</summary>
四元组是源 IP、源端口、目的 IP、目的端口。客户端 connect() 时内核会分配一个临时源端口，所以同一客户端连同一服务端时，源端口不同，四元组就不同，内核就能区分成不同连接。
</details>

<br>

2. `127.0.0.1:12345` 和 `0.0.0.0:12345` 在监听范围上有什么区别？
<details>
<summary>[点击展开]</summary>
127.0.0.1 只监听回环，只有本机能通过回环地址访问；0.0.0.0 监听本机所有 IPv4 接口，远程才有连进来的可能，但还要看防火墙和路由。排查时看 ss -tlnp 的 Local Address。
</details>

<br>

3. 客户端报 `Connection refused` 和 `Connection timed out`，排查方向有什么不同？
<details>
<summary>[点击展开]</summary>
refused 通常是目标端口没有进程在 LISTEN，内核回了 RST，优先查服务有没有监听、绑在哪。timed out 通常是 SYN 被静默丢弃，优先查绑错地址、防火墙/安全组、路由，以及全连接队列是否溢出。
</details>

<br>

4. 全连接队列满时，Linux 默认会让客户端看到 `refused` 还是 `timed out`？`tcp_abort_on_overflow` 会怎样影响这个行为？
<details>
<summary>[点击展开]</summary>
默认是 timed out。Linux 默认 tcp_abort_on_overflow=0，队列满时静默丢弃，客户端只能超时。设为 1 后内核会回 RST，客户端通常看到 Connection reset by peer，不是 refused。
</details>


<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>