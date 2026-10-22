# 05 HTTPS 与 TLS：混合加密、证书、RSA/ECDHE 握手与优化
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
你在咖啡店连公共 Wi-Fi，打开网银登录。你可能会想：**HTTP 明文传输，中间人是不是能看到我的密码？**

如果网站是 HTTPS，地址栏有小锁，你稍微安心一点。但再想一步：**抓包工具为什么又能看到 HTTPS 数据？是不是 HTTPS 也不安全？**

还有：RSA 和 ECDHE 握手差在哪？为什么现代更推荐 ECDHE？证书到底在证明什么？

这一章咱们用一条问题链往前走：

1. HTTP 明文有什么风险，HTTPS 怎么补？
2. 为什么不全用非对称加密，混合加密到底混了什么？
3. 服务器说“这是我的公钥”，客户端凭什么信？证书和证书链在干什么？
4. RSA 和 ECDHE 握手差在哪，前向安全是什么？
5. HTTPS 有性能损耗，怎么优化，代价是什么？

<br>

<br>

## 优先级说明
***
这篇是 HTTPS 与 TLS 的核心，面试和项目都会反复用到。先分清主次：

**必须掌握（面试常问、项目真用得上）：**
- HTTPS 解决窃听、篡改、冒充，分别靠混合加密、摘要+签名、证书
- 混合加密：非对称解决“密钥怎么安全给”，对称解决“正文怎么快”
- 数字签名不是加密正文，通常是对摘要签名；“私钥签名、公钥验签”用于身份和完整性
- 证书把公钥和身份绑在一起，客户端沿证书链验到信任的根
- RSA 不支持前向安全，ECDHE 支持前向安全
- ECDHE 的共享 x 不是最终会话密钥，还要混合两个随机数派生
- 会话复用会削弱前向安全，0-RTT 有重放风险

**了解即可（知道大概，不展开）：**
- TLS 1.2 和 TLS 1.3 的完整握手报文细节
- OCSP Stapling 的具体协议流程
- 证书链中中间证书的部署细节
- 具体密码套件的优先级配置

**主线**：HTTP 有窃听/篡改/冒充风险 → HTTPS 加 TLS 补上 → 混合加密保机密、摘要+签名保完整、证书防冒充 → 握手协商会话密钥（RSA vs ECDHE）→ 优化少握手、快计算、小证书、少查 CA。

<br>

<br>

## 知识点 1：HTTPS 不是新协议，是 HTTP 套了一层 TLS
***
### 问题

HTTPS 比 HTTP 多了什么？为什么说它解决窃听、篡改、冒充？加了 TLS 就“绝对安全”了吗？

**你可能会想**：HTTPS 是不是一个新协议，和 HTTP 完全不一样？

不是。HTTPS 还是 HTTP，只是把 HTTP 报文装进了一个“加锁信封”里，这个信封就是 TLS。

<br>

### 核心概念与方案

先把 HTTPS 换成你熟悉的场景：**HTTP 像寄明信片，路上谁都能看、能改、能冒充寄件人；HTTPS 像寄加锁信封，还附了一张身份证明。**

HTTPS 在 TCP 和 HTTP 之间加入 TLS：

- **混合加密**保证机密性，解决窃听。
- **摘要算法 + 数字签名**保证完整性，解决篡改。
- **数字证书**解决冒充。

HTTP 默认 80，HTTPS 默认 443；HTTPS 在 TCP 三次握手后还要 TLS 握手；需要 CA 证书。

所以 HTTPS 不是新造了一个应用层协议，而是 HTTP 套了一层 TLS，把“明文明信片”变成了“加锁信封 + 验身份”。

<br>

### 最小实验：只看 TLS 握手有没有发生

环境前提：Linux/macOS，或 Windows WSL；已安装 `openssl`，需要外网。下面只观察 TLS 握手，不看证书细节。

````bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | grep -E 'Protocol|Cipher'
````

**这次只看一件事**：输出里有没有 `Protocol` 和 `Cipher`。

可能看到类似：

````text
Protocol: TLSv1.3
Cipher: TLS_AES_256_GCM_SHA384
````

有 `Protocol` 和 `Cipher`，说明 TLS 握手成功了，双方协商出了协议版本和密码套件。如果连接失败，先检查网络和 `openssl` 是否可用。

