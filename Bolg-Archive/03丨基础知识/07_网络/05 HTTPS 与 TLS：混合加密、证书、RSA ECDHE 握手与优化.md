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
HTTP 明文传输会窃听、篡改、冒充。HTTPS 加了 TLS 就安全了吗？为什么抓包工具又能看到 HTTPS 数据？RSA 和 ECDHE 握手差在哪？  
这一章咱们用问题链往前走：**HTTP 有什么风险 → HTTPS 怎么补 → 证书怎么证明身份 → 握手怎么协商密钥 → 怎么优化。**  
先别急着背“非对称加密、对称加密、数字证书”这几个词，先看一个具体场景：你在咖啡店连 Wi-Fi 登录网银，中间人能不能看到密码？如果能，HTTPS 是靠什么把它挡住的？挡不住的时候又是因为什么？

<br>

<br>

## 知识点 1：HTTPS 解决什么
***
### 问题
HTTPS 比 HTTP 多了什么？为什么说它解决窃听、篡改、冒充？加了 TLS 就“绝对安全”了吗？

<br>

### 方案
HTTPS 在 TCP 和 HTTP 之间加入 SSL/TLS：  
- **混合加密**保证机密性，解决窃听；  
- **摘要算法 + 数字签名**保证完整性，解决篡改；  
- **数字证书**解决冒充。  
HTTP 默认 80，HTTPS 默认 443；HTTPS 在 TCP 三次握手后还要 TLS 握手；需要 CA 证书。  
所以 HTTPS 不是新造了一个应用层协议，而是 HTTP 套了一层 TLS，把“明文明信片”变成了“加锁信封 + 验身份”。

<br>

### 最小示例
```text
HTTP：  应用层 HTTP → TCP
HTTPS： 应用层 HTTP → TLS → TCP
```

结论：HTTPS 不是新应用层协议，而是 HTTP 套了一层 TLS。  
注意：这里画的是逻辑层次，实际抓包看到的是 TCP 载荷里装着 TLS 记录，TLS 记录里装着 HTTP 报文。

<br>

### 一句话本质
**HTTPS = HTTP + TLS，给数据加锁、验指纹、验身份。**

<br>

### 易错点
坑一：**HTTPS 不是“绝对安全”**，中间人攻击成功往往是因为用户接受了非法证书，或系统被导入了恶意根证书。  
坑二：**TLS 和 SSL 可以视作同一个东西的不同阶段**，现在常说 SSL/TLS；说“SSL 证书”通常就是指 TLS 证书。  
坑三：**HTTPS 默认 443，但端口可以改**，排查时不能只看 443。

<br>

### 自测
HTTPS 解决的是 HTTP 的哪三个安全问题？分别靠什么机制？

<br>

<br>

## 知识点 2：混合加密、摘要与数字签名
***
### 问题
为什么不全用非对称加密？非对称加密不是更安全吗？数字签名到底在签什么，它和“加密正文”是一回事吗？

<br>

### 方案
非对称加密慢，对称加密快但密钥交换难。HTTPS 用**非对称手段先解决“会话密钥怎么安全地协商出来”，后续用对称加密传数据**，这就是混合加密。  
这里要区分两种“非对称手段”：  
- **RSA 密钥交换**：客户端用服务器公钥加密 pre-master 发给服务器，属于“用非对称加密传密钥”；  
- **ECDHE 密钥协商**：双方交换临时公钥，各自算出共享秘密，属于“协商”而不是“加密传输”。  
两者都服务于同一个目的：**让双方安全地得到同一把会话密钥。**  
摘要算法生成内容“指纹”；数字签名用私钥对摘要做签名运算，公钥验签，证明消息来源且未被篡改。注意：**“私钥签名、公钥验签”主要用于身份和完整性，不是用来加密大量正文。**

<br>

### 最小示例
```text
发送方：内容 → Hash → 摘要 → 私钥做签名运算 → 签名
接收方：内容 → Hash → 摘要1
        签名 → 公钥验签 → 摘要2
        比较摘要1 和 摘要2
```

结论：摘要相同且验签通过，说明内容完整且来源可信。  
注意：RSA 下常把“私钥做签名运算”口语化说成“私钥加密摘要”，但它的目的不是保密，而是签名；术语上别和“公钥加密、私钥解密”的保密用法混为一谈。

