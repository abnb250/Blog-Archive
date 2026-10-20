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
如果让你从零写一个网络发消息程序，怎么下手？为什么 UDP 客户端不用先连接？为什么 TCP 服务端要 `listen` 和 `accept`？  
这一章咱们用问题链往前走：**Socket 是什么 → UDP 怎么写 → TCP 怎么写 → 两者差在哪。**  
先别急着背 `bind`、`listen`、`accept` 这些 API，先看一个具体场景：你想写一个“收到小写字母就回大写”的服务端。用 UDP 写，客户端直接发就行；用 TCP 写，服务端却要先监听、再接受连接。为什么一个不用连、一个必须连？这背后就是传输层给应用层暴露的不同“大门”。

<br>

<br>

## 知识点 1：Socket 是应用层和传输层的大门
***
### 问题
应用进程怎么把数据交给传输层？为什么说 Socket 不是协议，而是 API？

<br>

### 方案
Socket 是操作系统提供的一组网络编程接口，不是物理硬件，也不是某个协议。你创建套接字时指定两件事：  
- **地址族**：`AF_INET` 表示 IPv4；  
- **套接字类型**：`SOCK_DGRAM` 是 UDP，`SOCK_STREAM` 是 TCP。  
应用层只能选套接字类型，不能改底层协议机制。选了 UDP，就要自己处理丢包和消息边界；选了 TCP，三次握手、重传、按序交付由内核完成。  
C++ 里这些接口来自 POSIX socket API，需要包含 `<sys/socket.h>`、`<netinet/in.h>`、`<arpa/inet.h>`、`<unistd.h>` 等头文件；Windows 下要改用 Winsock，这里以 Linux/POSIX 为例。

<br>

### 最小示例
```cpp
#include <sys/socket.h>
#include <netinet/in.h>
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
```

编译运行：

```bash
g++ -std=c++17 -O2 socket_demo.cpp -o socket_demo
./socket_demo
```

输出可能类似：

```text
udp_sock=3, tcp_sock=4
```

结论：Socket 就是应用进程操纵网络的“文件句柄/大门”，选什么类型，决定你走 UDP 还是 TCP。

<br>

### 一句话本质
**Socket 是应用进程操纵网络的“文件句柄/大门”。**

<br>

### 易错点
坑一：**Socket 不是协议**，是编程接口；UDP/TCP 才是传输层协议。  
坑二：**`AF_INET` 是地址族，`SOCK_DGRAM/SOCK_STREAM` 才是传输层类型**，别把两者混成一个参数。  
坑三：**应用层控制不了底层协议细节**，选了 UDP 就不能指望内核帮你重传。  
坑四：C++ 里 socket 是文件描述符，用完要 `close`；错误时返回负值，别直接继续用。

<br>

### 自测
`AF_INET` 和 `SOCK_DGRAM` 分别代表什么？它们解决的是同一个层面的问题吗？

<br>

<br>

## 知识点 2：UDP 最小实现
***
### 问题
不怕丢的实时消息怎么写？为什么 UDP 客户端不用先 `connect`？服务端回信时地址从哪来？

<br>

### 方案
UDP 是无连接的。客户端直接 `sendto(数据, 目的地址)` 把数据报扔出去；服务端 `bind` 到本地端口后，用 `recvfrom` 阻塞收，并从参数里拿到 `clientAddress`，再用这个地址 `sendto` 回信。  
UDP 客户端不需要 `connect`，因为每个数据报都自带目的地址；客户端源端口由系统临时分配。服务端回信必须用 `recvfrom` 记下的客户端地址，不能自己猜。  
下面代码用固定端口 12000，服务端把收到的小写字符串转成大写回给客户端。

<br>

### 最小示例
UDP 服务端：

```cpp
// udp_server.cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <algorithm>
#include <cctype>
#include <cstring>
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
        if (n <= 0) {
            continue;
        }
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
```

UDP 客户端：

```cpp
// udp_client.cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
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
```

编译运行：

```bash
g++ -std=c++17 -O2 udp_server.cpp -o udp_server
g++ -std=c++17 -O2 udp_client.cpp -o udp_client

# 终端 1
./udp_server

# 终端 2
./udp_client hello
```

输出：

```text
HELLO
```

结论：UDP 客户端不用 `connect`，直接 `sendto`；服务端从 `recvfrom` 拿到客户端地址再回信。

<br>

### 一句话本质
**UDP 是贴上地址就扔出去，能不能收到看命；回信地址得从收报里记。**

<br>

### 易错点
坑一：**客户端端口由系统临时分配**，不是固定值；服务端回信要用 `recvfrom` 返回的地址。  
坑二：**服务端回信必须用 `recvfrom` 记下的 `clientAddr`**，不能自己编一个。  
坑三：**`bind(INADDR_ANY, port)` 表示绑定所有本地接口**，不是只绑 localhost。  
坑四：C++ 里 `recvfrom` 返回字节数，不是字符串长度；要手动补 `'\0'` 再构造 `std::string`。

