# 18 TCP 基础与连接管理：报文、三次握手、四次挥手、TIME_WAIT
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
TCP 可靠、面向连接、基于字节流。**为什么建立连接要三次握手？断开为什么四次挥手？主动关闭方为什么最后要 TIME_WAIT 等 2MSL？**  
先别急着背“SYN、ACK、FIN”这几个标志位，先看一个具体场景：你写了个客户端，调用 `connect` 连服务器，然后 `send` 了一段数据，最后 `close`。抓包一看，连接建立时是三步，断开时却是四步，最后还多了一个“等 2MSL”的状态。为什么建连三次就够了，断连却要四次？TIME_WAIT 到底在等什么？这一章咱们把 TCP 的连接生命周期从报文格式到握手挥手完整走一遍。

<br>

<br>

## 知识点 1：TCP 报文与连接唯一性
***
### 问题
TCP 靠什么标识一条连接？序号和确认号到底是什么意思？为什么说 TCP 是“字节流”而不是“报文流”？

<br>

### 方案
TCP 连接由**四元组**唯一确定：**源 IP、源端口、目的 IP、目的端口**。同一个服务器 80 端口可以同时服务多个客户端，就是因为每个连接的源 IP 或源端口不同，四元组不同。  
TCP 是**字节流协议**：**序号**标识本报文段第一个数据字节的编号；**确认号**表示期望收到的下一个字节序号，采用**累积确认**——确认号之前的字节都收到了。  
控制位：**SYN** 建连、**ACK** 确认、**FIN** 结束、**RST** 重置。TCP 头还有窗口字段用于流量控制，首部通常 20 字节，选项可变长。

<br>

### 最小示例
```text
发 Seq=1, Len=500 → 接收方回 Ack=501
表示：1~500 这些字节都收到了，下一个期望 501。
```

结论：序号按字节编号，不是按包编号；确认号是“我下一个想要谁”，不是“我收到了谁”。

<br>

### 一句话本质
**TCP 连接是四元组契约，序号按字节编号，确认号表示“我下一个想要谁”。**

<br>

### 易错点
坑一：**序号不是按包编号，是按字节**；一个报文段可以携带多个字节，下一个报文段的序号要加上本段数据长度。  
坑二：**确认号是期望的下一个字节**，不是“最后收到的字节”；Ack=501 表示 501 之前都收到了。  
坑三：**累积确认意味着 Ack=2000 表示 2000 之前都收到**，中间缺了会重复确认之前的字节。  
坑四：**四元组相同不代表是同一条连接**，还要看时间；旧连接的延迟报文可能和新连接撞四元组，这正是 TIME_WAIT 要防的。

<br>

### 自测
收到 Ack=2000 意味着什么？如果发送方发了 Seq=1500、Len=600，接收方会回什么确认号？

<br>

<br>

## 知识点 2：三次握手与四次挥手
***
### 问题
为什么建连三次、断连四次？三次握手能不能改成两次？四次挥手为什么不能合并成两次？TIME_WAIT 到底在等什么？

<br>

### 方案
**三次握手**：  
1. 客户端发 SYN（携带 `client_isn`）；  
2. 服务端回 SYN+ACK（携带 `server_isn`，`Ack=client_isn+1`）；  
3. 客户端回 ACK（`Ack=server_isn+1`）。  
目的有三个：**同步双方初始序列号、确认双方收发能力、防止历史连接**。第三次握手可以携带数据。  
**四次挥手**：  
1. 主动方发 FIN；  
2. 被动方回 ACK（此时可能还有数据要发）；  
3. 被动方发 FIN；  
4. 主动方回 ACK。  
因为 TCP 是全双工，两个方向要分别关闭；被动方收到 FIN 后可能还有数据没发完，所以 ACK 和 FIN 通常不能合并。  
**TIME_WAIT**：主动关闭方最后进入 TIME_WAIT，等待 **2MSL**。目的有两个：**防止旧连接延迟报文被新同四元组连接误收；保证最后的 ACK 能让被动方收到**，如果 ACK 丢了，被动方重发 FIN，主动方还能再回一次 ACK。  
补充：如果被动方没数据要发且开启延迟确认，四次挥手可能合并成三次；但标准流程仍是四次。

<br>

### 最小示例
```text
握手：SYN → SYN+ACK → ACK
挥手：FIN → ACK → FIN → ACK
主动关闭方最后 TIME_WAIT 2MSL。
```

结论：握手三次是为了双方都确认“我能发、你能收、你能发、我能收”；挥手四次是因为两个方向要分别关。

<br>

