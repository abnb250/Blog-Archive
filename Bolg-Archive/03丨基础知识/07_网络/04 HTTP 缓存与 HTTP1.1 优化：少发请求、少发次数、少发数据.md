# 04 HTTP 缓存与 HTTP/1.1 优化：少发请求、少发次数、少发数据
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
同一个网页反复打开，为什么有时很快？网页有十几张小图，浏览器真的要建十几次连接吗？HTTP/1.1 性能一般，到底怎么优化？  
这一章咱们用三个问题串起来：**能不能不发请求 → 能不能少发请求 → 能不能让响应更小。**  
先别急着背 `Cache-Control`、`ETag`、`gzip` 这些名词，先看一个具体场景：你第二次打开同一个网页，浏览器怎么知道哪些资源可以直接用本地副本？缓存过期了，但服务器资源没变，还要不要重新传一遍？

<br>

<br>

## 知识点 1：强制缓存——没过期就别去服务器
***
### 问题
浏览器怎么判断“直接用本地缓存”？为什么有时候刷新页面很快，有时候又得重新下载？

<br>

### 方案
强制缓存由响应头 `Cache-Control`（相对时间）和 `Expires`（绝对时间）实现；两者同时存在时 **`Cache-Control` 优先级更高**。  
流程是：浏览器第一次请求资源，服务器返回资源并在响应头加 `Cache-Control` 设置过期时间；浏览器再次请求时，先算资源是否过期；没过期直接用本地缓存，过期才重新请求服务器。  
在开发者工具里看到 `from disk cache`，通常说明用了本地缓存；具体是强缓存命中，还是协商缓存返回 304 后复用本地副本，要结合请求头一起看。

<br>

### 最小示例
一个带强制缓存的响应头可能长这样：

```http
HTTP/1.1 200 OK
Cache-Control: max-age=3600
Expires: Wed, 21 Oct 2026 07:28:00 GMT
Content-Type: text/css
```

`Cache-Control: max-age=3600` 表示资源在 3600 秒内可以直接用本地缓存，不必再问服务器；`Expires` 给的是绝对时间，优先级低于 `Cache-Control`。

<br>

### 一句话本质
**强制缓存就是“没过期就别去服务器”。**

<br>

### 易错点
坑一：**`Expires` 依赖客户端时间**，客户端时间不准就会影响判断，不如 `Cache-Control` 精细。  
坑二：**`Cache-Control` 优先级高于 `Expires`**，两者同时存在时先看 `Cache-Control`。  
坑三：强制缓存没命中，才轮到协商缓存；别把 304 当成强缓存命中。

<br>

### 自测
响应头同时有 `Cache-Control: max-age=60` 和 `Expires`，浏览器优先看哪个？

<br>

<br>

## 知识点 2：协商缓存——问服务器“我这份旧的还能用吗”
***
### 问题
缓存过期了，但服务器资源没变，还要传一遍吗？服务器怎么告诉浏览器“你本地那份还能用”？

<br>

### 方案
协商缓存由服务端告知客户端是否可用本地缓存，常见结果是 **304 Not Modified**。两种常见实现：  
- `Last-Modified` + `If-Modified-Since`：基于最后修改时间。  
- `ETag` + `If-None-Match`：基于资源唯一标识，通常更准确。  
两者都有时 **`ETag` 优先级更高**。  
协商缓存字段要配合强制缓存 `Cache-Control` 使用：**只有未命中强制缓存，才发起带协商字段的请求。** 服务器判断资源没变，就返回 304，通常不带 body，浏览器继续用本地副本。

<br>

### 最小示例
第一次响应可能带：

```http
Last-Modified: Mon, 10 Oct 2023 12:00:00 GMT
ETag: "abc123"
```

再次请求时浏览器带上条件字段：

```http
If-Modified-Since: Mon, 10 Oct 2023 12:00:00 GMT
If-None-Match: "abc123"
```

如果资源没变，服务器返回：

```http
HTTP/1.1 304 Not Modified
```

<br>

### 一句话本质
**协商缓存就是“问服务器：我这份旧的还能用吗？”**

<br>

### 易错点
坑一：**文件内容没变但修改时间变了**，`Last-Modified` 会误判，可能让缓存失效。  
坑二：**秒级内多次修改**，时间粒度不够，`ETag` 能解决这类问题。  
坑三：**304 不是错误**，是缓存有效；它通常没有响应 body，但可能更新部分响应头。  
坑四：`ETag` 更准，但服务器计算 `ETag` 也有开销，不是越复杂越好。

