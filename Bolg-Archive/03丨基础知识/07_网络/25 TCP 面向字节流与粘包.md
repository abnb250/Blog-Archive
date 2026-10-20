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
TCP 是面向字节流，UDP 是面向报文。**为什么 TCP 会“粘包”？接收方怎么知道一条消息到哪结束？HTTP 为什么用回车换行做边界？**  
先别急着把“粘包”当成 TCP 的 bug，先看一个具体场景：你写了个聊天程序，客户端连续调用两次 `send`，一次发 `"Hi."`，一次发 `"I am Xiaolin"`。服务端调用一次 `recv`，可能直接收到 `"Hi.I am Xiaolin"`；也可能先收到 `"Hi.I am"`，再收到 `" Xiaolin"`。发送方明明分了两次，接收方却没法靠 TCP 自己区分“一条消息到哪结束”。这就是这一章要解决的问题：**TCP 只保证字节按序到达，不保证消息边界；边界得应用层自己划。**

<br>

<br>

## 知识点 1：TCP 字节流与 UDP 报文边界
***
### 问题
“面向字节流”到底什么意思？为什么 UDP 能靠一个报文区分一条消息，TCP 却不行？一次 `send()` 是不是就对应一次 `recv()`？

<br>

### 方案
UDP 发送时，操作系统不拆分用户消息，一个 UDP 报文就是一个用户消息边界；接收方读一个报文，就是完整消息。  
TCP 把数据看成**连续字节流**：用户消息可能被拆成多个 TCP 报文，也可能多个消息合在一个报文里。`send()` 只是把数据拷到内核发送缓冲区，什么时候发、怎么分组由 TCP 决定，应用层控制不了。  
所以接收方如果不知道消息边界，就会出现所谓的“粘包/拆包”解析问题。注意：**“粘包”不是 TCP 的 bug**，TCP 本来就不提供消息边界；这是应用层协议要解决的事。也不能假设一次 `send()` 对应一次 `recv()`，更不能假设一次 `recv()` 就拿到一条完整消息。

<br>

### 最小示例
```text
发“Hi.”和“I am Xiaolin”
可能收到：“Hi.I am Xiaolin”
也可能收到：“Hi.I am” + “ Xiaolin”
也可能收到：“Hi.” + “I am Xiaolin”（碰巧）
```

结论：TCP 只保证字节顺序，不保证你发几次、对方就收几次；消息边界要靠应用层自己定义。

<br>

### 一句话本质
**UDP 一个报文一条消息；TCP 是连续字节流，消息边界要应用层自己划。**

<br>

### 易错点
坑一：**“粘包”不是 TCP 的 bug**，是字节流没有消息边界；别把锅甩给 TCP。  
坑二：**不能假设一次 `send()` 对应一次 `recv()`**，发送缓冲区和接收缓冲区都会影响实际收发次数。  
坑三：**`send()` 成功不等于对方已收到**，只表示数据进了本机内核发送缓冲区。  
坑四：**TCP 不保留消息边界，但保留字节顺序**；你收到的字节顺序是对的，只是不知道一条消息到哪结束。

<br>

### 自测
为什么 TCP 不能像 UDP 那样靠一个报文区分一条消息？如果接收方连续调用 `recv`，一定能一次拿到一条完整消息吗？

<br>

<br>

## 知识点 2：三种分包方式
***
### 问题
应用层怎么划消息边界？固定长度、特殊字符、长度字段分别适合什么场景？HTTP 头部和正文之间的空行属于哪种方式？

<br>

### 方案
常见三种分包方式：  
1. **固定长度**：每条消息固定 N 字节，简单但不灵活；消息长短不一就会浪费空间，太短又不够用。  
2. **特殊字符边界**：用特殊分隔符划分消息，如 HTTP 用 `\r\n\r\n` 分隔头部和正文；内容里出现特殊字符要转义，否则会提前截断。  
3. **自定义消息结构：包头 + 包体**。包头固定大小，并包含包体长度；接收方先读包头，再按长度读指定字节数。这是最通用、最常用的方式。  
HTTP 头部和正文之间的空行属于**特殊字符边界**；正文边界则常用 `Content-Length` 或分块编码，属于“长度字段/分块”思路。  
一句话：定长、分隔符、长度字段，是应用层给字节流画边界的三种笔。

<br>

### 最小示例
用长度字段分包，结构可以长这样：

```c
struct MessageHeader {
    uint32_t message_length;  // 网络字节序，表示包体长度
};
// 后面跟 message_length 字节的包体
```

接收方先读固定 4 字节长度，再按长度读包体，就能从字节流中还原一条完整消息。  
注意：长度字段要用网络字节序，并且要校验最大值，防止恶意超大长度导致内存耗尽。

<br>

### 一句话本质
**定长、分隔符、长度字段，是应用层给字节流画边界的三种笔。**

<br>

