# 07 WebSocket、SSE 与轮询：实时通信怎么选
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 场景问题

***

你要给产品加一个“AI 回答逐字显示”的功能。前端同学说：“用 WebSocket 吧，实时。”后端同学说：“WebSocket 要维护长连接、心跳、重连，我们现在的负载均衡还是 HTTP/1.1 的，搞不动。”这时候旁边有人补了一句：“ChatGPT 的流式输出用的就是 SSE，不是 WebSocket。”

你打开 DevTools，Network 里那条请求的 `Content-Type` 是 **`text/event-stream`**，响应体一行一行往外吐 `data: ...`，Connection 一直挂着不结束。**但它看起来又像是一个普通的 HTTP 响应。** 那它和 WebSocket 到底差在哪里？为什么一个“单向”的 SSE 能支撑 AI 产品的核心交互？轮询在这种场景下又错在哪？

这一篇不教你写完整聊天室，而是帮你建立一条**选型决策线**：**先问“谁向谁说话、说多少次”，再问“用什么协议”**。同时你会亲手用 `curl` 看到 SSE 的流式效果，以及浏览器在 HTTP/1.1 下对 SSE 施加的那个容易被忽视的限制。

<br>

<br>

## 核心概念

***

### 先把问题翻译对：四种方案，四种“说话方式”

<br>

**所有实时通信方案本质上都在回答同一个问题：服务器有新数据时，怎么让客户端“知道”** 。HTTP 的默认模型是“客户端不问，服务器不说”[reference:0]。四种主流方案是在这个约束下做的四种妥协：

**短轮询（short polling）** ：客户端每隔固定间隔问一次“有新数据吗”。服务器无论有没有都立刻返回。**实时性由间隔决定，服务器压力由在线人数乘以频率决定**。适合更新频率极低、实时性要求松的场景，比如每天变几次的公告，或者 Serverless 环境——函数跑完就结束，根本没地方挂长连接[reference:1]。

**长轮询（long polling）** ：客户端问一次“有新数据告诉我”，服务器**不立刻返回**，而是把请求挂起，等到有数据或超时才响应。客户端收到响应后立刻发起下一次。它减少了无效请求，但**服务器仍然要维护大量挂起连接**，每个挂起连接都占资源[reference:2]。

**SSE（Server-Sent Events，服务器发送事件）** ：客户端发起一个普通 HTTP 请求，服务器返回 `Content-Type: text/event-stream`，然后**保持响应不结束**，持续往这个响应里写文本事件。它是**单向的**：服务器 → 客户端。客户端不能再这条连接上往回发数据；如果客户端要说话，得另发一个普通 HTTP 请求[reference:3]。浏览器内置了 `EventSource`，**自动重连**是它的默认行为，这一点在后面“动手实验”里会展开。

**WebSocket** ：客户端先发一个 HTTP 请求，带 `Upgrade: websocket` 和 `Connection: Upgrade`，服务器如果支持，返回 **`101 Switching Protocols`**。从这一刻起，**HTTP 就“退出”了这条连接**，后续字节在同一根 TCP 连接上按 WebSocket 成帧协议传输[reference:4]。它是**全双工的**，双方都可以随时发消息。

**一句话的选型直觉**：**只需要服务器单向推 → SSE；需要双向频繁互发 → WebSocket；数据几乎不变或架构没法挂长连接 → 轮询就够。** 这不是绝对规则，但它是面试和项目讨论中第一个应该被说出来的判断。

<br>

### WebSocket 的“升级”到底升了什么

<br>

WebSocket 常被误解成“一种替代 TCP 的东西”。**不是。每一条 WebSocket 连接本身就是一条 TCP 连接**，WebSocket 只是在 TCP 之上加了一层**成帧（framing）** 协议，让你可以发“消息”而不是裸字节流[reference:6]。TCP 没有消息边界：你发两个 100 字节，对方可能收到一个 200 字节的块。WebSocket 的每一帧自带边界，**一次 `send()` 在对面产生一条完整的消息事件**[reference:7]。

关键细节在**握手只发生一次**。`101 Switching Protocols` 之后，HTTP 头部那套东西（Cookie、Authorization、Content-Type）不会再伴随每条消息出现。一个 HTTP 请求头通常 400–800 字节，而 WebSocket 帧头对小消息只要 **2 字节**，加上客户端掩码也才 6–14 字节[reference:8]。**高频双向场景下，这个开销差异是 WebSocket 存在的核心理由之一。**

