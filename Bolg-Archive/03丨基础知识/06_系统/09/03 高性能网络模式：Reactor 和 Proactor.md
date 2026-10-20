# 03 高性能网络模式：Reactor 和 Proactor
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击返回目录](https://www.cnblogs.com/abnb250/articles/23373186)  

<br>

<br>

## 痛点引入
***
用 epoll 写网络程序，代码是面向过程的，事件来了手动 dispatch，业务一复杂就乱。**Reactor 把“监听事件 + 分发处理”封装成模式；Proactor 更进一步，让内核把读写都做完再通知你。** Redis、Nginx、Netty 都绕不开这两个模式。

<br>

<br>

## 知识点 1：从线程池到 I/O 多路复用
***
### 问题
***
每连接一线程扛不住，线程池里一个线程管多个连接，但 `read` 阻塞会卡住整个线程，怎么办？

<br>

### 方案
***
把 socket 设非阻塞后轮询 `read` 浪费 CPU；**I/O 多路复用**用一个系统调用监听多个连接，没事件就阻塞，有事件才返回。Reactor 就是基于 I/O 多路复用的封装。**多路复用解决“等谁”，Reactor 解决“事件来了怎么分发”。**

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
    std::cout << "events=" << n << "\n";
    close(epfd);
}
```
编译运行：
```bash
g++ rea1.cpp -o rea1
./rea1
```
**易错点：** 这只是多路复用，不是完整 Reactor；真实 Reactor 还要 Acceptor、Handler、dispatch。

<br>

### 一句话总结
***
**I/O 多路复用负责高效等事件，Reactor 负责把事件分发给对应处理器。**

<br>

### 自测
***
自测：线程池方案里，一个线程管多个连接为什么会阻塞？

<br>

<br>

## 知识点 2：单 Reactor 单进程/线程
***
### 问题
***
最简单的 Reactor 怎么组织？谁监听、谁 accept、谁处理业务？

<br>

### 方案
***
Reactor 用 select/epoll 监听事件并 dispatch；连接事件交给 **Acceptor** `accept` 并创建 **Handler**；读写事件交给 Handler 走 `read → 业务处理 → send`。全部在一个进程/线程里。**实现简单、无进程通信和锁，但无法利用多核，业务耗时会阻塞其他连接。** Redis 就是单 Reactor 单进程。

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <functional>
#include <string>

struct Handler {
    void on_read() { std::cout << "handler: read -> process -> send\n"; }
};

struct Acceptor {
    Handler* accept() {
        std::cout << "acceptor: accept new connection\n";
        static Handler h;
        return &h;
    }
};

int main() {
    std::string event = "connection";
    if (event == "connection") {
        Acceptor a;
        Handler* h = a.accept();
        h->on_read();
    }
}
```
编译运行：
```bash
g++ rea2.cpp -o rea2
./rea2
```
输出：
```text
acceptor: accept new connection
handler: read -> process -> send
```
**易错点：** 教学模拟不是真实网络代码；单线程里 Handler 业务一慢，所有连接都等。

<br>

### 一句话总结
***
**单 Reactor 单进程：一个 Reactor 监听分发，Acceptor 接连接，Handler 做业务，简单但吃不满多核。**

<br>

### 自测
***
自测：单 Reactor 单进程为什么不适合计算密集型业务？

<br>

<br>

## 知识点 3：单 Reactor 多线程
***
### 问题
***
业务处理耗时，单线程 Reactor 被卡住，怎么利用多核又不破坏事件循环？

<br>

### 方案
***
Reactor 仍单线程监听和分发；Handler 只负责 `read/send`，把业务交给子线程 **Processor** 处理，处理完再交回 Handler 发送。**利用多核，但共享数据要加锁；单 Reactor 仍可能成为高并发瓶颈。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <string>

std::mutex m;

void processor(std::string data) {
    std::string result = "processed:" + data;
    std::lock_guard<std::mutex> lock(m);
    std::cout << result << "\n";
}

