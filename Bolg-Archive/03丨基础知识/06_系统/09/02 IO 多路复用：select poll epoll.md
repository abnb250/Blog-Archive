# 02 I/O 多路复用：select/poll/epoll
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击返回目录](https://www.cnblogs.com/abnb250/articles/23373186)  

<br>

<br>

## 痛点引入
***
一个服务器只能服务一个客户端？多进程/多线程每连接一个又扛不住 C10K。**I/O 多路复用让一个进程用一个系统调用同时监听多个 socket，有事件才处理，没事件就阻塞，不用轮询、不用每连接开线程。**

<br>

<br>

## 知识点 1：基本 TCP Socket 模型
***
### 问题
***
服务端怎么从监听连接走到收发数据？

<br>

### 方案
***
服务端：`socket → bind → listen → accept`；客户端：`socket → connect`。内核为监听 socket 维护**半连接队列**和**全连接队列**；`accept` 从全连接队列取已连接 socket。**监听 socket 和已连接 socket 是两个 socket。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

int main() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    addr.sin_port = htons(8080);

    bind(listen_fd, (sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 5);
    std::cout << "listening on 127.0.0.1:8080\n";
    close(listen_fd);
}
```
编译运行：
```bash
g++ mux1.cpp -o mux1
./mux1
```
输出：
```text
listening on 127.0.0.1:8080
```
**易错点：** `bind` 绑定 IP 和端口，`listen` 只把 socket 变监听态；`accept` 返回的是新 socket，别拿监听 fd 收发数据。

<br>

### 一句话总结
***
**TCP 服务端 socket→bind→listen→accept，监听 socket 负责接客，已连接 socket 负责服务。**

<br>

### 自测
***
自测：为什么 `accept` 返回的 socket 和监听 socket 不是同一个？

<br>

<br>

## 知识点 2：多进程/多线程模型的瓶颈
***
### 问题
***
每来一个连接 fork 一个进程或创建一个线程，10000 连接会怎样？

<br>

### 方案
***
多进程每连接一个进程，上下文切换重、内存占用大，还要处理僵尸进程；多线程轻一些，但每连接一线程仍受线程数和调度限制。线程池复用线程能缓解创建销毁开销，但连接数上万时仍不划算。**每连接一进程/线程，扛不住 C10K。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <thread>
#include <vector>

void handle(int id) {
    std::cout << "handle conn " << id << "\n";
}

int main() {
    std::vector<std::thread> ts;
    for (int i = 0; i < 5; ++i) ts.emplace_back(handle, i);
    for (auto& t : ts) t.join();
}
```
编译运行：
```bash
g++ mux2.cpp -o mux2 -pthread
./mux2
```
**易错点：** 示例只创建 5 个线程；真实每连接一线程时，线程栈和调度开销会随连接数爆炸。

<br>

### 一句话总结
***
**每连接一进程/线程简单但重，连接数一上来，创建、切换和内存都成瓶颈。**

<br>

### 自测
***
自测：多线程模型相比多进程模型，主要省在哪部分开销？

<br>

<br>

## 知识点 3：select 和 poll
***
### 问题
***
能不能一个线程同时等多个 socket，有事件再处理？

<br>

### 方案
***
`select` 把 fd 集合拷到内核，内核遍历检查，有事件标记后拷回用户态，用户态再遍历找可读可写。**2 次拷贝 + 2 次遍历**，fd 数受 `FD_SETSIZE` 限制（常 1024）。`poll` 用动态数组突破数量限制，但仍是线性结构、遍历 O(n)、用户态内核态拷贝。**select/poll 本质都是“线性扫集合”。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <sys/select.h>
#include <unistd.h>

int main() {
    fd_set readfds;
    FD_ZERO(&readfds);
    FD_SET(STDIN_FILENO, &readfds);

    timeval tv{2, 0};
    int n = select(STDIN_FILENO + 1, &readfds, nullptr, nullptr, &tv);
    std::cout << "select returned " << n << "\n";
    if (n > 0 && FD_ISSET(STDIN_FILENO, &readfds)) {
        std::cout << "stdin readable\n";
    }
}
```
编译运行：
```bash
g++ mux3.cpp -o mux3
./mux3
```
2 秒内输入回车会输出 `stdin readable`，否则 `select returned 0`。
**易错点：** `select` 会修改 fd 集合，每次调用前要重新 `FD_ZERO/FD_SET`；返回可读不代表 `read` 一定不阻塞，最好配非阻塞 I/O。

<br>

### 一句话总结
***
**select/poll 用一个系统调用等多个 fd，但每次都要拷集合、线性遍历，fd 多了就慢。**

<br>

### 自测
***
自测：select 和 poll 最本质的共同缺陷是什么？

<br>

<br>

## 知识点 4：epoll 的红黑树与就绪链表
***
### 问题
***
fd 上万时，每次拷贝整个集合、遍历所有 fd 太浪费，能不能只关注“有事件的”？

<br>

### 方案
***
epoll 三接口：`epoll_create` 建实例，`epoll_ctl` 增删改关注 fd（内核红黑树，O(log n)），`epoll_wait` 等事件。内核用**就绪链表**记录有事件的 fd，`epoll_wait` 只返回就绪数量，不用遍历全集合。**红黑树管注册，就绪链表管事件，避免全量拷贝和扫描。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <sys/epoll.h>
#include <unistd.h>

int main() {
    int epfd = epoll_create1(0);
    epoll_event ev{};
    ev.events = EPOLLIN;
    ev.data.fd = STDIN_FILENO;
    epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &ev);

    epoll_event events[4];
    int n = epoll_wait(epfd, events, 4, 2000);
    std::cout << "epoll_wait returned " << n << "\n";
    close(epfd);
}
```
编译运行：
```bash
g++ mux4.cpp -o mux4
./mux4
```
2 秒内输入回车会返回 1，否则返回 0。
**易错点：** `epoll_wait` 返回的是就绪事件数，不是 fd；`epoll_ctl` 的 `data` 要自己维护映射。

<br>

### 一句话总结
***
**epoll 用红黑树注册 fd、就绪链表收事件，只返回有事件的 fd，避免全量拷贝和遍历。**

<br>

### 自测
***
自测：epoll 为什么比 select/poll 更适合大量连接？

<br>

<br>

## 知识点 5：水平触发与边缘触发
***
### 问题
***
epoll 的 LT 和 ET 到底差在哪？为什么 ET 常配非阻塞 I/O？

<br>

### 方案
***
**水平触发 LT**：只要缓冲区还有数据，`epoll_wait` 就一直通知，直到读完。**边缘触发 ET**：只在状态变化时通知一次，没读完也不再提醒，所以必须循环读到 `EAGAIN`。**ET 通知少、效率高，但要非阻塞 + 读干净；LT 编程简单，默认模式。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <sys/epoll.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int epfd = epoll_create1(0);
    epoll_event ev{};
    ev.events = EPOLLIN | EPOLLET;
    ev.data.fd = STDIN_FILENO;
    epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &ev);

    int flags = fcntl(STDIN_FILENO, F_GETFL);
    fcntl(STDIN_FILENO, F_SETFL, flags | O_NONBLOCK);

    epoll_event events[4];
    int n = epoll_wait(epfd, events, 4, 2000);
    std::cout << "ET epoll_wait returned " << n << "\n";
    close(epfd);
}
```
**易错点：** ET 模式下只读一次可能留下数据，必须循环 `read` 到 `EAGAIN`；LT 不读会反复通知，可能空转。

<br>

### 一句话总结
***
**LT 有数据就一直通知，ET 只通知一次；ET 高效但要非阻塞并循环读干净。**

<br>

### 自测
***
自测：ET 模式下为什么必须把 socket 设为非阻塞？

<br>

<br>

## 完整可运行/可复现示例
***
```cpp
#include <iostream>
#include <sys/epoll.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int epfd = epoll_create1(0);
    epoll_event ev{};
    ev.events = EPOLLIN;
    ev.data.fd = STDIN_FILENO;
    epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &ev);

    epoll_event events[8];
    int n = epoll_wait(epfd, events, 8, 3000);
    std::cout << "ready events=" << n << "\n";

    for (int i = 0; i < n; ++i) {
        if (events[i].data.fd == STDIN_FILENO) {
            char buf[64];
            int r = read(STDIN_FILENO, buf, sizeof(buf) - 1);
            if (r > 0) {
                buf[r] = 0;
                std::cout << "read: " << buf << "\n";
            }
        }
    }
    close(epfd);
}
```
编译运行：
```bash
g++ mux_all.cpp -o mux_all
./mux_all
# 输入 hello 回车
```
输出类似：
```text
ready events=1
read: hello
```
**注意：** 示例只监听 stdin；真实服务器要管理多个 socket 和连接生命周期。

<br>

<br>

## 自测三问
***
1. select/poll 的两次拷贝和两次遍历分别是什么？
2. epoll 的红黑树和就绪链表各解决什么问题？
3. LT 和 ET 的区别是什么？ET 为什么配非阻塞？

<br>

<br>

## 一句话总结全章
***
**select/poll 线性扫集合、全量拷贝；epoll 用红黑树注册、就绪链表收事件，只返回有事件的 fd；LT 反复通知，ET 只通知一次但要非阻塞读干净。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>