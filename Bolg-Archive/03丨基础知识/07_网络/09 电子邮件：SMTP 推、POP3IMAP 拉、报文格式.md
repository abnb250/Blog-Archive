# 09 电子邮件：SMTP 推、POP3/IMAP 拉、报文格式
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击回到目录](https://www.cnblogs.com/abnb250/articles/23375238)

<br>

<br>

## 痛点引入
***
点“发送”后邮件经历了什么？对方没开电脑，信会丢吗？为什么发邮件和收邮件不是同一个协议？  
这一章咱们用问题链往前走：**邮件怎么存不丢 → SMTP 怎么发 → 报文怎么组织 → 客户端怎么收。**  
先别急着背 SMTP、POP3、IMAP 这三个缩写，先看一个具体场景：你给朋友发邮件，朋友电脑关着、邮件服务器也正好在维护，这封信会直接退回来吗？再想想，你手机上读完一封邮件，电脑网页版为什么也显示已读？这背后其实分别是“发送”和“收取”两套不同的机制在干活。

<br>

<br>

## 知识点 1：邮件三大件与离线不丢信
***
### 问题
邮件系统由什么组成？对方没开电脑，信会丢吗？发送方服务器和目标服务器之间到底谁在等谁？

<br>

### 方案
邮件系统可以拆成三大件：  
- **用户代理 UA**：Outlook、Foxmail、网页邮箱，负责写信和看信。  
- **邮件服务器**：有邮箱，也有外出队列，负责暂存和转发。  
- **SMTP**：负责把邮件从客户端提交到服务器，或在服务器之间中继。  
接收方服务器关机时，发送方服务器不会立刻退信，而是把信放进外出队列，约每 30 分钟重试一次，连续几天送不到才退信。所以“对方没开电脑”不等于“信会丢”，真正决定丢不丢的是发送方服务器愿不愿意继续重试。

<br>

### 最小示例
```text
Alice 写邮件 → Alice 的邮件服务器（SMTP 提交）
Alice 的服务器把信放入外出队列
Alice 的服务器尝试连接 Bob 的服务器（SMTP 中继）
Bob 的服务器把信存入 Bob 邮箱
Bob 用 UA 从服务器拉取邮件（POP3/IMAP/HTTP）
```

结论：发送和接收是两段独立的过程；发送方服务器负责“送到”，接收方服务器负责“存着”，UA 负责“看”。

<br>

### 一句话本质
**UA 是笔，邮件服务器是邮局，SMTP 是快递员；信没送到就暂存重试。**

<br>

### 易错点
坑一：**SMTP 既用于客户端提交，也用于服务器间中继**，不只是“服务器之间”。  
坑二：**目的服务器宕机不会立刻退信**，会先重试；退信是重试失败后的结果。  
坑三：**邮件服务器不是“永远在线”也行**，发送方服务器会暂存重试，但重试有期限。  
坑四：**“离线不丢信”靠的是发送方服务器的队列**，不是接收方客户端。

<br>

### 自测
如果 Bob 的邮件服务器一直宕机，Alice 发出的邮件会经历什么？

<br>

<br>

## 知识点 2：SMTP 是“推”的协议
***
### 问题
SMTP 怎么对话？为什么说它是“推”的？`MAIL FROM` 和报文里的 `From:` 是一回事吗？

<br>

### 方案
SMTP 基于 TCP，默认端口 25；客户端提交常用 587，通常配合 STARTTLS 加密。典型命令顺序是：`HELO`/`EHLO` → `MAIL FROM` → `RCPT TO` → `DATA`（单独一行 `.` 结束）→ `QUIT`。  
说它“推”，是因为**发送方主动把邮件推给接收方服务器**，而不是等对方来拉。  
SMTP 原本只支持 7 位 ASCII，发图片、视频、附件要靠 MIME 编码。  
关键区分：**SMTP 命令里的 `MAIL FROM` 是信封寄件人，报文首部里的 `From:` 是信纸落款**，两码事；前者用于投递和退信，后者是给收件人看的显示信息。

<br>

### 最小示例
一次 SMTP 对话大致长这样：

```text
220 smtp.example.com ESMTP
EHLO alice.com
250 Hello
MAIL FROM: <alice@alice.com>
250 Sender ok
RCPT TO: <bob@bob.com>
250 Recipient ok
DATA
354 Enter mail, end with "." on a line by itself
From: alice@alice.com
To: bob@bob.com
Subject: Ketchup?

Do you like ketchup?
.
250 Message accepted for delivery
QUIT
221 closing connection
```

注意：`DATA` 之后先写报文首部，空一行，再写正文；最后单独一行 `.` 表示结束。

<br>

### 一句话本质
**SMTP 就是一套死板的邮局黑话，按顺序递材料，最后用单独一行点号说“我说完了”。**

<br>

### 易错点
坑一：**`MAIL FROM` 和 `From:` 不是一回事**，前者是信封，后者是信纸落款。  
坑二：**命令格式必须对**，比如 `MAIL FROM:` 后要有尖括号；`DATA` 结束的点号必须单独一行。  
坑三：**SMTP 原本只支持 ASCII**，发图片视频要 MIME 编码。  
坑四：**端口 25 和 587 用途不同**，25 常用于服务器间中继，587 常用于客户端提交；别混着配。

<br>

### 自测
SMTP 交互中，哪条命令告诉服务器“收件人是谁”？`MAIL FROM` 和 `From:` 分别在哪一步出现、起什么作用？

<br>

<br>

## 知识点 3：报文格式与收信协议
***
### 问题
一封邮件怎么组织？为什么收信不用 SMTP？POP3 和 IMAP 到底差在哪，多设备同步该选谁？

<br>

### 方案
邮件报文按 RFC 5322 组织：**首部、空行、主体**。首部有 `From`、`To`、`Subject` 等字段；空行不能少，它是首部和主体的分隔符；主体可以是纯文本，也可以是 MIME 多媒体。  
收信是“拉”：  
- **POP3**：默认 110，加密 995，偏单设备下载，下载后可能删除服务器邮件。  
- **IMAP**：默认 143，加密 993，邮件留在服务器上管理文件夹，多端同步已读未读和文件夹状态。  
- **HTTP/HTTPS**：用于 Web 邮箱，浏览器直接和邮件服务器交互。  
所以“发”和“收”用不同协议，是因为它们解决的问题不同：SMTP 负责投递，POP3/IMAP 负责让用户访问邮箱。

<br>

### 最小示例
一封最小邮件报文：

```text
From: alice@example.com
To: bob@example.com
Subject: Test

这是正文。
```

结论：首部是抬头，空行分隔，主体是正文；少空行会解析错乱。  
带附件的邮件还会出现 `MIME-Version`、`Content-Type: multipart/mixed`、`boundary` 等字段，把正文和附件分成不同部分。

<br>

### 一句话本质
**SMTP 寄信，POP3 把信箱搬回家，IMAP 在邮局里管理信箱。**

<br>

### 易错点
坑一：**POP3/IMAP 不是发邮件**，它们是收信/访问邮箱的协议。  
坑二：**多设备同步选 IMAP**，POP3 下载后可能删服务器邮件，导致另一台设备看不到。  
坑三：**首部和主体之间空行不能少**，少了会把首部当正文或解析失败。  
坑四：**IMAP 的“已读”状态存在服务器上**，所以多端能同步；POP3 通常没有这种服务器端状态管理。

<br>

### 自测
手机和电脑要看到相同邮件文件夹状态，该用 POP3 还是 IMAP？为什么？

<br>

<br>

## 完整可复现示例：用 telnet 模拟 SMTP 发信
***
```bash
# 需要可用的 SMTP 服务器；没有就用本地测试工具
# 下面以 smtp.example.com 为例，实际请换成你能连上的测试服务器
telnet smtp.example.com 25
```

连接成功后，按顺序输入：

```text
EHLO yourdomain.com
MAIL FROM: <sender@yourdomain.com>
RCPT TO: <receiver@yourdomain.com>
DATA
From: sender@yourdomain.com
To: receiver@yourdomain.com
Subject: Test Email

This is a test body.
.
QUIT
```

输出/现象/结论可能类似：

```text
220 smtp.example.com ESMTP
250 Hello yourdomain.com
250 Sender OK
250 Recipient OK
354 Enter mail, end with "." on a line by itself
250 Message accepted for delivery
221 Closing connection
```

验证结果：`250 Message accepted` 表示服务器已把邮件放入队列；之后客户端可用 POP3/IMAP/HTTP 收信。  
注意：很多 SMTP 服务器现在要求 TLS 和认证，直接 `telnet` 到 25 端口可能被拒绝或无法投递；这个示例的重点是看命令顺序和响应码，不是真的发一封能送达的信。实际测试建议用本地邮件测试工具或自己的测试服务器。

<br>

<br>

## 自测三问
***
1. 如果接收方服务器关机，发送方服务器会怎么处理？  
2. SMTP 的 `MAIL FROM` 和报文首部 `From:` 有什么区别？  
3. 为什么多设备同步邮件状态要用 IMAP 而不是 POP3？

<br>

<br>

## 一句话总结全章
***
**电子邮件系统是“UA + 邮件服务器 + 协议”的协作：SMTP 推邮件，POP3/IMAP/HTTP 拉邮件，服务器提供暂存区保证离线不丢信。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>