**边界说明**：HTTP/2 **明确禁止**使用 `Upgrade` 机制升级到 WebSocket[reference:9]。在 HTTP/2 上引导 WebSocket 需要额外机制（如 Extended CONNECT），不在本篇展开，**具体行为以当前主流实现和 RFC 8441 为准，需查证**。

<br>

### SSE 的两个“隐藏成本”：连接数限制与断线重连语义

<br>

SSE 看起来比 WebSocket 简单，但有两个机制如果不知道，会在项目里以很奇怪的方式暴露出来。

**第一，HTTP/1.1 下每个浏览器 + 域名最多大约 6 条并发连接，SSE 占其中一条且不释放**。MDN 明确指出，不使用 HTTP/2 时，SSE 受到打开连接数上限限制，这个限制是**针对浏览器 + 域名**的，在 Chrome 和 Firefox 中数值很低（6），并且该问题被标记为“不会修复”[reference:10]。关键点在于：**这 6 条连接是跨标签页共享的**，不是每个标签页 6 条。如果你在一个页面里开了 6 条 SSE，**同一个域名下的普通 `fetch()` 请求会排队**，表现成“接口莫名其妙卡住”。HTTP/2 下这个限制消失，并发流数量由协商决定，默认通常为 100[reference:13]。

**第二，SSE 的自动重连不是“重连就完事”** 。`EventSource` 在连接断开后会自动尝试重连。如果服务器在事件里写了 `id:` 字段，浏览器重连时会在请求头里带上 **`Last-Event-ID`**，服务器据此可以从断点之后继续推送[reference:14][reference:15]。**但“带上 ID”不等于“服务器一定支持断点续传”** 。如果你的服务端没有根据 `Last-Event-ID` 做重放，断线期间的事件就丢了。SSE 给你的是**重连机制**，**不自动给你“不丢消息”的保证**。

<br>

<br>

## 动手实验：用 `curl -N` 看 SSE 流式，再用 DevTools 撞一次 6 连接限制

***

### 实验一：`curl -N` 观察 SSE 的“流式响应”

<br>

**目标**：不写前端代码，只用命令行看到 SSE 的响应体是一行一行持续到达的，而不是像普通 HTTP 那样等整个响应结束才输出。

在终端执行（`-N` 关闭 curl 的输出缓冲，让数据到达就显示）：

```bash
curl -N -s https://example.com/sse-demo
```

> **预期现象（示意，具体事件内容取决于你实际可用的 SSE 端点）：**
>
> ```text
> data: first event
>
> data: second event
>
> : heartbeat
>
> data: third event
> ```
>
> 命令**不会自己结束**，光标停在最后一行等待新数据。

**看到什么说明什么**：普通 HTTP 响应会等 `Content-Length` 满足或连接关闭才结束；SSE 的响应**没有“结束”的语义**（除非服务器主动关闭或客户端 `close()`）。`data:` 行之后跟着一个空行，才构成一个完整事件。`: heartbeat` 是以冒号开头的**注释行**，用来保活，浏览器会忽略它，但代理和负载均衡器看到的是“连接上还在有字节流动”。

**如果你访问 `https://example.com` 没有现成的 SSE 端点**，可以用一个最小的本地服务模拟（Python 示例，仅用于观察流式行为）：

```python
# 保存为 sse_demo.py，运行：python3 sse_demo.py
from http.server import BaseHTTPRequestHandler, HTTPServer
import time

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "text/event-stream")
        self.send_header("Cache-Control", "no-cache")
        self.end_headers()
        for i in range(5):
            self.wfile.write(f"data: event-{i}\n\n".encode())
            self.wfile.flush()
            time.sleep(1)
        # 不主动关闭，模拟长连接

HTTPServer(("127.0.0.1", 8000), Handler).serve_forever()
```

然后在另一个终端执行：

```bash
curl -N -s http://127.0.0.1:8000/
```

> **预期现象**：每秒多出一行 `data: event-0` 到 `data: event-4`，而不是 5 秒后一次性全部出现。

**看到这个现象说明哪个机制在起作用**：**SSE 的“流式”不是客户端在轮询，而是服务器在一个 HTTP 响应体内持续写入，客户端按行解析 `text/event-stream` 格式。** 这正是 LLM token 流式输出选择 SSE 的原因：它天然适配“服务器持续产出、客户端持续消费”的单向模式。

