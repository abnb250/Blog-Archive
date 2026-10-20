# 02 如何从日志分析 PV、UV？
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击返回目录](https://www.cnblogs.com/abnb250/articles/23373420)  

<br>

<br>

## 痛点引入
***
Nginx 的 `access.log` 一行一次访问，可几十万行日志直接 `cat` 会把线上机器拖垮。**日志分析要先保安全，再用 awk/sort/uniq/wc 组合出 PV、UV、终端分布和 TOP 请求。** 核心思路：取列 → 排序 → 去重计数 → 排序取 TOP。

<br>

<br>

## 知识点 1：先看大小，别急着 cat
***
### 问题
***
日志很大，直接 `cat` 会怎样？

<br>

### 方案
***
先 `ls -lh` 看大小；大文件用 `less` 按需加载，或 `tail -n` 看末尾、`tail -f` 实时跟踪；线上大文件最好 `scp` 到闲置机器再分析。**大日志别 cat，别在线上硬跑重命令。**

<br>

### 最小示例
***
```bash
ls -lh access.log
less access.log
tail -n 5 access.log
tail -f access.log
```
**易错点：** `cat` 会一次性读整个文件；`less` 不会全加载；`tail -f` 是阻塞跟踪，别在脚本里乱用。

<br>

### 一句话总结
***
**先 ls 看大小，大文件用 less/tail，线上大日志先 scp 再分析。**

<br>

### 自测
***
自测：为什么大日志不建议直接 `cat`？

<br>

<br>

## 知识点 2：PV 统计与按天分组
***
### 问题
***
总 PV 怎么算？每天 PV 怎么分组？

<br>

### 方案
***
一行日志一次访问，`wc -l access.log` 就是总 PV。按天分组：`awk '{print $4}'` 取时间列，`substr` 截年月日，`sort` 排序，`uniq -c` 计数。**uniq 前必须 sort，因为 uniq 只去相邻重复行。**

<br>

### 最小示例
***
```bash
wc -l access.log

awk '{print substr($4, 2, 11)}' access.log \
  | sort \
  | uniq -c
```
输出类似：
```text
   2300 2024-05-01
   2800 2024-05-02
```
**易错点：** 时间在第 4 列且带 `[`，`substr($4,2,11)` 从第 2 个字符截 11 位；不 sort 直接 uniq 会重复计数。

<br>

### 一句话总结
***
**总 PV 用 wc -l，按天 PV 用 awk 取日期 + sort + uniq -c。**

<br>

### 自测
***
自测：为什么 `uniq -c` 前面一定要 `sort`？

<br>

<br>

## 知识点 3：UV 统计
***
### 问题
***
日志没有用户 ID，怎么近似算访问人数？

<br>

### 方案
***
用客户端 IP 近似 UV：`awk '{print $1}'` 取第一列 IP，`sort` 排序，`uniq` 去重，`wc -l` 计数。**UV 是去重人数，PV 是访问次数；IP 近似会受 NAT 影响。**

<br>

### 最小示例
***
```bash
awk '{print $1}' access.log | sort | uniq | wc -l
```
输出类似：
```text
2589
```
**易错点：** `uniq` 不带 `-c` 只去重不计数；多个用户共用出口 IP 会低估 UV。

<br>

### 一句话总结
***
**UV 用 IP 去重近似：取第一列 → sort → uniq → wc -l。**

<br>

### 自测
***
自测：为什么用 IP 算 UV 只是近似？

<br>

<br>

## 知识点 4：按天 UV 分组
***
### 问题
***
每天独立访客数怎么算？

<br>

### 方案
***
取“日期 + IP”，`sort` 后 `uniq` 去重，再用 awk 数组按日期累加：`awk '{uv[$1]++; next} END {for (d in uv) print d, uv[d]}'`。**next 跳过后续规则，END 在所有行处理完后输出。**

<br>

### 最小示例
***
```bash
awk '{print substr($4,2,11), $1}' access.log \
  | sort \
  | uniq \
  | awk '{uv[$1]++; next} END {for (d in uv) print d, uv[d]}'
```
输出类似：
```text
2024-05-01 1200
2024-05-02 1500
```
**易错点：** 先 `uniq` 去重日期+IP，再按日期计数；顺序反了会把同一 IP 多次访问算多次。

<br>

### 一句话总结
***
**按天 UV：日期+IP 去重，再用 awk 数组按日期累加。**

<br>

### 自测
***
自测：`next` 和 `END` 在 awk 里分别起什么作用？

<br>

<br>

## 知识点 5：终端分析与 TOP 请求
***
### 问题
***
哪些客户端访问最多？哪些路径最热？

<br>

### 方案
***
User Agent 在第 12 列，路径在第 7 列。通用套路：`awk '{print $N}'` → `sort` → `uniq -c` → `sort -rn` → `head -n 3`。**取列、排序、计数、再按数值倒序取 TOP。**

<br>

### 最小示例
***
```bash
# 终端 TOP
awk '{print $12}' access.log | sort | uniq -c | sort -rn | head -n 3

# 请求路径 TOP3
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -n 3
```
输出类似：
```text
  5000 "Mozilla/5.0 ...
  1200 "curl/7.68.0"
```
**易错点：** `sort -rn` 的 `r` 是倒序、`n` 是按数值；路径里可能带查询参数，需要先截断再统计。

<br>

### 一句话总结
***
**取列 → sort → uniq -c → sort -rn → head，就能出终端和路径 TOP。**

<br>

### 自测
***
自测：`sort -rn` 和 `sort -n` 有什么区别？

<br>

<br>

## 完整可运行/可复现示例
***
```bash
#!/bin/bash
LOG=access.log

echo "total PV:"
wc -l < "$LOG"

echo "daily PV:"
awk '{print substr($4,2,11)}' "$LOG" | sort | uniq -c

echo "UV:"
awk '{print $1}' "$LOG" | sort | uniq | wc -l

echo "daily UV:"
awk '{print substr($4,2,11), $1}' "$LOG" \
  | sort | uniq \
  | awk '{uv[$1]++; next} END {for (d in uv) print d, uv[d]}'

echo "top paths:"
awk '{print $7}' "$LOG" | sort | uniq -c | sort -rn | head -n 3
```
运行：
```bash
chmod +x log_all.sh
./log_all.sh
```
**注意：** 列号依赖 nginx 默认日志格式；自定义格式要按实际列调整。

<br>

<br>

## 自测三问
***
1. PV 和 UV 分别怎么统计？为什么 UV 要用去重？
2. 为什么 `uniq` 前必须 `sort`？
3. 按天 UV 分组的命令顺序是什么？

<br>

<br>

## 一句话总结全章
***
**日志分析先保安全再看大小；PV 用 wc -l，按天 PV 用日期+sort+uniq -c；UV 用 IP 去重，按天 UV 先日期+IP 去重再 awk 数组累加；终端和 TOP 请求都是取列→sort→uniq -c→sort -rn→head。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>