注意：不同站点、不同 `openssl` 版本下协议版本和套件可能不同，**需按环境核实**。HTTP 请求本身没有这层协商，这就是 HTTPS 和 HTTP 最直观的差别之一。

<br>

### 易错点与边界

坑一：**HTTPS 不是“绝对安全”**，中间人攻击成功往往是因为用户接受了非法证书，或系统被导入了恶意根证书。

坑二：**TLS 和 SSL 可以视作同一个东西的不同阶段**，现在常说 SSL/TLS；说“SSL 证书”通常就是指 TLS 证书。

坑三：**HTTPS 默认 443，但端口可以改**，排查时不能只看 443。

坑四：**HTTPS 保护的是传输过程**，不保护服务器本身被攻破、也不保护应用层漏洞。

<br>

### 一句话本质

**HTTPS = HTTP + TLS，给数据加锁、验指纹、验身份；它保护传输，不是“绝对安全”。**

<br>

### 自测

HTTPS 解决的是 HTTP 的哪三个安全问题？分别靠什么机制？

<details><summary>答案</summary>窃听靠混合加密保证机密性；篡改靠摘要算法 + 数字签名保证完整性；冒充靠数字证书证明身份。</details>

<br>

<br>

## 知识点 2：混合加密、摘要与数字签名
***
### 问题

为什么不全用非对称加密？非对称加密不是更安全吗？数字签名到底在签什么，它和“加密正文”是一回事吗？

**你可能会想**：非对称加密有公钥私钥，看起来更安全，那全部用它不就行了？

不行，非对称加密慢。HTTPS 只用它来安全地协商会话密钥，真正传正文还是用对称加密。这就是“混合”的意思。

<br>

### 核心概念与方案

先把混合加密换成你熟悉的场景：**非对称像保险箱，安全但开关麻烦；对称像普通锁，快但钥匙得先安全送到对方手里。**

HTTPS 的做法是：**先用非对称手段安全地把会话密钥协商出来，后续用对称加密传数据。**

这里要区分两种“非对称手段”：

- **RSA 密钥交换**：客户端用服务器公钥加密 pre-master 发给服务器，属于“用非对称加密传密钥”。
- **ECDHE 密钥协商**：双方交换临时公钥，各自算出共享秘密，属于“协商”而不是“加密传输”。

两者都服务于同一个目的：**让双方安全地得到同一把会话密钥。**

摘要算法生成内容“指纹”；数字签名用私钥对摘要做签名运算，公钥验签，证明消息来源且未被篡改。

注意：**“私钥签名、公钥验签”主要用于身份和完整性，不是用来加密大量正文。** 别和“公钥加密、私钥解密”的保密用法混为一谈。

<br>

### 最小实验：只看摘要是否一致

环境前提：Linux/macOS，或 Windows WSL；已安装 `sha256sum`（Linux）或 `shasum`（macOS）。下面只观察摘要变化。

````bash
printf 'hello' | sha256sum
printf 'hello!' | sha256sum
````

**这次只看一件事**：两次输出的摘要是不是完全不同。

预期输出类似：

````text
2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824  -
d6f1b0e5e5a7e2c4d4b0e4c3a0b8e6f2c8d9a1b2c3d4e5f6a7b8c9d0e1f2a3b4  -
````

只改了一个字符，摘要就完全不同。这就是摘要的“雪崩效应”：内容一变，指纹就变。

**但摘要只能证明内容有没有变，不能证明是谁发的。** 要证明来源，还得靠数字签名，用私钥对摘要签名，公钥验签。

<br>

### 易错点与边界

坑一：**哈希能防篡改，但不能单独证明来源**，还要数字签名。

坑二：**数字签名不是加密正文**，通常是对摘要签名。

坑三：**公钥加密、私钥解密**用于保密；**私钥签名、公钥验签**用于身份和完整性，目的不同。

坑四：**“非对称交换会话密钥”在 RSA 下成立，在 ECDHE 下更准确说是“协商”**，别把两者混成同一个动作。

<br>

### 一句话本质

**非对称解决“密钥怎么安全给”，对称解决“正文怎么快”，摘要+签名解决“内容有没有被改、是不是你发的”。**

<br>

### 自测