int main() {
    std::string data = "request";
    std::thread t(processor, data);
    t.join();
    std::cout << "handler sends response\n";
}
```
编译运行：
```bash
g++ rea3.cpp -o rea3 -pthread
./rea3
```
输出：
```text
processed:request
handler sends response
```
**易错点：** 子线程处理完把结果交回主线程发送，涉及共享数据竞争，要加锁或消息队列。

<br>

### 一句话总结
***
**单 Reactor 多线程：Reactor 管事件，Handler 管收发，Processor 子线程管业务，多核用上了但共享数据要同步。**

<br>

### 自测
***
自测：单 Reactor 多线程里，为什么还需要给共享数据加锁？

<br>

<br>

## 知识点 4：多 Reactor 多进程/线程
***
### 问题
***
单 Reactor 在高并发下仍是瓶颈，怎么把 accept 和读写分散到多个 Reactor？

<br>

### 方案
***
**MainReactor** 只监听连接，accept 后把新连接分配给某个 **SubReactor**；SubReactor 监听该连接的读写事件并创建 Handler 处理。主从分工明确，主线程只传连接，子线程直接回客户端。**Netty/Memcache 用多 Reactor 多线程，Nginx 用类似多 Reactor 多进程。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <thread>
#include <vector>

void sub_reactor(int id) {
    std::cout << "sub reactor " << id << " handles connection\n";
}

int main() {
    std::cout << "main reactor accepts connection\n";
    std::vector<std::thread> subs;
    for (int i = 0; i < 2; ++i) subs.emplace_back(sub_reactor, i);
    for (auto& t : subs) t.join();
}
```
编译运行：
```bash
g++ rea4.cpp -o rea4 -pthread
./rea4
```
输出：
```text
main reactor accepts connection
sub reactor 0 handles connection
sub reactor 1 handles connection
```
**易错点：** 教学模拟；真实实现还要处理连接分配、惊群、负载均衡和生命周期。

<br>

### 一句话总结
***
**多 Reactor：MainReactor 接连接，SubReactor 管读写，主从分工把高并发压力分散开。**

<br>

### 自测
***
自测：多 Reactor 方案里 MainReactor 和 SubReactor 各负责什么？

<br>

<br>

## 知识点 5：Proactor 与异步 I/O
***
### 问题
***
Reactor 是“事件就绪通知你，你自己 read”；能不能内核把数据读完再通知我？

<br>

### 方案
***
**Reactor = 非阻塞同步**：感知可读可写，应用自己 `read/write`。**Proactor = 异步**：应用发起异步读写并传缓冲区，内核完成“数据准备 + 拷贝到用户空间”后通知，应用直接处理。Linux 网络 socket 的 POSIX AIO 支持不完善，高性能网络多用 Reactor；Windows IOCP 是真正异步。**Reactor 通知“待完成”，Proactor 通知“已完成”。**

<br>

### 最小示例
***
```cpp
#include <iostream>

int main() {
    std::cout << "Reactor: fd readable -> app calls read -> app processes\n";
    std::cout << "Proactor: app submits async read -> kernel fills buffer -> app processes\n";
}
```
编译运行：
```bash
g++ rea5.cpp -o rea5
./rea5
```
输出：
```text
Reactor: fd readable -> app calls read -> app processes
Proactor: app submits async read -> kernel fills buffer -> app processes
```
**易错点：** Linux 的 `aio` 对 socket 支持有限，别以为用了 `aio_*` 就是网络 Proactor；IOCP 才是 Windows 真异步。

<br>

### 一句话总结
***
**Reactor 通知就绪、应用自己读写；Proactor 内核完成读写、通知应用直接处理。**

<br>

### 自测
***
自测：为什么 Linux 高性能网络程序大多用 Reactor 而不是 Proactor？

<br>

<br>

## 完整可运行/可复现示例
***
```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <mutex>

std::mutex m;

void sub_reactor(int id) {
    std::lock_guard<std::mutex> lock(m);
    std::cout << "sub reactor " << id << ": read -> process -> send\n";
}

int main() {
    std::cout << "main reactor: accept connection\n";
    std::vector<std::thread> subs;
    for (int i = 0; i < 3; ++i) subs.emplace_back(sub_reactor, i);
    for (auto& t : subs) t.join();
}
```
编译运行：
```bash
g++ rea_all.cpp -o rea_all -pthread
./rea_all
```
**注意：** 模拟主从 Reactor 分工，不是真实 epoll 服务器。

<br>

<br>

## 自测三问
***
1. Reactor 模式里 Reactor、Acceptor、Handler 各负责什么？
2. 单 Reactor 多线程和多 Reactor 多线程分别解决什么问题？
3. Reactor 和 Proactor 的核心区别是什么？

<br>

<br>

## 一句话总结全章
***
**Reactor 基于 I/O 多路复用，事件就绪后分发给 Handler，应用自己读写；单 Reactor 简单、多线程用多核、多 Reactor 分散高并发；Proactor 让内核完成读写再通知，是异步模式。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>