### 一句话本质
**三次握手是“你听得到我吗、我听得到你、好开聊”；四次挥手是“我发完了、知道了、我也发完了、好拜拜”。**

<br>

### 易错点
坑一：**TIME_WAIT 只出现在主动关闭方**，被动关闭方通常进入 CLOSE_WAIT，等应用调用 `close` 后才发 FIN。  
坑二：**2MSL 不是随便等的**，MSL 是报文最大生存时间，2MSL 保证旧报文在网络中消失，且最后 ACK 有重发机会。  
坑三：**如果被动方没数据要发且开启延迟确认，四次挥手可能合并成三次**，但不要因此认为标准挥手是三次。  
坑四：**第三次握手可以携带数据**，但前两次通常不行，因为连接还没完全建立。  
坑五：**大量 TIME_WAIT 不一定是 bug**，主动关闭方多、短连接多时就会出现；要结合业务判断，不能无脑调小。

<br>

### 自测
为什么主动关闭方需要 TIME_WAIT？如果不要 TIME_WAIT，可能出现什么问题？

<br>

<br>

## 完整可复现示例：用 C++ 抓一次握手挥手
***
下面这个示例用 C++ 写一个最小客户端，连接本地 8080 端口，发一条消息后关闭；同时用 `tcpdump` 抓包观察 SYN、SYN+ACK、ACK、FIN、ACK。  
先准备一个简单的服务端，可以直接用 `nc -l 8080`，也可以自己写一个最小 TCP 服务端。这里用 `nc` 简化。

C++ 客户端：

```cpp
// tcp_connect_demo.cpp
// 编译：g++ -std=c++17 -O2 tcp_connect_demo.cpp -o tcp_connect_demo
// 运行：./tcp_connect_demo
// 环境：Linux/POSIX

#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
#include <iostream>

int main() {
    const char* server_ip = "127.0.0.1";
    const int server_port = 8080;

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(server_port);

    if (inet_pton(AF_INET, server_ip, &addr.sin_addr) <= 0) {
        std::cerr << "invalid address\n";
        close(sock);
        return 1;
    }

    // connect 触发三次握手
    if (connect(sock, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        std::cerr << "connect failed\n";
        close(sock);
        return 1;
    }

    const char* msg = "Hello";
    send(sock, msg, std::strlen(msg), 0);

    // close 触发四次挥手（主动关闭方）
    close(sock);
    std::cout << "client done\n";
    return 0;
}
```

运行步骤：

```bash
# 终端 1：启动一个简单服务端
nc -l 8080

# 终端 2：抓包，观察 8080 端口
sudo tcpdump -i lo -n tcp port 8080 -c 20

# 终端 3：编译并运行客户端
g++ -std=c++17 -O2 tcp_connect_demo.cpp -o tcp_connect_demo
./tcp_connect_demo
```

输出/现象/结论可能类似：

```text
# tcpdump 输出（简化）
Flags [S]    客户端 → 服务端   ; 第一次握手 SYN
Flags [S.]   服务端 → 客户端   ; 第二次握手 SYN+ACK
Flags [.]    客户端 → 服务端   ; 第三次握手 ACK
Flags [P.]   客户端 → 服务端   ; 发送 "Hello"
Flags [F.]   客户端 → 服务端   ; 主动关闭 FIN
Flags [.]    服务端 → 客户端   ; ACK
Flags [F.]   服务端 → 客户端   ; 服务端 FIN
Flags [.]    客户端 → 服务端   ; 最后 ACK
```

验证结果：抓包能看到 SYN、SYN+ACK、ACK 三次握手，以及 FIN、ACK、FIN、ACK 四次挥手；客户端作为主动关闭方，最后会进入 TIME_WAIT。  
注意：`nc -l 8080` 的可用性和参数在不同系统上可能不同；抓包需要权限，没有权限时可以只用 `ss -tan` 观察连接状态，但看不到逐包标志位。`tcpdump` 输出里的 `[S.]` 表示 SYN+ACK，`[F.]` 表示 FIN+ACK，`[.]` 表示 ACK，具体格式随版本略有差异。

<br>

<br>

## 自测三问
***
1. TCP 连接四元组是什么？为什么同一个服务器端口能同时服务多个客户端连接？  
2. 三次握手的最主要原因是什么？为什么不能只握两次？  
3. TIME_WAIT 的两个作用是什么？为什么它只出现在主动关闭方？

<br>

<br>

## 一句话总结全章
***
**TCP 用四元组标识连接，用序号/确认号保证字节流可靠，用三次握手同步状态并防历史连接，用四次挥手分别关闭两个方向，用 TIME_WAIT 防止旧报文捣乱并保证最后 ACK 到达。**

<br>
<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>