为什么 HTTPS 不直接用非对称加密传所有网页数据？

<details><summary>答案</summary>非对称加密计算开销大、速度慢，不适合传大量正文；HTTPS 只用非对称手段安全地协商会话密钥，后续用对称加密传数据，兼顾安全和性能。</details>

<br>

<br>

## 知识点 3：数字证书与证书链
***
### 问题

公钥被伪造怎么办？服务器说“这是我的公钥”，客户端凭什么信？为什么需要 CA 和证书链？

**你可能会想**：服务器把公钥发过来，客户端直接用不就行了？

不行。中间人可以把自己的公钥冒充成服务器的公钥。客户端需要一个可信第三方来证明“这个公钥确实属于这个服务器”，这就是 CA 和证书。

<br>

### 核心概念与方案

先把证书换成你熟悉的场景：**证书就像 CA 给公钥开的“身份证明”，上面写着“这个公钥属于这个域名”，并盖了 CA 的章。**

CA 对“持有者信息 + 公钥 + 用途 + 有效期”等做 Hash，再用 CA 私钥对 Hash 做签名，打包成数字证书。客户端用系统/浏览器内置的 CA 公钥验签。

实际常是**根证书 → 中间证书 → 服务器证书**的信任链；根证书严格隔离，避免根失守。

如果中间人伪造证书，浏览器会提示证书有问题；如果用户执意继续，或系统被导入恶意根证书，才可能被中间人解密。

<br>

### 最小实验：只看证书的 subject 和 issuer

环境前提：同上，需要外网和 `openssl`。下面只观察证书的持有者和签发者。

````bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
````

**这次只看一件事**：输出里有没有 `subject`、`issuer` 和有效期。

可能看到类似：

````text
subject=CN = example.com
issuer=C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
notBefore=...
notAfter=...
````

`subject` 是证书持有者，`issuer` 是签发者。issuer 不是根证书，说明中间还有中间证书，这就是证书链的一环。

不同站点的证书信息不同，**需按环境核实**；看不到输出时先检查网络和 `openssl` 是否可用。

<br>

### 易错点与边界

坑一：**证书验证不只看单张证书**，还要走信任链，检查有效期、域名、用途等。

坑二：**根证书不能随便导入**，导入恶意根证书等于把家门钥匙给别人。

坑三：**抓包工具能看 HTTPS，是因为它让你信任了它的根证书**，不是 HTTPS 本身有漏洞。

坑四：**证书过期、域名不匹配、证书链不完整**都会导致验证失败，排查时先看浏览器提示的具体原因。

<br>

### 一句话本质

**证书就是 CA 给公钥开的“身份证明”，证书链就是一层信任一层；抓包工具能看 HTTPS，是因为你信任了它的根证书。**

<br>

### 自测

为什么说“抓包工具能看 HTTPS”不等于 HTTPS 不安全？

<details><summary>答案</summary>抓包工具能看 HTTPS，是因为它让你在系统或浏览器里信任了它自己的根证书，相当于你主动把中间人当成了可信方。正常 HTTPS 下，客户端会验证证书链，不会信任未知根证书；所以这不是 HTTPS 本身的漏洞。</details>

<br>

<br>

## 知识点 4：RSA 与 ECDHE 握手
***
### 问题

TLS 握手到底交互什么？RSA 和 ECDHE 差在哪？为什么现代更推荐 ECDHE？前向安全又是什么？

**你可能会想**：RSA 和 ECDHE 不都是协商密钥吗，能差多少？

差别在“密钥材料怎么过去”和“私钥泄漏后过去的数据还安不安全”。RSA 把密钥材料加密传过去，私钥泄漏可能影响过去；ECDHE 是双方临时协商，私钥泄漏不影响过去的会话。

<br>

### 核心概念与方案

先把两种握手换成你熟悉的场景：

- **RSA 像你把钥匙放进保险箱寄过去**，对方用保险箱钥匙打开。保险箱钥匙（服务器私钥）一旦泄漏，过去寄过的保险箱都可能被打开。
- **ECDHE 像双方各自临时造一个秘密，再交换一些公开材料，各自算出同一个共享秘密**。临时秘密用完就丢，服务器长期私钥泄漏也不影响过去的会话。

