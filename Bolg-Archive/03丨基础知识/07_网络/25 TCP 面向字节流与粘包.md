# 25 TCP 面向字节流与粘包：字节流边界、三种分包方式
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
你写了个聊天程序。客户端连续调用两次 `send`，先发 `"Hi."`，再发 `"I am Xiaolin"`。

你可能会想：**我发了两次，服务端应该收两次吧？一次 `"Hi."`，一次 `"I am Xiaolin"`。**

但服务端一次 `recv`，可能直接收到 `"Hi.I am Xiaolin"`；也可能先收到 `"Hi.I am"`，再收到 `" Xiaolin"`。发送方明明分两次发，接收方却没法靠 TCP 自己区分“一条消息到哪结束”。

这就是所谓的“粘包/拆包”。**但这不是 TCP 的 bug，TCP 本来就不保证消息边界，它只保证字节按序到达。**

这一章咱们用一条问题链往前走：

1. TCP 面向字节流，UDP 面向报文，到底差在哪？
2. 为什么一次 `send()` 不一定对应一次 `recv()`？
3. 应用层怎么给字节流划边界？定长、分隔符、长度字段分别适合什么场景？
4. HTTP 头部和正文之间的空行属于哪种分包方式？

<br>

<br>

## 优先级说明
***
这篇是 TCP 应用协议设计的核心，面试常问，项目里写自定义协议、排查“消息粘在一起/收不全”也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- TCP 是连续字节流，没有天然消息边界；UDP 一个报文一条消息
- `send()` 只是把数据拷到内核发送缓冲区，什么时候发、怎么分组由 TCP 决定
- 一次 `send()` 不一定对应一次 `recv()`，一次 `recv()` 也不一定拿到一条完整消息
- 三种常见分包方式：定长、特殊分隔符、长度字段 + 包体
- HTTP 头部和正文之间的空行属于特殊字符边界；正文边界靠 `Content-Length` 或分块编码
- 长度字段分包要处理“半包”和“超大长度”
- 长度字段要用网络字节序

**了解即可（知道大概，不展开）：**
- HTTP 分块编码的具体格式
- Protobuf 等序列化框架的消息边界处理
- 分隔符转义的具体方案
- `recv` 返回 0 和负值的具体语义

**主线**：TCP 是字节流 → 没有消息边界 → 应用层用定长/分隔符/长度字段划边界 → 长度字段最常见，但要处理半包和超大长度。

<br>

<br>

## 知识点 1：TCP 字节流与 UDP 报文边界——发了两次，不代表对方收两次
***
### 问题

“面向字节流”到底什么意思？为什么 UDP 能靠一个报文区分一条消息，TCP 却不行？一次 `send()` 是不是就对应一次 `recv()`？

**你可能会想**：我调了两次 `send`，内核应该发两个包，对方就该收两次吧？

不是。**`send()` 只是把数据拷到内核发送缓冲区，什么时候发、怎么分组由 TCP 决定，应用层控制不了。** TCP 可能把两条消息合在一个报文里发，也可能把一条消息拆成多个报文发。

<br>

### 核心概念与方案

先把 TCP 换成你熟悉的场景：**TCP 像一根水管，你往里倒水，对方接到的是连续的水流，不知道你倒了几杯；UDP 像寄包裹，一个包裹就是一个边界。**

- **UDP**：发送时操作系统不拆分用户消息，一个 UDP 报文就是一个用户消息边界；接收方读一个报文，就是完整消息。
- **TCP**：把数据看成**连续字节流**：用户消息可能被拆成多个 TCP 报文，也可能多个消息合在一个报文里。`send()` 只是把数据拷到内核发送缓冲区，什么时候发、怎么分组由 TCP 决定，应用层控制不了。

所以接收方如果不知道消息边界，就会出现所谓的“粘包/拆包”解析问题。**“粘包”不是 TCP 的 bug**，TCP 本来就不提供消息边界；这是应用层协议要解决的事。也不能假设一次 `send()` 对应一次 `recv()`，更不能假设一次 `recv()` 就拿到一条完整消息。

<br>

### 最小实验：只看服务端一次 recv 收到的是不是两条消息合在一起

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 且支持 C++17。下面只在本机回环地址上运行，不对外网开放。

服务端保存为 `tcp_stream_server.cpp`：

````cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <iostream>