<br>

### 一句话本质
**非对称解决“密钥怎么安全给”，对称解决“正文怎么快”，摘要+签名解决“内容有没有被改、是不是你发的”。**

<br>

### 易错点
坑一：**哈希能防篡改，但不能单独证明来源**，还要数字签名。  
坑二：**数字签名不是加密正文**，通常是对摘要签名。  
坑三：**公钥加密、私钥解密**用于保密；**私钥签名、公钥验签**用于身份和完整性，目的不同。  
坑四：**“非对称交换会话密钥”在 RSA 下成立，在 ECDHE 下更准确说是“协商”**，别把两者混成同一个动作。

<br>

### 自测
为什么 HTTPS 不直接用非对称加密传所有网页数据？

<br>

<br>

## 知识点 3：数字证书与证书链
***
### 问题
公钥被伪造怎么办？服务器说“这是我的公钥”，客户端凭什么信？为什么需要 CA 和证书链？

<br>

### 方案
CA 对“持有者信息 + 公钥 + 用途 + 有效期”等做 Hash，再用 CA 私钥对 Hash 做签名，打包成数字证书。客户端用系统/浏览器内置的 CA 公钥验签。  
实际常是**根证书 → 中间证书 → 服务器证书**的信任链；根证书严格隔离，避免根失守。  
如果中间人伪造证书，浏览器会提示证书有问题；如果用户执意继续，或系统被导入恶意根证书，才可能被中间人解密。

<br>

### 最小示例
```text
CA 签发：
  证书信息 → Hash → CA 私钥做签名运算 → 签名
  数字证书 = 证书信息 + 签名

客户端验证：
  证书信息 → Hash → H1
  签名 → CA 公钥验签 → H2
  H1 == H2 ? 可信 : 不可信
```

结论：证书把“公钥”和“身份”绑在一起，客户端沿证书链一路验到信任的根。

<br>

### 一句话本质
**证书就是 CA 给公钥开的“身份证明”，证书链就是一层信任一层。**

<br>

### 易错点
坑一：**证书验证不只看单张证书**，还要走信任链，检查有效期、域名、用途等。  
坑二：**根证书不能随便导入**，导入恶意根证书等于把家门钥匙给别人。  
坑三：**抓包工具能看 HTTPS，是因为它让你信任了它的根证书**，不是 HTTPS 本身有漏洞。

<br>

### 自测
为什么说“抓包工具能看 HTTPS”不等于 HTTPS 不安全？

<br>

<br>

## 知识点 4：RSA 与 ECDHE 握手
***
### 问题
TLS 握手到底交互什么？RSA 和 ECDHE 差在哪？为什么现代更推荐 ECDHE？前向安全又是什么？

<br>

### 方案
RSA 握手大致：ClientHello（TLS 版本、密码套件、Client Random）→ ServerHello（Server Random、密码套件、证书）→ 客户端验证证书，用服务器 RSA 公钥加密 pre-master 发送 → 服务端解密，双方用三个随机数生成会话密钥，Change Cipher Spec + Encrypted Handshake Message。  
RSA **不支持前向安全**：服务器私钥泄漏，过去截获的密文可被破解。  
ECDHE 握手：双方交换椭圆曲线公钥，各自用私钥和对方公钥算出共享点，取 x 坐标作为共享秘密，再结合 Client Random、Server Random 生成会话密钥；支持**前向安全**，且可 False Start：客户端在发送自己的 Finished 之后、收到服务器 Finished 之前就发加密应用数据，把 TLS 往返从 2 RTT 降到约 1 RTT。  
注意：**ECDHE 中 x 不是最终会话密钥**，还要混合两个随机数走 PRF/HKDF。

<br>

### 最小示例
```text
RSA：
  Client Random + Server Random + pre-master（公钥加密传）
  → 会话密钥

ECDHE：
  Client Random + Server Random + ECDHE 共享 x
  → 会话密钥
```

结论：RSA 把“密钥材料”加密传过去；ECDHE 是双方各自算出同一个共享秘密，谁也没直接传会话密钥。

<br>

### 一句话本质
**RSA 靠公钥传密钥但怕私钥泄漏；ECDHE 靠临时密钥协商，支持前向安全。**

<br>