RSA 握手大致：ClientHello（TLS 版本、密码套件、Client Random）→ ServerHello（Server Random、密码套件、证书）→ 客户端验证证书，用服务器 RSA 公钥加密 pre-master 发送 → 服务端解密，双方用三个随机数生成会话密钥。

ECDHE 握手：双方交换椭圆曲线公钥，各自用私钥和对方公钥算出共享点，取 x 坐标作为共享秘密，再结合 Client Random、Server Random 生成会话密钥；支持**前向安全**，且可 False Start，把 TLS 往返从 2 RTT 降到约 1 RTT。

注意：**ECDHE 中 x 不是最终会话密钥**，还要混合两个随机数走 PRF/HKDF。

<br>

### 最小实验：只看密钥交换算法是不是 ECDHE

环境前提：同上，需要外网和 `openssl`。下面只观察密钥交换算法。

````bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | grep -E 'Server Temp Key|Peer signature type'
````

**这次只看一件事**：输出里有没有 `ECDHE` 或 `X25519` 相关的临时密钥。

可能看到类似：

````text
Server Temp Key: X25519, 253 bits
Peer signature type: RSA-PSS
````

`Server Temp Key` 是临时密钥交换用的曲线，`X25519` 属于 ECDHE 家族。有它，说明这次握手用了 ECDHE 类密钥交换，支持前向安全。

不同站点、不同 `openssl` 版本输出可能不同，**需按环境核实**；看不到 `Server Temp Key` 时，可能是 TLS 1.3 下输出格式不同，不代表没用 ECDHE。

<br>

### 易错点与边界

坑一：**ECDHE 的 x 不是最终会话密钥**，它还要和两个随机数一起派生。

坑二：**TLS 1.2 完整握手常见 2 RTT，TLS 1.3 简化到 1 RTT**；False Start 是在 TLS 1.2 上省往返的技巧，不是所有场景都开。

坑三：**RSA 不是“完全不安全”**，而是不支持前向安全；现代更常用 ECDHE，RSA 更多用于签名和证书。

坑四：**前向安全保护的是过去的会话**，不保护当前会话被实时攻破的情况。

<br>

### 一句话本质

**RSA 靠公钥传密钥但怕私钥泄漏；ECDHE 靠临时密钥协商，支持前向安全，服务器长期私钥泄漏也不影响过去的会话。**

<br>

### 自测

为什么 ECDHE 支持前向安全，而 RSA 不支持？

<details><summary>答案</summary>ECDHE 每次握手使用临时密钥对，共享秘密由双方临时私钥和对方公钥算出，会话结束后临时私钥可丢弃；即使服务器长期私钥 later 泄漏，也无法恢复过去的会话密钥。RSA 密钥交换把 pre-master 用服务器长期公钥加密传输，服务器长期私钥泄漏后，过去截获的密文可能被解密，所以不支持前向安全。</details>

<br>

<br>

## 知识点 5：HTTPS 优化——少握手、快计算、小证书、少查 CA
***
### 问题

HTTPS 有性能损耗，怎么优化？会话复用和 0-RTT 是不是越多越好？

**你可能会想**：会话复用和 0-RTT 能省握手，那全都打开不就行了？

不行。会话复用会削弱前向安全，0-RTT 有重放攻击风险。优化要权衡安全和性能。

<br>

### 核心概念与方案

先把 HTTPS 优化换成四个你熟悉的动作：

- **少握手**：会话复用（Session ID、Session Ticket、TLS1.3 PSK）、TLS 1.3、False Start。
- **快计算**：选支持 AES-NI 的 CPU、升级 OpenSSL、选 ECDHE + x25519、ECDSA 证书。
- **小证书**：ECDSA 证书比 RSA 证书小，减少传输和验证开销。
- **少查 CA**：OCSP Stapling，服务器把 OCSP 响应随证书一起发给客户端，减少客户端单独查 CA 的延迟。

会话复用会削弱前向安全；0-RTT 还有重放攻击风险，要对会话密钥设合理过期时间，敏感请求慎用。

<br>

### 最小案例：用配置片段看清优化点

````nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ecdh_curve x25519;

# TLS 1.2 及以下套件
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384';

