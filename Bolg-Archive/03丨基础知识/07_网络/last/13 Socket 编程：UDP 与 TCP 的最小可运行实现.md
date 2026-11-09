# 13 Socket 编程：UDP 与 TCP 的最小可运行实现
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
你想写一个网络程序：客户端发一段小写字母，服务端收到后回大写。

你可能会想：这还不简单，找个 `send` 和 `recv` 不就行了？

但一动手就卡住：**UDP 客户端不用先连接就能发，TCP 服务端却要先 `listen` 再 `accept`。为什么一个不用连、一个必须连？`accept` 返回的又为什么是一个新套接字？**

这一章咱们用一条问题链往前走：

1. Socket 到底是什么，为什么说它是 API 不是协议？
2. UDP 怎么写，为什么客户端不用 `connect`，服务端回信地址从哪来？
3. TCP 怎么写，为什么服务端要 `listen` 和 `accept`，`accept` 返回的新套接字是干什么的？
4. 两者在代码结构上到底差在哪？

<br>

<br>

## 优先级说明
***
这篇是 Socket 编程的入门，面试常问，项目里写网络程序、排查连接问题也会用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- Socket 是操作系统提供的网络编程接口，不是协议
- `AF_INET` 是地址族，`SOCK_DGRAM`/`SOCK_STREAM` 是套接字类型
- UDP 客户端不用 `connect`，直接 `sendto`；服务端从 `recvfrom` 拿客户端地址回信
- TCP 服务端 `socket → bind → listen → accept`，`accept` 返回新套接字专门通信
- 原 `serverSocket` 继续监听，`accept` 返回的才是真正收发数据的连接
- `send` 不保证一次发完、`recv` 不保证一次收全，短消息示例是简化

**了解即可（知道大概，不展开）：**
- `listen` 参数和半连接/全连接队列的具体关系
- `SO_REUSEADDR` 在不同系统上的行为差异
- 多客户端并发、IO 多路复用、超时和信号处理
- Windows Winsock 与 POSIX socket 的差异

**主线**：Socket 是应用层接入传输层的大门 → UDP 是“贴地址就扔”，无连接但快 → TCP 是“先接通再说话”，有连接但可靠 → 两者代码结构差在 `connect`/`listen`/`accept`。

<br>

<br>

## 知识点 1：Socket 是应用层和传输层的大门，不是协议
***
### 问题

应用进程怎么把数据交给传输层？为什么说 Socket 不是协议，而是 API？

**你可能会想**：Socket 是不是一种网络协议？和 TCP/UDP 是一回事吗？

不是。**Socket 是操作系统提供的一组网络编程接口，不是协议。** 你创建套接字时选 `SOCK_DGRAM` 还是 `SOCK_STREAM`，才决定走 UDP 还是 TCP。

<br>

### 核心概念与方案

先把 Socket 换成你熟悉的场景：**Socket 像一扇门，门后面是传输层。你选哪扇门，决定你走 UDP 还是 TCP。**

创建套接字时指定两件事：

- **地址族**：`AF_INET` 表示 IPv4。
- **套接字类型**：`SOCK_DGRAM` 是 UDP，`SOCK_STREAM` 是 TCP。

应用层只能选套接字类型，不能改底层协议机制。选了 UDP，就要自己处理丢包和消息边界；选了 TCP，三次握手、重传、按序交付由内核完成。

C++ 里这些接口来自 POSIX socket API，需要包含 `<sys/socket.h>`、`<netinet/in.h>`、`<arpa/inet.h>`、`<unistd.h>` 等头文件；Windows 下要改用 Winsock，这里以 Linux/POSIX 为例。

<br>

### 最小实验：只看两个套接字能不能创建成功

环境前提：Linux/macOS，或 Windows WSL；已安装 `g++` 且支持 C++17。下面只创建套接字再关闭，不联网。

保存为 `socket_demo.cpp`：

````cpp
#include <sys/socket.h>
#include <unistd.h>
#include <iostream>