### 易错点
坑一：**特殊字符要做转义**，否则消息内容里的分隔符会被误当成边界。  
坑二：**长度字段要防超大值攻击**，不能对方说 4GB 你就直接分配 4GB。  
坑三：**HTTP 用头部边界 + `Content-Length`/分块编码解决 body 边界**，不是只靠一个空行就解决整个报文。  
坑四：**长度字段本身也要考虑字节序**，跨平台传输要用网络字节序，别直接塞主机字节序。  
坑五：**接收方要处理“半包”**：包头可能只收到一部分，包体也可能只收到一部分，不能假设一次 `recv` 就收全。

<br>

### 自测
HTTP 头部和正文之间的空行属于哪种分包方式？如果消息内容里本身包含分隔符，应该怎么处理？

<br>

<br>

## 完整可复现示例：用 C++ 实现长度字段分包
***
下面这个 C++ 示例实现“4 字节网络字节序长度 + 包体”的分包与解包，并故意模拟“一次 `recv` 只收到部分数据”的情况，验证接收方必须按长度循环读。  
编译运行：

```bash
g++ -std=c++17 -O2 length_prefix_demo.cpp -o length_prefix_demo
./length_prefix_demo
```

```cpp
// length_prefix_demo.cpp
#include <arpa/inet.h>

#include <cstdint>
#include <cstring>
#include <iostream>
#include <string>
#include <vector>

// 打包：4 字节网络字节序长度 + 包体
std::vector<uint8_t> pack_message(const std::string& body) {
    uint32_t len = static_cast<uint32_t>(body.size());
    uint32_t net_len = htonl(len);

    std::vector<uint8_t> packet(sizeof(net_len) + body.size());
    std::memcpy(packet.data(), &net_len, sizeof(net_len));
    std::memcpy(packet.data() + sizeof(net_len), body.data(), body.size());
    return packet;
}

// 模拟接收缓冲区：可能只收到部分数据
class ByteStreamReader {
public:
    explicit ByteStreamReader(std::vector<uint8_t> data)
        : data_(std::move(data)), pos_(0) {}

    // 尝试从字节流中解析一条完整消息
    // 返回 true 表示解析成功；false 表示数据还不够
    bool try_parse_one(std::string& out_body, uint32_t max_len = 1024 * 1024) {
        // 1. 包头都不够
        if (data_.size() - pos_ < sizeof(uint32_t)) {
            return false;
        }

        uint32_t net_len = 0;
        std::memcpy(&net_len, data_.data() + pos_, sizeof(net_len));
        uint32_t len = ntohl(net_len);

        // 2. 防超大长度
        if (len > max_len) {
            std::cerr << "message too large: " << len << "\n";
            return false;
        }

        // 3. 包体还不够
        if (data_.size() - pos_ - sizeof(uint32_t) < len) {
            return false;
        }

        // 4. 取出包体，移动读位置
        const char* body_ptr = reinterpret_cast<const char*>(
            data_.data() + pos_ + sizeof(uint32_t));
        out_body.assign(body_ptr, len);
        pos_ += sizeof(uint32_t) + len;
        return true;
    }

    bool empty() const { return pos_ >= data_.size(); }

private:
    std::vector<uint8_t> data_;
    size_t pos_;
};

int main() {
    // 模拟发送方连续发两条消息
    auto p1 = pack_message("hello");
    auto p2 = pack_message("TCP boundary");

    // 拼成一个连续字节流，模拟 TCP 可能合并到达
    std::vector<uint8_t> stream;
    stream.insert(stream.end(), p1.begin(), p1.end());
    stream.insert(stream.end(), p2.begin(), p2.end());

    // 模拟“一次只给一部分数据”的接收过程
    ByteStreamReader reader(std::move(stream));

    // 这里为了演示，ByteStreamReader 内部已经拿到完整流；
    // 真实网络里应循环 recv，把数据追加到缓冲区，再尝试解析。
    std::string body;
    while (reader.try_parse_one(body)) {
        std::cout << "parsed: [" << body << "]\n";
    }

    if (!reader.empty()) {
        std::cout << "还有不完整数据，等待下一次 recv\n";
    }

    return 0;
}
```

输出/现象/结论：

```text
parsed: [hello]
parsed: [TCP boundary]
```

验证结果：发送方把两条消息拼成一个连续字节流，接收方靠“先读 4 字节长度，再按长度读包体”正确还原两条消息；即使多条消息合在一起到达，也不会混淆边界。  
注意：这个示例把完整流一次性放进 `ByteStreamReader` 演示解析逻辑；真实网络程序要循环 `recv`，把数据追加到缓冲区，再反复调用 `try_parse_one`。另外，长度字段要做最大值校验，防止恶意超大长度；包头和包体都可能“半包”，不能假设一次 `recv` 就收全。

<br>

<br>

## 自测三问
***
1. TCP 面向字节流和 UDP 面向报文在发送端有什么区别？为什么一次 `send()` 不一定对应一次 `recv()`？  
2. 三种常见分包方式是什么？HTTP 头部和正文之间的空行属于哪一种？  
3. 为什么长度字段分包要处理“半包”和“超大长度”？如果接收方只收到 2 字节长度字段，应该怎么办？

<br>

<br>

## 一句话总结全章
***
**TCP 是连续字节流，没有天然消息边界；应用层必须用定长、特殊分隔符或“长度字段+包体”自己划边界，否则就会出现粘包/拆包解析问题。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>