# TLS 1.3 套件：nginx 的 ssl_ciphers 不直接管 TLS 1.3，
# 需要 OpenSSL 1.1.1+ 和 nginx 1.19.4+，用 ssl_conf_command 设置
ssl_conf_command Ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384;

ssl_stapling on;
````

**这次只看一件事**：配置里有没有 `TLSv1.3`、`x25519`、`ssl_stapling on`。

有这三项，说明开启了较新的协议、较快的曲线和 OCSP Stapling。具体指令和可用性取决于 nginx 和 OpenSSL 版本，**需按环境核实**；TLS 1.3 套件不能靠传统 `ssl_ciphers` 配。

<br>

### 易错点与边界

坑一：**会话复用会削弱前向安全**，密钥泄漏可能影响复用会话。

坑二：**0-RTT 不是所有请求都能随便用**，它有重放风险，敏感操作要谨慎。

坑三：**不是所有客户端都支持最新算法**，要考虑兼容。

坑四：**nginx 的 `ssl_ciphers` 不直接控制 TLS 1.3 套件**，TLS 1.3 要用 `ssl_conf_command Ciphersuites` 或对应 OpenSSL 配置。

<br>

### 一句话本质

**HTTPS 优化就是少握手、快计算、小证书、少查 CA；但会话复用和 0-RTT 要权衡前向安全与重放风险。**

<br>

### 自测

为什么 OCSP Stapling 能减少证书验证延迟？

<details><summary>答案</summary>OCSP Stapling 让服务器把 CA 的 OCSP 响应随证书一起发给客户端，客户端不用再单独向 CA 查询证书状态，减少一次网络往返和查询延迟。</details>

<br>

<br>

## 完整可复现示例：用 openssl 看证书、TLS 版本和密钥交换
***
你需要 Linux/macOS 或 WSL，已安装 `openssl`，需要外网。下面把三个观察点串起来，每个步骤仍只看一件事。

````bash
set -e

echo "=== 1. 证书 subject/issuer/有效期 ==="
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

echo "=== 2. TLS 版本和密码套件 ==="
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | grep -E 'Protocol|Cipher'

echo "=== 3. 临时密钥交换算法 ==="
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | grep -E 'Server Temp Key|Peer signature type'
````

可能的输出/现象：

````text
=== 1. 证书 ===
subject=CN = example.com
issuer=C = US, O = DigiCert Inc, CN = ...
notBefore=...
notAfter=...

=== 2. TLS 版本和套件 ===
Protocol: TLSv1.3
Cipher: TLS_AES_256_GCM_SHA384

=== 3. 临时密钥交换算法 ===
Server Temp Key: X25519, 253 bits
Peer signature type: RSA-PSS
````

**关键观察点**：第 1 步看 subject 和 issuer 是不是不同；第 2 步看协议版本和套件；第 3 步看有没有 `X25519` 或 `ECDHE`。

验证结果：证书包含持有者、签发者、有效期；TLS 版本和密码套件决定握手和加密方式；临时密钥算法说明是否用了 ECDHE 类密钥交换。

注意：不同站点、不同 OpenSSL 版本输出格式可能不同；`example.com` 的证书和协议版本会随站点配置变化；TLS 1.3 下部分输出可能和 TLS 1.2 不同，**需按环境核实**。

<br>

<br>

## 排障速查表
***
| 现象 | 先查什么 | 常用命令/观察点 | 解释 |
|---|---|---|---|
| 浏览器提示证书无效 | 证书有效期、域名、证书链 | `openssl s_client -connect 域名:443 -servername 域名` 看 `subject`、`issuer`、`notAfter` | 过期、域名不匹配、链不完整都会导致无效；先看浏览器具体提示 |
| 浏览器提示“您的连接不是私密连接” | 证书链、系统时间、中间人 | 看证书详情、系统时间、是否装了未知根证书 | 可能是自签名证书、链不完整、时间不对，或代理/抓包工具根证书 |
| `openssl s_client` 连接失败 | 网络、端口、SNI | `-connect 域名:443 -servername 域名` | 不带 `-servername` 时，共享 IP 的服务器可能返回默认证书或握手失败 |
| 抓包工具能看 HTTPS | 是否信任了抓包工具根证书 | 看系统/浏览器受信任根证书列表 | 抓包工具让你信任它的根证书，不是 HTTPS 本身被攻破 |
| 握手慢 | 协议版本、密钥交换、证书大小 | 看 `Protocol`、`Server Temp Key`、证书链长度 | TLS 1.2 + RSA 可能比 TLS 1.3 + ECDHE 多往返；证书链太长也增加验证开销 |
| 会话复用没生效 | 服务器配置、客户端支持 | 看服务器 `Session Ticket`、TLS 1.3 PSK 配置 | 复用能省握手，但会削弱前向安全；配置和客户端支持都会影响 |
| 0-RTT 被重放 | 是否对敏感请求开了 0-RTT | 查服务器 TLS 1.3 0-RTT 配置和业务日志 | 0-RTT 有重放风险，敏感操作应禁用或加防重放机制 |