int main() {
    const int port = 18082;

    int serverSocket = socket(AF_INET, SOCK_STREAM, 0);
    if (serverSocket < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    int opt = 1;
    setsockopt(serverSocket, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(port);

    if (bind(serverSocket, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        std::cerr << "bind failed\n";
        close(serverSocket);
        return 1;
    }

    listen(serverSocket, 1);
    std::cout << "server listening on " << port << "\n";

    sockaddr_in clientAddr{};
    socklen_t clientLen = sizeof(clientAddr);
    int conn = accept(serverSocket, reinterpret_cast<sockaddr*>(&clientAddr), &clientLen);
    if (conn < 0) {
        std::cerr << "accept failed\n";
        close(serverSocket);
        return 1;
    }

    char buf[1024];
    ssize_t n = recv(conn, buf, sizeof(buf) - 1, 0);
    if (n > 0) {
        buf[n] = '\0';
        std::cout << "一次 recv 收到 " << n << " 字节: [" << buf << "]\n";
    }

    close(conn);
    close(serverSocket);
    return 0;
}
````

客户端保存为 `tcp_stream_client.cpp`：

````cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
#include <iostream>

int main() {
    const char* server_ip = "127.0.0.1";
    const int server_port = 18082;

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(server_port);
    inet_pton(AF_INET, server_ip, &addr.sin_addr);

    if (connect(sock, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        std::cerr << "connect failed\n";
        close(sock);
        return 1;
    }

    const char* msg1 = "Hi.";
    const char* msg2 = "I am Xiaolin";

    send(sock, msg1, std::strlen(msg1), 0);
    send(sock, msg2, std::strlen(msg2), 0);

    close(sock);
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 tcp_stream_server.cpp -o tcp_stream_server
g++ -std=c++17 -O2 tcp_stream_client.cpp -o tcp_stream_client

# 终端 1
./tcp_stream_server

# 终端 2
./tcp_stream_client
````

**这次只看一件事**：服务端一次 `recv` 收到的是不是 `"Hi.I am Xiaolin"`，还是只收到一部分。

可能看到：

````text
一次 recv 收到 16 字节: [Hi.I am Xiaolin]
````

也可能只收到一部分，比如：

````text
一次 recv 收到 3 字节: [Hi.]
````

**具体收到多少取决于发送时机、缓冲区和系统调度，需按环境核实。** 但关键是：**你不能假设一次 `recv` 就对应一次 `send`，也不能假设一次 `recv` 就拿到一条完整消息。**

如果服务端一次收到完整两条消息合并，说明 TCP 把两次 `send` 合并了；如果只收到一部分，说明 TCP 把消息拆开了。**两种都是正常的。**

<br>

### 易错点与边界

坑一：**“粘包”不是 TCP 的 bug**，是字节流没有消息边界；别把锅甩给 TCP。

坑二：**不能假设一次 `send()` 对应一次 `recv()`**，发送缓冲区和接收缓冲区都会影响实际收发次数。

坑三：**`send()` 成功不等于对方已收到**，只表示数据进了本机内核发送缓冲区。

坑四：**TCP 不保留消息边界，但保留字节顺序**；你收到的字节顺序是对的，只是不知道一条消息到哪结束。

<br>

### 一句话本质

**UDP 一个报文一条消息；TCP 是连续字节流，发了两次不代表对方收两次，消息边界要应用层自己划。**

<br>

### 自测

为什么 TCP 不能像 UDP 那样靠一个报文区分一条消息？如果接收方连续调用 `recv`，一定能一次拿到一条完整消息吗？

<details><summary>答案</summary>TCP 是连续字节流，`send` 只是把数据拷到内核发送缓冲区，TCP 可能合并多个消息、拆分一个消息，不保留消息边界。UDP 一个报文就是一个用户消息边界。接收方连续 `recv` 也不一定能一次拿到完整消息，可能收到半条、多条合并，或者一条消息分多次到达，必须靠应用层协议划边界。</details>

<br>

<br>

## 知识点 2：三种分包方式——定长、分隔符、长度字段
***
### 问题

应用层怎么划消息边界？固定长度、特殊字符、长度字段分别适合什么场景？HTTP 头部和正文之间的空行属于哪种方式？

**你可能会想**：加个特殊字符当分隔符不就行了？

可以，但**如果消息内容本身包含这个特殊字符，就会被提前截断**，需要转义。所以分隔符方式适合内容里不太可能出现该字符的场景，或者你能可靠转义。

<br>

### 核心概念与方案

先把分包换成你熟悉的场景：**给字节流画边界，有三种笔：定长尺子、分隔线、先写长度再写内容。**

常见三种分包方式：

- **固定长度**：每条消息固定 N 字节，简单但不灵活；消息长短不一就会浪费空间，太短又不够用。
- **特殊字符边界**：用特殊分隔符划分消息，如 HTTP 用 `\r\n\r\n` 分隔头部和正文；内容里出现特殊字符要转义，否则会提前截断。
- **自定义消息结构：包头 + 包体**。包头固定大小，并包含包体长度；接收方先读包头，再按长度读指定字节数。这是最通用、最常用的方式。

HTTP 头部和正文之间的空行属于**特殊字符边界**；正文边界则常用 `Content-Length` 或分块编码，属于“长度字段/分块”思路。

一句话：**定长、分隔符、长度字段，是应用层给字节流画边界的三种笔。**

<br>

### 最小实验：只看长度字段分包能不能把两条消息正确拆开

环境前提：同上。下面只做本地模拟，不联网。

保存为 `length_prefix_demo.cpp`：

````cpp
#include <arpa/inet.h>

#include <cstdint>
#include <cstring>
#include <iostream>
#include <string>
#include <vector>

std::vector<uint8_t> pack_message(const std::string& body) {
    uint32_t len = static_cast<uint32_t>(body.size());
    uint32_t net_len = htonl(len);

    std::vector<uint8_t> packet(sizeof(net_len) + body.size());
    std::memcpy(packet.data(), &net_len, sizeof(net_len));
    std::memcpy(packet.data() + sizeof(net_len), body.data(), body.size());
    return packet;
}

bool try_parse_one(const std::vector<uint8_t>& data, size_t& pos,
                   std::string& out_body, uint32_t max_len = 1024 * 1024) {
    if (data.size() - pos < sizeof(uint32_t)) return false;

    uint32_t net_len = 0;
    std::memcpy(&net_len, data.data() + pos, sizeof(net_len));
    uint32_t len = ntohl(net_len);

    if (len > max_len) {
        std::cerr << "message too large: " << len << "\n";
        return false;
    }

    if (data.size() - pos - sizeof(uint32_t) < len) return false;

    const char* body_ptr = reinterpret_cast<const char*>(
        data.data() + pos + sizeof(uint32_t));
    out_body.assign(body_ptr, len);
    pos += sizeof(uint32_t) + len;
    return true;
}

int main() {
    auto p1 = pack_message("hello");
    auto p2 = pack_message("TCP boundary");

    std::vector<uint8_t> stream;
    stream.insert(stream.end(), p1.begin(), p1.end());
    stream.insert(stream.end(), p2.begin(), p2.end());

    size_t pos = 0;
    std::string body;
    while (try_parse_one(stream, pos, body)) {
        std::cout << "parsed: [" << body << "]\n";
    }

    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 length_prefix_demo.cpp -o length_prefix_demo
./length_prefix_demo
````

**这次只看一件事**：程序能不能把拼在一起的两条消息正确拆成两条。

预期输出：

````text
parsed: [hello]
parsed: [TCP boundary]
````

**关键观察点**：即使两条消息拼成一个连续字节流，接收方靠“先读 4 字节长度，再按长度读包体”也能正确还原两条消息。

如果只输出第一条，说明长度字段或读位置写错了；如果输出乱码，先检查字节序和长度字段有没有用 `htonl/ntohl`。

<br>

### 易错点与边界

坑一：**特殊字符要做转义**，否则消息内容里的分隔符会被误当成边界。

坑二：**长度字段要防超大值攻击**，不能对方说 4GB 你就直接分配 4GB。

坑三：**HTTP 用头部边界 + `Content-Length`/分块编码解决 body 边界**，不是只靠一个空行就解决整个报文。

坑四：**长度字段本身也要考虑字节序**，跨平台传输要用网络字节序，别直接塞主机字节序。

坑五：**接收方要处理“半包”**：包头可能只收到一部分，包体也可能只收到一部分，不能假设一次 `recv` 就收全。

<br>

### 一句话本质

**定长、分隔符、长度字段，是应用层给字节流画边界的三种笔；长度字段最常见，但要处理半包、超大长度和字节序。**

<br>

### 自测

HTTP 头部和正文之间的空行属于哪种分包方式？如果消息内容里本身包含分隔符，应该怎么处理？

<details><summary>答案</summary>HTTP 头部和正文之间的空行属于特殊字符边界，用 `\r\n\r\n` 分隔头部和正文。如果消息内容本身包含分隔符，需要转义，或者改用长度字段分包，否则内容里的分隔符会被误认为边界，导致消息被提前截断。</details>

<br>

<br>

## 完整可复现示例：C++ 实现长度字段分包并模拟半包
***
你需要 Linux/macOS 或 WSL，已安装 `g++`。下面把长度字段分包的打包和解包串起来，并模拟“一次只收到部分数据”的情况。

````bash
g++ -std=c++17 -O2 length_prefix_demo.cpp -o length_prefix_demo
./length_prefix_demo
````

预期输出：

````text
parsed: [hello]
parsed: [TCP boundary]
````

**关键观察点**：两条消息即使拼成一个连续字节流，也能被正确拆开。

验证结果：发送方把两条消息拼成一个连续字节流，接收方靠“先读 4 字节长度，再按长度读包体”正确还原两条消息；即使多条消息合在一起到达，也不会混淆边界。

注意：这个示例把完整流一次性放进内存演示解析逻辑；真实网络程序要循环 `recv`，把数据追加到缓冲区，再反复尝试解析。长度字段要做最大值校验，防止恶意超大长度；包头和包体都可能“半包”，不能假设一次 `recv` 就收全。**具体行为需按环境核实。**

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| 服务端一次收到多条消息合在一起 | 应用层有没有划边界 | 抓包看字节流；看应用层解析逻辑 | TCP 是字节流，不保留消息边界；要靠定长/分隔符/长度字段 |
| 服务端只收到半条消息 | 是否处理半包 | 看 `recv` 返回值；看缓冲区逻辑 | TCP 可能拆包，包头和包体都可能只收到一部分，要循环读 |
| 分隔符分包内容被截断 | 内容里是否含分隔符、是否转义 | 看原始消息；查转义逻辑 | 内容里的分隔符会被误认为边界，需要转义或改用长度字段 |
| 长度字段解析出超大值 | 是否做最大值校验 | 看长度字段；查校验逻辑 | 恶意超大长度可能导致内存耗尽，要设上限 |
| 跨平台长度字段乱码 | 是否用网络字节序 | 看打包/解包代码 | 长度字段要 `htonl/ntohl`，不能直接塞主机字节序 |
| HTTP body 边界解析错 | 是 `Content-Length` 还是分块编码 | 抓包看头部；查解析逻辑 | HTTP 头部用空行分隔，body 用 `Content-Length` 或分块编码，不是只靠空行 |
| `recv` 返回 0 | 对端是否关闭连接 | `ss -tan` 看状态；抓包看 FIN | `recv` 返回 0 通常表示对端正常关闭，不是“没数据” |

<br>

<br>

## 常见误区
***
- **把“粘包”当成 TCP 的 bug**：TCP 是字节流，本来就不提供消息边界。
- **以为一次 `send()` 对应一次 `recv()`**：发送缓冲区和接收缓冲区都会影响实际收发次数。
- **以为 `send()` 成功就是对方收到**：只表示数据进了本机内核发送缓冲区。
- **分隔符分包不转义**：内容里的分隔符会被误认为边界，导致消息截断。
- **长度字段不做最大值校验**：恶意超大长度可能导致内存耗尽。
- **长度字段不用网络字节序**：跨平台传输会乱码。
- **假设一次 `recv` 收全**：包头和包体都可能半包，要循环读。
- **以为 HTTP 只靠空行分 body**：头部用空行分隔，body 还要靠 `Content-Length` 或分块编码。

<br>

<br>

## 自测三问
***
1. TCP 面向字节流和 UDP 面向报文在发送端有什么区别？为什么一次 `send()` 不一定对应一次 `recv()`？
2. 三种常见分包方式是什么？HTTP 头部和正文之间的空行属于哪一种？
3. 为什么长度字段分包要处理“半包”和“超大长度”？如果接收方只收到 2 字节长度字段，应该怎么办？

<details><summary>参考答案</summary>1. UDP 发送时操作系统不拆分用户消息，一个报文一个边界；TCP 把数据看成连续字节流，`send` 只是把数据拷到内核发送缓冲区，什么时候发、怎么分组由 TCP 决定，所以一次 `send` 不一定对应一次 `recv`，一次 `recv` 也不一定拿到完整消息。2. 三种方式是定长、特殊分隔符、长度字段 + 包体。HTTP 头部和正文之间的空行属于特殊字符边界；body 边界靠 `Content-Length` 或分块编码。3. 半包是因为 TCP 可能只送达部分数据，包头和包体都可能不完整，要循环 `recv` 并缓冲；超大长度要校验，防止恶意长度导致内存耗尽。如果只收到 2 字节长度字段，应该继续 `recv`，把数据追加到缓冲区，等够 4 字节再解析长度。</details>

<br>

<br>

## 总结
***
**TCP 是连续字节流，没有天然消息边界；应用层必须用定长、特殊分隔符或“长度字段+包体”自己划边界，否则就会出现粘包/拆包解析问题；长度字段最常见，但要处理半包、超大长度和字节序。**

<br>

<br>

## 参考资料
***
- RFC 9293：Transmission Control Protocol
- RFC 9112：HTTP/1.1（头部和 body 边界、`Content-Length`、分块编码）
- `man 2 send`、`man 2 recv`
- 需按环境核实：一次 `send` 对应几次 `recv` 取决于发送缓冲区、接收缓冲区、系统调度和网络条件；`recv` 返回 0 通常表示对端关闭，但具体语义需按实现核实；长度字段的最大值校验阈值因应用场景而异。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>