<br>

### 实验二：在 DevTools 里“撞”一次 SSE 连接数限制

<br>

**目标**：亲眼看到 HTTP/1.1 下 SSE 占满浏览器连接池后，普通 `fetch` 被排队的效果。

**前提**：你有一个可以建立多条 SSE 连接的测试端点（可以用上面的 Python 脚本稍作改造，让每条连接持续更久）。**本实验在 HTTP/1.1 下观察**；如果你访问的站点是 HTTP/2，限制不会以同样方式出现。

**步骤**：在同一个页面的 Console 里执行（URL 替换为你自己的 SSE 端点）：

```javascript
// 建立 6 条 SSE 连接，占满 HTTP/1.1 的每域名连接池
for (let i = 0; i < 6; i++) {
  const es = new EventSource("/sse-demo?conn=" + i);
}
// 然后立刻发起一个普通 fetch
fetch("/api/ping").then(r => console.log("fetch resolved", r.status));
```

> **预期现象**：`fetch("/api/ping")` **不会立刻 resolve**，在 DevTools Network 里可以看到它处于 `Stalled` 或 `Pending`，直到某条 SSE 连接断开、让出连接池槽位。

**看到这个现象说明什么**：**SSE 在 HTTP/1.1 下不是“免费的”** 。它占用的是浏览器对同一个 origin 的并发连接配额。**如果页面里有多个组件各自开了 SSE，普通接口请求会被挤到排队**。这是生产环境中一个真实存在的坑：前端用了 SSE，后端接口却“偶尔卡住”，排查方向容易跑偏到服务端。

**常见失败原因与解决办法**：

| 现象 | 可能原因 | 怎么办 |
|---|---|---|
| `fetch` 一直 pending | SSE 占满了 6 条 HTTP/1.1 连接 | 升级到 HTTP/2；或把 SSE 集中到一个连接上，由前端分发事件（multiplexing） |
| `curl -N` 一次性输出而不是逐行 | 中间有代理/缓冲层（如 Nginx）缓冲了响应 | 服务端设置 `X-Accel-Buffering: no`，并确保 `Content-Type: text/event-stream`；以实际代理配置为准 |
| `EventSource` 重连后收不到断点前的事件 | 服务端没有根据 `Last-Event-ID` 做重放 | 服务端实现事件 ID 与断点重放，不能只依赖浏览器自动重连 |

<br>

<br>

## 开发启示

***

### 启示一：选型先看“交互方向”，不要先看“哪个听起来高级”

<br>

**现象**：项目要做“服务器推送通知”，团队直接上了 WebSocket，结果负载均衡需要 sticky session，运维成本陡增；或者要做“AI 逐字输出”，用了 WebSocket，却发现客户端只需要收、不需要在这条连接上发，白白背上了双向连接管理的复杂度[reference:18]。

**底层原因**：**SSE 和 WebSocket 解决的是不同形状的问题**。SSE 是单向服务器推送，跑在普通 HTTP 上，负载均衡器把它当普通 HTTP 响应处理，不需要粘性会话；自动重连是浏览器内置的[reference:19]。WebSocket 是全双工，适合聊天、协作编辑、游戏这类双方频繁互发的场景，但代价是你**自己负责协议生命周期**：心跳、重连、粘性会话、连接状态管理。

**可操作动作 / 面试答法**：先画一张表：**“谁发起、谁接收、频率多高、数据是文本还是二进制”** 。只推文本且客户端不需要在同连接上说话 → 优先 SSE；双向高频 → WebSocket；更新频率极低或部署环境无法挂长连接 → 轮询。面试时不要只背“SSE 单向、WebSocket 双向”，要补一句“所以 AI 流式输出用 SSE，负载均衡友好；聊天用 WebSocket，因为需要客户端随时打断或回发”。

<br>

### 启示二：SSE 不是“永不丢失”的推送，`Last-Event-ID` 需要服务端兑现

<br>

**现象**：SSE 连接因为网络抖动断了，浏览器自动重连，但客户端发现中间少了几条事件；日志里看不出任何错误。

**底层原因**：`EventSource` 的自动重连只负责**重建连接**。如果服务器在事件中写了 `id:`，浏览器重连时会带上 `Last-Event-ID` 请求头，**把“我从哪里断的”告诉服务器**[reference:22]。但**服务器必须自己实现“根据这个 ID 把断点之后的事件补发出去”** 。没有服务端重放逻辑，`Last-Event-ID` 只是一个被忽略的请求头。