<br>

### 自测
UDP 服务端为什么必须用 `recvfrom` 返回的客户端地址回信？如果自己随便填一个地址会怎样？

<br>

<br>

## 知识点 3：TCP 最小实现
***
### 问题
不能丢的数据怎么写？为什么 TCP 服务端要 `listen` 和 `accept`？`accept` 返回的为什么是新套接字？

<br>

### 方案
TCP 面向连接。服务端流程是 `socket` → `bind` → `listen` → `accept`；`accept` 返回一个**新的连接套接字**专门服务该客户端，原来的 `serverSocket` 继续监听、接待下一个。客户端 `connect` 后直接用 `send/recv`，不需要每次写对方地址。  
三次握手在 `connect`/`accept` 底层完成；`listen` 的参数是已完成连接队列和半连接队列相关的排队上限，不是“最多服务多少个连接”。真正通信的是 `accept` 返回的新套接字。

<br>

### 最小示例
TCP 服务端：

```cpp
// tcp_server.cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <algorithm>
#include <cctype>
#include <cstring>
#include <iostream>
#include <string>

int main() {
    const int serverPort = 12000;

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
        if (connectionSocket < 0) {
            continue;
        }

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
```

TCP 客户端：

```cpp
// tcp_client.cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
#include <iostream>
#include <string>

int main(int argc, char* argv[]) {
    const char* serverName = "127.0.0.1";
    const int serverPort = 12000;

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
```

编译运行：

```bash
g++ -std=c++17 -O2 tcp_server.cpp -o tcp_server
g++ -std=c++17 -O2 tcp_client.cpp -o tcp_client

# 终端 1
./tcp_server

# 终端 2
./tcp_client "hello tcp"
```

输出：

```text
From Server: HELLO TCP
```

结论：TCP 客户端先 `connect` 建连，服务端 `accept` 后派一个新套接字专门通信；之后 `send/recv` 不用再写对方地址。

<br>

### 一句话本质
**TCP 是服务端设前台，客户端打电话，接通后派专属客服。**

<br>

### 易错点
坑一：**`accept` 不是建立连接**，三次握手在底层已完成；`accept` 是从已完成队列取一个连接。  
坑二：**真正通信的是 `accept` 返回的新套接字**，原 `serverSocket` 继续监听。  
坑三：**`listen` 的参数是排队上限**，不是连接数上限，也不是并发服务数上限。  
坑四：C++ 里 `send` 不保证一次发完、`recv` 不保证一次收全，示例是短消息简化；真实程序要循环处理或加长度前缀。  
坑五：服务端重启常遇到 `Address already in use`，示例用 `SO_REUSEADDR` 缓解，但真正排查还要看端口占用。

<br>

### 自测
为什么 TCP 服务端 `accept` 之后，`recv/send` 不需要再写客户端 IP 和端口？

<br>

<br>

## 完整可复现示例：运行 UDP 和 TCP 回声
***
把上面四个文件保存好，分别编译：

```bash
g++ -std=c++17 -O2 udp_server.cpp -o udp_server
g++ -std=c++17 -O2 udp_client.cpp -o udp_client
g++ -std=c++17 -O2 tcp_server.cpp -o tcp_server
g++ -std=c++17 -O2 tcp_client.cpp -o tcp_client
```

运行：

```bash
# 终端 1：UDP 服务端
./udp_server

# 终端 2：UDP 客户端
./udp_client hello

# 终端 3：TCP 服务端
./tcp_server

# 终端 4：TCP 客户端
./tcp_client "hello tcp"
```

输出/现象/结论：

```text
UDP 客户端终端：
HELLO

TCP 客户端终端：
From Server: HELLO TCP
```

验证结果：UDP 无连接直接发，服务端从 `recvfrom` 拿地址回信；TCP 先 `connect` 建连，服务端 `accept` 后派新套接字通信。  
注意：UDP 和 TCP 服务端都用了 12000 端口，但一个是 `SOCK_DGRAM`、一个是 `SOCK_STREAM`，端口空间按协议分开，所以可以同时监听；如果先跑 TCP 再跑 UDP，不冲突。实际程序还要处理信号、超时、多客户端并发和错误码，这里只保留最小可运行骨架。

<br>

<br>

## 自测三问
***
1. UDP 客户端为什么不需要 `connect`？服务端回信地址从哪来？  
2. TCP 服务端 `listen` 和 `accept` 分别做什么？`accept` 返回的为什么是新套接字？  
3. 为什么说 Socket 是 API 而不是协议？`AF_INET` 和 `SOCK_STREAM` 各管什么？

<br>

<br>

## 一句话总结全章
***
**Socket 是应用层接入网络的大门：UDP 是“拿地址发快件”，无连接但快；TCP 是“接电话派客服”，必须建连但可靠。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>