### 易错点
坑一：**ECDHE 的 x 不是最终会话密钥**，它还要和两个随机数一起派生。  
坑二：**TLS 1.2 完整握手常见 2 RTT，TLS 1.3 简化到 1 RTT**；False Start 是在 TLS 1.2 上省往返的技巧，不是所有场景都开。  
坑三：**RSA 不是“完全不安全”**，而是不支持前向安全；现代更常用 ECDHE，RSA 更多用于签名和证书。

<br>

### 自测
为什么 ECDHE 支持前向安全，而 RSA 不支持？

<br>

<br>

## 知识点 5：HTTPS 优化
***
### 问题
HTTPS 有性能损耗，怎么优化？会话复用和 0-RTT 是不是越多越好？

<br>

### 方案
- **硬件**：选计算力强、支持 AES-NI 的 CPU。  
- **软件**：升级 Linux/OpenSSL。  
- **协议**：选 **ECDHE + x25519**，尽量升级 **TLS 1.3**。  
- **证书**：选 ECDSA 证书，开启 **OCSP Stapling**，减少客户端查 CA。  
- **会话复用**：Session ID、Session Ticket、TLS1.3 PSK。  
会话复用会削弱前向安全；0-RTT 还有重放攻击风险，要对会话密钥设合理过期时间，敏感请求慎用。

<br>

### 最小示例
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ecdh_curve x25519;

# TLS 1.2 及以下套件
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384';

# TLS 1.3 套件：nginx 的 ssl_ciphers 不直接管 TLS 1.3，
# 需要 OpenSSL 1.1.1+ 和 nginx 1.19.4+，用 ssl_conf_command 设置
ssl_conf_command Ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384;

ssl_stapling on;
```

结论：优先快曲线、快套件、TLS1.3、OCSP Stapling 和会话复用；但注意 TLS 1.3 套件不能靠传统 `ssl_ciphers` 配。

<br>

### 一句话本质
**HTTPS 优化就是少握手、快计算、小证书、少查 CA。**

<br>

### 易错点
坑一：**会话复用会削弱前向安全**，密钥泄漏可能影响复用会话。  
坑二：**0-RTT 不是所有请求都能随便用**，它有重放风险，敏感操作要谨慎。  
坑三：**不是所有客户端都支持最新算法**，要考虑兼容。  
坑四：**nginx 的 `ssl_ciphers` 不直接控制 TLS 1.3 套件**，TLS 1.3 要用 `ssl_conf_command Ciphersuites` 或对应 OpenSSL 配置。

<br>

### 自测
为什么 OCSP Stapling 能减少证书验证延迟？

<br>

<br>

## 完整可复现示例：用 openssl 看证书和 TLS 握手
***
```bash
set -e

echo "=== 1. 查看服务器证书 ==="
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

echo "=== 2. 查看 TLS 版本和密码套件 ==="
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | grep -E 'Protocol|Cipher'

echo "=== 3. 查看证书链 ==="
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null 2>/dev/null \
  | grep -E '^[[:space:]]*(s|i):'
```

输出/现象/结论可能类似：

```text
=== 1. 证书 ===
subject=...
issuer=...
notBefore=...
notAfter=...

=== 2. TLS 版本和套件 ===
Protocol: TLSv1.3
Cipher: TLS_AES_256_GCM_SHA384

=== 3. 证书链 ===
s:...（服务器证书）
i:...（中间证书）
```

验证结果：证书包含持有者、签发者、有效期；TLS 版本和密码套件决定握手和加密方式；证书链展示信任层级。  
注意：不同站点、不同 OpenSSL 版本输出格式可能不同；`example.com` 的证书和协议版本会随站点配置变化，重点看结构。

<br>

<br>

## 自测三问
***
1. HTTPS 解决窃听、篡改、冒充分别靠什么机制？  
2. 为什么 ECDHE 支持前向安全，而 RSA 不支持？  
3. OCSP Stapling 和会话复用分别优化了什么？会话复用又带来了什么代价？

<br>

<br>

## 一句话总结全章
***
**HTTPS 用混合加密保机密、摘要+签名保完整、证书防冒充；RSA 怕私钥泄漏，ECDHE 支持前向安全；优化靠快算法、TLS1.3、ECDSA 证书、OCSP Stapling 和会话复用，但复用和 0-RTT 要权衡前向安全与重放风险。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>