<br>

### 自测
为什么 `ETag` 通常比 `Last-Modified` 更准确？

<br>

<br>

## 知识点 3：HTTP/1.1 三类优化
***
### 问题
HTTP/1.1 怎么优化？为什么会有 HTTP/2 和 HTTP/3？只靠合并文件、压缩图片够不够？

<br>

### 方案
三个方向：  
1. **避免发请求**：用强制缓存和协商缓存。  
2. **减少请求次数**：避免不必要重定向、合并请求（CSS Sprites、打包 JS/CSS、Base64 内嵌小图）、延迟/按需获取资源（懒加载）。  
3. **减少响应大小**：无损压缩（去空格、gzip、Brotli）和有损压缩（WebP、音视频编码）。  
HTTP/1.1 还有瓶颈：头部不压缩且重复、响应队头阻塞、没有请求优先级、服务器只能被动响应。这些靠“外部优化”解决不彻底，所以才有 HTTP/2 和 HTTP/3。HTTP/2 做了头部压缩、多路复用、请求优先级等；HTTP/3 基于 QUIC/UDP 继续优化传输层队头阻塞等问题。

<br>

### 最小示例
```text
优化方向          手段                         代价/注意
避免请求          强制缓存、协商缓存            缓存过期要重新验证
减少请求次数      合并小图、打包 JS/CSS          一个小资源变化可能重下大文件
减少响应大小      gzip/Brotli、WebP             有损压缩牺牲质量
```

结论：**缓存少发请求，合并/延迟少发次数，压缩少发数据。**

<br>

### 一句话本质
**缓存少发请求，合并/延迟少发次数，压缩少发数据。**

<br>

### 易错点
坑一：**合并资源后，一个小资源变化可能导致整个大文件重下**，缓存粒度变粗。  
坑二：**HTTP/1.1 的队头阻塞常和管线化一起讨论**；浏览器通常不开管线化，而是用多连接缓解，但连接数有限仍会排队。  
坑三：**压缩不是越多越好**，有损压缩会牺牲质量；Base64 内嵌小图也会增大 HTML/CSS 体积，且不能单独缓存。

<br>

### 自测
为什么“把很多小图合并成大图”能提速，但又可能带来新问题？

<br>

<br>

## 完整可复现示例：用 curl 观察缓存和压缩
***
```bash
set -e

echo "=== 1. 查看响应缓存头 ==="
curl -sI https://example.com/ | grep -i -E 'cache-control|expires|etag|last-modified' || true

echo "=== 2. 发送条件 GET ==="
LAST_MODIFIED=$(curl -sI https://example.com/ | tr -d '\r' | awk -F': ' 'tolower($1)=="last-modified"{print $2; exit}')
if [ -n "$LAST_MODIFIED" ]; then
  echo "Last-Modified: $LAST_MODIFIED"
  curl -s -o /dev/null -w "HTTP Code: %{http_code}\n" -H "If-Modified-Since: $LAST_MODIFIED" https://example.com/
else
  echo "响应里没有 Last-Modified，跳过条件 GET"
fi

echo "=== 3. 查看压缩相关头 ==="
curl -sI -H "Accept-Encoding: gzip, br" https://example.com/ | grep -i -E 'content-encoding|content-length' || true
```

输出/现象/结论可能类似：

```text
=== 1. 缓存头 ===
可能看到 cache-control、etag、last-modified 等

=== 2. 条件 GET ===
Last-Modified: ...
HTTP Code: 304 或 200（取决于资源和服务器）

=== 3. 压缩头 ===
可能看到 content-encoding: gzip/br
```

验证结果：强制缓存看 `Cache-Control/Expires`；协商缓存看 `ETag/Last-Modified` 和 304；压缩看 `Content-Encoding`。这些手段分别对应“少发请求、少发次数、少发数据”。  
注意：`example.com` 的响应头会随站点和网络环境变化；HEAD 请求不一定触发压缩，看不到 `Content-Encoding` 也正常。

<br>

<br>

## 自测三问
***
1. 强制缓存和协商缓存分别在什么时候起作用？  
2. 为什么 `ETag` 的优先级通常高于 `Last-Modified`？  
3. HTTP/1.1 优化有哪三个方向？各举一个手段。

<br>

<br>

## 一句话总结全章
***
**HTTP/1.1 优化就是三句话：缓存避免发请求，合并/延迟减少请求次数，压缩减少响应大小；但这些都是外部优化，HTTP/2 和 HTTP/3 才从协议本身继续解决队头阻塞等问题。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>