**可操作动作 / 面试答法**：SSE 事件格式中带 `id:` 字段；服务端维护一个可重放的事件缓冲（内存或持久化），根据 `Last-Event-ID` 决定从哪个序号开始补发。面试被问“SSE 断线重连会不会丢消息”，正确答案是“**浏览器给你重连和 `Last-Event-ID`，丢不丢取决于服务端有没有实现断点续传**”[reference:23]。

<br>

### 启示三：面试陷阱——“WebSocket 比 HTTP 快，所以实时场景都用它”

<br>

**常见误解**：WebSocket 建连之后没有 HTTP 头部开销，所以它“比 HTTP 快”，凡是实时场景都应该用 WebSocket。

**正确思路**：WebSocket **不可能比 TCP 快**，因为它的每一个字节都仍然走 TCP 栈[reference:24]。它减少的是 **HTTP 每请求重复头部**的开销，这对**高频小消息**有意义；但对于**低频推送**，SSE 的简单性和 HTTP 基础设施兼容性往往更划算。而且 SSE 在 HTTP/2 下**连接数限制消失**，多路复用让一个连接承载多个流，单向推送的性价比更高[reference:25]。**把“WebSocket 低开销”直接等同于“所有实时场景的最优解”，是选型题里最常见的过度概括。**

**面试答法**：先承认 WebSocket 在全双工和高频小消息上的优势，再补一句“**选型看形状**：单向文本流用 SSE，双向高频用 WebSocket，极低频用轮询；WebSocket 的快是相对 HTTP 头部开销而言，不是相对 TCP”[reference:26]。

<br>

<br>

## 小结与下一篇预告

***

回到开头：**AI 逐字输出为什么用 SSE 而不是 WebSocket？** 因为它的交互形状是“服务器持续产出 token，客户端持续消费”，**单向、文本、HTTP 友好**。SSE 让负载均衡器把它当普通 HTTP 响应处理，自动重连由浏览器兜底，工程复杂度远低于维护一条全双工 WebSocket。**但 SSE 不是银弹**：HTTP/1.1 下它有连接池配额的问题，断点续传需要服务端自己实现 `Last-Event-ID` 重放。**选型的第一步永远是把交互翻译成“方向 + 频率 + 数据形态”，再挑协议。**

下一篇进入运输层：**TCP vs UDP：开发者到底该怎么选**。你会从 Socket 接口的差异出发，理解“可靠”和“实时”在协议层面分别意味着什么，以及为什么 HTTP/3 要跑在 QUIC（基于 UDP）之上。

自测问题：

1. SSE 和 WebSocket 在“通信方向”上的根本差异是什么？各自最适合举一个开发场景。
2. HTTP/1.1 下 SSE 的 6 连接限制是谁限制的、限制范围是什么？对普通 `fetch` 有什么影响？

<br>

<details>
<summary>点击查看自测问题答案</summary>

**第 1 题答案**

- **根本差异**：SSE 是**单向**的，数据只能从服务器流向客户端，客户端不能在同一条 SSE 连接上往回发数据（要说话得另发 HTTP 请求）[reference:27]。WebSocket 是**全双工**的，客户端和服务器可以在同一条持久连接上随时互发消息[reference:28]。
- **场景举例**：SSE 适合 **AI 回答逐字输出 / 服务器日志流 / 通知推送**；WebSocket 适合 **聊天室 / 实时协作编辑 / 在线游戏**这类双方频繁互发的场景[reference:29]。

**第 2 题答案**

- **谁限制的**：**浏览器**，不是服务器。Chrome 和 Firefox 对 HTTP/1.1 下每个 **浏览器 + 域名** 的并发连接数限制在约 6 条，SSE 占其中一条且不释放[reference:30]。
- **限制范围**：**跨标签页共享**。同一个浏览器里打开多个标签页，对同一个域名的 SSE 连接总数仍然受这个配额约束，不是每个标签页各 6 条。
- **对 `fetch` 的影响**：如果 SSE 占满了 6 条，同一个域名下的普通 `fetch()` 请求会**排队等待**连接池出现空位，表现成“接口卡住”。HTTP/2 下这个限制消失，并发流数量由协商决定（默认通常 100）[reference:34]。

</details>


<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>