int main() {
    int udp_sock = socket(AF_INET, SOCK_DGRAM, 0);
    int tcp_sock = socket(AF_INET, SOCK_STREAM, 0);

    if (udp_sock < 0 || tcp_sock < 0) {
        std::cerr << "socket create failed\n";
        return 1;
    }

    std::cout << "udp_sock=" << udp_sock << ", tcp_sock=" << tcp_sock << "\n";

    close(udp_sock);
    close(tcp_sock);
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 socket_demo.cpp -o socket_demo
./socket_demo
````

**这次只看一件事**：输出里有没有两个非负的套接字编号。

预期输出类似：

````text
udp_sock=3, tcp_sock=4
````

有编号，说明两个套接字都创建成功了。**具体编号因系统而异，需按环境核实**；看到 `socket create failed`，说明创建失败，先检查环境或权限。

<br>

### 易错点与边界

坑一：**Socket 不是协议**，是编程接口；UDP/TCP 才是传输层协议。

坑二：**`AF_INET` 是地址族，`SOCK_DGRAM/SOCK_STREAM` 才是传输层类型**，别把两者混成一个参数。

坑三：**应用层控制不了底层协议细节**，选了 UDP 就不能指望内核帮你重传。

坑四：C++ 里 socket 是文件描述符，用完要 `close`；错误时返回负值，别直接继续用。

<br>

### 一句话本质

**Socket 是应用进程操纵网络的“文件句柄/大门”；选什么套接字类型，决定你走 UDP 还是 TCP。**

<br>

### 自测

`AF_INET` 和 `SOCK_DGRAM` 分别代表什么？它们解决的是同一个层面的问题吗？

<details><summary>答案</summary>`AF_INET` 是地址族，表示 IPv4；`SOCK_DGRAM` 是套接字类型，表示 UDP。两者不是同一个层面：地址族决定用哪种地址格式，套接字类型决定走哪种传输层协议。</details>

<br>

<br>

## 知识点 2：UDP——贴地址就扔，回信地址从 recvfrom 记
***
### 问题

不怕丢的实时消息怎么写？为什么 UDP 客户端不用先 `connect`？服务端回信时地址从哪来？

**你可能会想**：UDP 客户端不 `connect`，那服务端怎么知道回信给谁？

靠 `recvfrom`。**每个 UDP 数据报都自带源地址，服务端收到时从 `recvfrom` 的参数里拿到客户端地址，再用这个地址 `sendto` 回信。**

<br>

### 核心概念与方案

先把 UDP 换成你熟悉的场景：**UDP 像寄明信片，贴上地址就扔出去，能不能收到看命；对方回信时，地址从你寄来的明信片上抄。**

UDP 是无连接的：

- 客户端直接 `sendto(数据, 目的地址)` 把数据报扔出去。
- 服务端 `bind` 到本地端口后，用 `recvfrom` 阻塞收，并从参数里拿到 `clientAddr`。
- 服务端再用 `clientAddr` `sendto` 回信。

UDP 客户端不需要 `connect`，因为每个数据报都自带目的地址；客户端源端口由系统临时分配。服务端回信必须用 `recvfrom` 记下的客户端地址，不能自己猜。

<br>

### 最小实验：只看服务端有没有把收到的小写转成大写回给客户端

环境前提：同上。下面两个程序只在本机回环地址上运行，不对外网开放。

UDP 服务端，保存为 `udp_server.cpp`：

````cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <algorithm>
#include <cctype>
#include <iostream>
#include <string>

int main() {
    const int serverPort = 12000;

    int serverSocket = socket(AF_INET, SOCK_DGRAM, 0);
    if (serverSocket < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    sockaddr_in serverAddr{};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(serverPort);

    if (bind(serverSocket, reinterpret_cast<sockaddr*>(&serverAddr), sizeof(serverAddr)) < 0) {
        std::cerr << "bind failed\n";
        close(serverSocket);
        return 1;
    }

    std::cout << "UDP server ready on port " << serverPort << "\n";

    char buffer[2048];
    while (true) {
        sockaddr_in clientAddr{};
        socklen_t clientLen = sizeof(clientAddr);

        ssize_t n = recvfrom(serverSocket, buffer, sizeof(buffer) - 1, 0,
                             reinterpret_cast<sockaddr*>(&clientAddr), &clientLen);
        if (n <= 0) continue;
        buffer[n] = '\0';

        std::string message(buffer);
        std::transform(message.begin(), message.end(), message.begin(),
                       [](unsigned char c) { return std::toupper(c); });

        sendto(serverSocket, message.data(), message.size(), 0,
               reinterpret_cast<sockaddr*>(&clientAddr), clientLen);
    }

    close(serverSocket);
    return 0;
}
````

UDP 客户端，保存为 `udp_client.cpp`：

````cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <iostream>
#include <string>

int main(int argc, char* argv[]) {
    const char* serverName = "127.0.0.1";
    const int serverPort = 12000;

    std::string message = (argc > 1) ? argv[1] : "hello";

    int clientSocket = socket(AF_INET, SOCK_DGRAM, 0);
    if (clientSocket < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    sockaddr_in serverAddr{};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(serverPort);
    if (inet_pton(AF_INET, serverName, &serverAddr.sin_addr) <= 0) {
        std::cerr << "invalid address\n";
        close(clientSocket);
        return 1;
    }

    sendto(clientSocket, message.data(), message.size(), 0,
           reinterpret_cast<sockaddr*>(&serverAddr), sizeof(serverAddr));

    char buffer[2048];
    sockaddr_in fromAddr{};
    socklen_t fromLen = sizeof(fromAddr);

    ssize_t n = recvfrom(clientSocket, buffer, sizeof(buffer) - 1, 0,
                         reinterpret_cast<sockaddr*>(&fromAddr), &fromLen);
    if (n > 0) {
        buffer[n] = '\0';
        std::cout << buffer << "\n";
    }

    close(clientSocket);
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 udp_server.cpp -o udp_server
g++ -std=c++17 -O2 udp_client.cpp -o udp_client

# 终端 1
./udp_server

# 终端 2
./udp_client hello
````

**这次只看一件事**：客户端终端有没有输出 `HELLO`。

预期输出：

````text
HELLO
````

有 `HELLO`，说明服务端收到小写、转成大写、用 `recvfrom` 记下的客户端地址回信成功。如果客户端卡住不输出，先检查服务端是否在运行、端口是否一致。

<br>

### 易错点与边界

坑一：**客户端端口由系统临时分配**，不是固定值；服务端回信要用 `recvfrom` 返回的地址。

坑二：**服务端回信必须用 `recvfrom` 记下的 `clientAddr`**，不能自己编一个。

坑三：**`bind(INADDR_ANY, port)` 表示绑定所有本地接口**，不是只绑 localhost。

坑四：C++ 里 `recvfrom` 返回字节数，不是字符串长度；要手动补 `'\0'` 再构造 `std::string`。

<br>

### 一句话本质

**UDP 是贴上地址就扔出去，能不能收到看命；回信地址得从 `recvfrom` 记下的客户端地址里拿。**

<br>

### 自测

UDP 服务端为什么必须用 `recvfrom` 返回的客户端地址回信？如果自己随便填一个地址会怎样？

<details><summary>答案</summary>UDP 客户端源端口由系统临时分配，服务端事先不知道。`recvfrom` 会返回数据报的源地址和源端口，服务端必须用这个地址回信，客户端才能收到。自己随便填地址，回信会发到错误的主机或端口，客户端收不到。</details>

<br>

<br>

## 知识点 3：TCP——先接通再说话，accept 派专属客服
***
### 问题

不能丢的数据怎么写？为什么 TCP 服务端要 `listen` 和 `accept`？`accept` 返回的为什么是新套接字？

**你可能会想**：TCP 服务端 `accept` 之后，直接用 `serverSocket` 收发不就行了，为什么还要一个新套接字？

因为 `serverSocket` 还要继续监听、接待下一个客户端。**`accept` 返回的新套接字，才是专门服务这个客户端的“专属客服”。**

<br>

### 核心概念与方案

先把 TCP 换成你熟悉的场景：**TCP 是服务端设前台，客户端打电话，接通后派专属客服。**

TCP 面向连接。服务端流程是：

- `socket` → `bind` → `listen` → `accept`。
- `accept` 返回一个**新的连接套接字**，专门服务该客户端。
- 原来的 `serverSocket` 继续监听、接待下一个。

客户端 `connect` 后直接用 `send/recv`，不需要每次写对方地址。

三次握手在 `connect`/`accept` 底层完成；`listen` 的参数是排队上限，不是“最多服务多少个连接”。真正通信的是 `accept` 返回的新套接字。

<br>

### 最小实验：只看 accept 返回的新套接字能不能收到大写回信

环境前提：同上。下面两个程序只在本机回环地址上运行。

TCP 服务端，保存为 `tcp_server.cpp`：

````cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <algorithm>
#include <cctype>
#include <iostream>
#include <string>

int main() {
    const int serverPort = 12001;

    int serverSocket = socket(AF_INET, SOCK_STREAM, 0);
    if (serverSocket < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    int opt = 1;
    setsockopt(serverSocket, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in serverAddr{};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(serverPort);

    if (bind(serverSocket, reinterpret_cast<sockaddr*>(&serverAddr), sizeof(serverAddr)) < 0) {
        std::cerr << "bind failed\n";
        close(serverSocket);
        return 1;
    }

    if (listen(serverSocket, 5) < 0) {
        std::cerr << "listen failed\n";
        close(serverSocket);
        return 1;
    }

    std::cout << "TCP server ready on port " << serverPort << "\n";

    while (true) {
        sockaddr_in clientAddr{};
        socklen_t clientLen = sizeof(clientAddr);

        int connectionSocket = accept(serverSocket,
                                      reinterpret_cast<sockaddr*>(&clientAddr),
                                      &clientLen);
        if (connectionSocket < 0) continue;

        char buffer[1024];
        ssize_t n = recv(connectionSocket, buffer, sizeof(buffer) - 1, 0);
        if (n > 0) {
            buffer[n] = '\0';
            std::string sentence(buffer);
            std::transform(sentence.begin(), sentence.end(), sentence.begin(),
                           [](unsigned char c) { return std::toupper(c); });
            send(connectionSocket, sentence.data(), sentence.size(), 0);
        }

        close(connectionSocket);
    }

    close(serverSocket);
    return 0;
}
````

TCP 客户端，保存为 `tcp_client.cpp`：

````cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <iostream>
#include <string>

int main(int argc, char* argv[]) {
    const char* serverName = "127.0.0.1";
    const int serverPort = 12001;

    std::string sentence = (argc > 1) ? argv[1] : "hello tcp";

    int clientSocket = socket(AF_INET, SOCK_STREAM, 0);
    if (clientSocket < 0) {
        std::cerr << "socket failed\n";
        return 1;
    }

    sockaddr_in serverAddr{};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(serverPort);
    if (inet_pton(AF_INET, serverName, &serverAddr.sin_addr) <= 0) {
        std::cerr << "invalid address\n";
        close(clientSocket);
        return 1;
    }

    if (connect(clientSocket, reinterpret_cast<sockaddr*>(&serverAddr), sizeof(serverAddr)) < 0) {
        std::cerr << "connect failed\n";
        close(clientSocket);
        return 1;
    }

    send(clientSocket, sentence.data(), sentence.size(), 0);

    char buffer[2048];
    ssize_t n = recv(clientSocket, buffer, sizeof(buffer) - 1, 0);
    if (n > 0) {
        buffer[n] = '\0';
        std::cout << "From Server: " << buffer << "\n";
    }

    close(clientSocket);
    return 0;
}
````

编译运行：

````bash
g++ -std=c++17 -O2 tcp_server.cpp -o tcp_server
g++ -std=c++17 -O2 tcp_client.cpp -o tcp_client

# 终端 1
./tcp_server

# 终端 2
./tcp_client "hello tcp"
````

**这次只看一件事**：客户端终端有没有输出 `From Server: HELLO TCP`。

预期输出：

````text
From Server: HELLO TCP
````

有输出，说明 `connect` 建连成功，服务端 `accept` 返回新套接字，收到数据后转大写回给客户端。如果 `connect failed`，先检查服务端是否在运行、端口是否一致。

<br>

### 易错点与边界

坑一：**`accept` 不是建立连接**，三次握手在底层已完成；`accept` 是从已完成队列取一个连接。

坑二：**真正通信的是 `accept` 返回的新套接字**，原 `serverSocket` 继续监听。

坑三：**`listen` 的参数是排队上限**，不是连接数上限，也不是并发服务数上限。

坑四：C++ 里 `send` 不保证一次发完、`recv` 不保证一次收全，示例是短消息简化；真实程序要循环处理或加长度前缀。

坑五：服务端重启常遇到 `Address already in use`，示例用 `SO_REUSEADDR` 缓解，但真正排查还要看端口占用。

<br>

### 一句话本质

**TCP 是服务端设前台，客户端打电话，接通后派专属客服；`accept` 返回的新套接字才是真正收发的连接，原套接字继续监听。**

<br>

### 自测

为什么 TCP 服务端 `accept` 之后，`recv/send` 不需要再写客户端 IP 和端口？

<details><summary>答案</summary>因为 `accept` 返回的新套接字已经绑定了这次连接的双方地址和端口，连接已经建立。之后在这个连接套接字上 `recv/send`，内核知道数据属于哪条连接，不需要每次再写对端地址。原 `serverSocket` 继续监听，接待下一个客户端。</details>

<br>

<br>

## 完整可复现示例：同时运行 UDP 和 TCP 回声
***
你需要 Linux/macOS 或 WSL，已安装 `g++`。下面把四个程序都编译，分别运行。

````bash
g++ -std=c++17 -O2 udp_server.cpp -o udp_server
g++ -std=c++17 -O2 udp_client.cpp -o udp_client
g++ -std=c++17 -O2 tcp_server.cpp -o tcp_server
g++ -std=c++17 -O2 tcp_client.cpp -o tcp_client
````

运行：

````bash
# 终端 1：UDP 服务端
./udp_server

# 终端 2：UDP 客户端
./udp_client hello

# 终端 3：TCP 服务端
./tcp_server

# 终端 4：TCP 客户端
./tcp_client "hello tcp"
````

可能的输出/现象：

````text
UDP 客户端终端：
HELLO

TCP 客户端终端：
From Server: HELLO TCP
````

**关键观察点**：UDP 客户端有没有输出 `HELLO`；TCP 客户端有没有输出 `From Server: HELLO TCP`。

验证结果：UDP 无连接直接发，服务端从 `recvfrom` 拿地址回信；TCP 先 `connect` 建连，服务端 `accept` 后派新套接字通信。

注意：UDP 用 12000，TCP 用 12001，避免端口冲突；TCP 和 UDP 端口空间按协议分开，理论上同端口也能同时监听，但示例分开更直观。实际程序还要处理信号、超时、多客户端并发和错误码，这里只保留最小可运行骨架，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| `bind failed` | 端口是否被占用、权限 | `ss -lunp \| grep ':12000'`；`ss -ltnp \| grep ':12001'` | 端口被占或权限不足；换端口或停掉占用进程 |
| `connect failed` | 服务端是否启动、地址端口是否对 | `ss -ltnp \| grep ':12001'`；`curl -v` 测端口 | 服务端没起、端口写错、防火墙挡住都可能失败 |
| UDP 客户端卡住不输出 | 服务端是否运行、端口是否一致 | 看服务端是否打印 ready；`ss -lunp` | UDP 无连接，服务端没起时客户端收不到回信，会一直等 |
| TCP 客户端卡住不输出 | 服务端是否 `accept`、是否回信 | 看服务端日志；`ss -tan` | 服务端可能没走到 `send`，或 `recv` 没收到完整数据 |
| `Address already in use` | 端口占用、TIME_WAIT | `ss -ltnp \| grep ':端口'`；`lsof -i :端口` | 换端口，或等 TIME_WAIT 结束；`SO_REUSEADDR` 可缓解但不能解决所有情况 |
| 收到乱码或截断 | 字节数、`'\0'`、粘包 | 看 `recv` 返回值；加长度前缀 | `recv` 返回字节数，不是字符串长度；TCP 是字节流，短消息示例简化了边界处理 |
| 多客户端只服务一个 | `accept` 是否循环、是否并发 | 看服务端是否在 `while` 里 `accept`；查并发模型 | 示例是单线程阻塞，一次只服务一个连接；多客户端要循环 `accept` 或加并发 |

<br>

<br>

## 常见误区
***
- **把 Socket 当成协议**：Socket 是编程接口，UDP/TCP 才是传输层协议。
- **把 `AF_INET` 和 `SOCK_DGRAM` 混成一个参数**：前者是地址族，后者是套接字类型。
- **以为 UDP 客户端必须 `connect`**：UDP 客户端可以直接 `sendto`；`connect` 只是绑定默认对端，不是握手。
- **服务端回信自己编地址**：必须用 `recvfrom` 返回的客户端地址。
- **以为 `accept` 建立连接**：三次握手在底层已完成，`accept` 只是从队列取连接。
- **用 `serverSocket` 直接收发**：真正通信的是 `accept` 返回的新套接字，原套接字继续监听。
- **以为 `listen` 参数是最大连接数**：它是排队上限，不是并发服务数上限。
- **以为 `send` 一次发完、`recv` 一次收全**：TCP 是字节流，短消息示例是简化，真实程序要处理边界。

<br>

<br>

## 自测三问
***
1. UDP 客户端为什么不需要 `connect`？服务端回信地址从哪来？
2. TCP 服务端 `listen` 和 `accept` 分别做什么？`accept` 返回的为什么是新套接字？
3. 为什么说 Socket 是 API 而不是协议？`AF_INET` 和 `SOCK_STREAM` 各管什么？

<details><summary>参考答案</summary>1. UDP 是无连接的，每个数据报自带目的地址，客户端可以直接 `sendto`；服务端从 `recvfrom` 返回的客户端地址回信，不能自己猜。2. `listen` 让服务端进入监听状态，设置排队上限；`accept` 从已完成连接队列取出一个连接，返回新套接字专门服务该客户端。原 `serverSocket` 继续监听，接待下一个。3. Socket 是操作系统提供的网络编程接口，不是协议；`AF_INET` 是地址族，表示 IPv4；`SOCK_STREAM` 是套接字类型，表示 TCP。</details>

<br>

<br>

## 总结
***
**Socket 是应用层接入网络的大门：UDP 是“拿地址发快件”，无连接但快，回信地址从 `recvfrom` 记；TCP 是“接电话派客服”，必须 `connect`/`accept` 建连，`accept` 返回的新套接字才是真正通信的连接。**

<br>

<br>

## 参考资料
***
- POSIX.1-2017：System Interfaces（socket、bind、listen、accept、sendto、recvfrom 等）
- RFC 768：User Datagram Protocol
- RFC 9293：Transmission Control Protocol
- `man 7 socket`、`man 2 socket`、`man 2 bind`、`man 2 listen`、`man 2 accept`、`man 2 sendto`、`man 2 recvfrom`
- 需按环境核实：套接字编号因系统而异；`SO_REUSEADDR` 在不同系统上的行为差异；`listen` 参数和半连接/全连接队列的具体关系因内核实现而异；示例是单线程阻塞骨架，多客户端并发、超时、信号处理需另行设计。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>