<br>

<br>

## 常见误区
***
- **以为 HTTPS 绝对安全**：中间人成功往往是因为用户接受了非法证书或系统被导入恶意根证书。
- **把数字签名当成加密正文**：签名通常是对摘要签名，用于身份和完整性，不是保密。
- **把“公钥加密、私钥解密”和“私钥签名、公钥验签”混为一谈**：前者保密，后者身份和完整性。
- **以为 RSA 完全不安全**：RSA 不是不安全，而是不支持前向安全；现代更多用 ECDHE。
- **以为 ECDHE 的 x 就是会话密钥**：x 还要和两个随机数一起派生会话密钥。
- **以为抓包工具能看 HTTPS 说明 HTTPS 有漏洞**：是因为你信任了抓包工具的根证书。
- **以为会话复用和 0-RTT 越多越好**：会话复用削弱前向安全，0-RTT 有重放风险。
- **以为 `ssl_ciphers` 能配 TLS 1.3 套件**：TLS 1.3 套件要用 `ssl_conf_command Ciphersuites` 或对应 OpenSSL 配置。

<br>

<br>

## 自测三问
***
1. HTTPS 解决窃听、篡改、冒充分别靠什么机制？为什么说它不是“绝对安全”？
2. 为什么 ECDHE 支持前向安全，而 RSA 不支持？ECDHE 的共享 x 是最终会话密钥吗？
3. OCSP Stapling 和会话复用分别优化了什么？会话复用又带来了什么代价？

<details><summary>参考答案</summary>1. 窃听靠混合加密保证机密性；篡改靠摘要 + 数字签名保证完整性；冒充靠数字证书证明身份。它不是绝对安全，因为中间人可能通过用户接受非法证书或系统导入恶意根证书成功。2. ECDHE 每次握手用临时密钥对，会话结束可丢弃临时私钥，服务器长期私钥 later 泄漏也不影响过去会话；RSA 把 pre-master 用服务器长期公钥加密传输，长期私钥泄漏可能解密过去密文。ECDHE 的共享 x 不是最终会话密钥，还要混合 Client Random、Server Random 走 PRF/HKDF 派生。3. OCSP Stapling 减少客户端单独查 CA 的延迟；会话复用减少完整握手次数。代价是会话复用会削弱前向安全，0-RTT 还有重放风险。</details>

<br>

<br>

## 总结
***
**HTTPS 用混合加密保机密、摘要+签名保完整、证书防冒充；RSA 怕私钥泄漏，ECDHE 支持前向安全；优化靠快算法、TLS1.3、ECDSA 证书、OCSP Stapling 和会话复用，但复用和 0-RTT 要权衡前向安全与重放风险。**

<br>

<br>

## 参考资料
***
- RFC 8446：The Transport Layer Security (TLS) Protocol Version 1.3
- RFC 5246：The Transport Layer Security (TLS) Protocol Version 1.2
- RFC 5280：Internet X.509 Public Key Infrastructure Certificate and CRL Profile
- RFC 6960：Online Certificate Status Protocol (OCSP)
- RFC 6961：TLS Multiple Certificate Status Request Extension（OCSP Stapling）
- RFC 8446 中 0-RTT 与重放风险相关章节
- 需按环境核实：不同站点、不同 `openssl` 版本下 `s_client` 输出差异；TLS 1.3 下 `Server Temp Key` 等字段可能不显示；nginx/OpenSSL 版本对 `ssl_conf_command Ciphersuites` 和 OCSP Stapling 的支持；证书链和协议版本取决于站点配置和中间层。

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>