# 02 CPU、存储、中断与浮点
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击返回目录](https://www.cnblogs.com/abnb250/articles/23347292)  

<br>

<br>

多核程序明明没有数据竞争，两个线程各改各的变量，性能却上不去；任务响应要求高，却总被普通任务拖延。**这背后是 Cache Line 伪共享和 Linux 调度策略。**

<br><br>

## 痛点引入
***
你知道有 L1/L2/L3 Cache，也知道线程会被调度，但不知道：两个无关变量为什么互相拖累？`nice` 和 `chrt` 到底改了什么？实时任务为什么能优先跑？

<br><br>

## 知识点 1：CPU 读写数据以 Cache Line 为单位
***
<br>

### 问题
多核 CPU 各改各的变量，为什么会互相影响？

<br>

### 方案
CPU 从内存读数据到 Cache 时，按 **Cache Line** 整块加载，常见 64 字节。若两个经常修改的变量落在同一 Cache Line，即使逻辑无关，也会因缓存一致性协议互相失效。

<br>

### 最小示例
```bash
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
```

典型输出 `64`。**一个 Cache Line 里的任意数据被改，整行状态都可能受影响。**

<br>

### 一句话本质
**CPU 缓存管理的最小单位不是变量，而是 Cache Line。**

<br>

### 易错点
- 变量地址连续不代表一定同 Line，要看对齐和大小。
- Cache Line 大小因架构而异，常见 64 字节。
- 伪共享不是数据竞争，逻辑上可以完全正确但性能很差。

<br>

### 自测
两个 `long` 变量相邻，是否一定在同一个 64 字节 Cache Line？不一定在哪里？

<br><br>

## 知识点 2：伪共享与避免方法
***
<br>

### 问题
两个核心分别修改变量 A 和 B，A/B 在同一 Cache Line，会发生什么？

<br>

### 方案
MESI 下，一个核心改 A，会把其他核心对应 Cache Line 置为失效；另一个核心改 B 时又要重新取回。**A 和 B 无关，却因为同处一行反复失效，这就是伪共享。** 解决：Cache Line 对齐、字节填充、把热点变量分开。

<br>

### 最小示例
```c
#include <stdalign.h>
struct {
    alignas(64) long a;
    alignas(64) long b;
} s;
```

用 64 字节对齐把 `a`、`b` 分到不同 Cache Line。**用空间换时间。**

<br>

### 一句话本质
**伪共享就是“不同变量挤同一行，写一个拖累另一个”。**

<br>

### 易错点
- 不是所有相邻变量都要填充；只有多线程频繁写热点才值得。
- 填充会浪费 Cache 空间，可能降低整体容量利用率。
- Java 等语言可用填充/继承布局，但要看 JVM 实现。

<br>

### 自测
如果两个变量只读不写，还会不会产生严重伪共享？

<br><br>

## 知识点 3：Linux 调度类与 CFS
***
<br>

### 问题
系统任务远多于 CPU 核心，Linux 先选谁？

<br>

### 方案
Linux 任务分实时任务（优先级 0~99）和普通任务（100~139）。调度类优先级大致是：**Deadline > Realtime > Fair**。普通任务由 CFS 管理，按 `vruntime` 选最少者，兼顾公平；`nice` 值影响普通任务权重。

<br>

### 最小示例
```bash
nice -n -3 ./mysqld     # 启动时调高普通任务优先级
renice -n 5 -p 1234     # 修改已运行进程
chrt -f 10 ./realtime_task  # 设为 SCHED_FIFO 实时任务
```

**`nice` 范围 -20~19，越低优先级越高；但它只调普通任务。**

<br>

### 一句话本质
**实时任务优先跑，普通任务按 CFS 公平跑；nice 调的是普通任务的权重。**

<br>

### 易错点
- `nice` 不能把普通任务变成实时任务。
- 实时任务滥用可能饿死普通任务。
- CFS 不是“绝对平均”，而是按权重分配并优先选 `vruntime` 少的任务。

<br>

### 自测
`nice -20` 和 `nice 19`，哪个普通任务优先级更高？

<br><br>

## 知识点 4：运行队列与响应要求高的任务
***
<br>

### 问题
我的任务对响应要求很高，怎么让它更容易先被调度？

<br>

### 方案
每个 CPU 有运行队列，包含 `dl_rq`、`rt_rq`、`cfs_rq`。选择顺序是 Deadline → Realtime → Fair。要求高可考虑实时调度策略 `SCHED_FIFO/SCHED_RR/SCHED_DEADLINE`，但必须评估是否会饿死其他任务。


<br>

### 最小示例
```bash
chrt -f 10 ./low_latency_task
# -f 表示 SCHED_FIFO，10 是实时优先级
```

**实时任务总是比普通任务优先。** 生产环境要谨慎，最好配合资源隔离和监控。

<br>

### 一句话本质
**想低延迟，不只是调 nice，而是选对调度类和优先级，并控制实时任务范围。**

<br>

### 易错点
- 实时优先级数值越小越高（0~99）。
- `chrt` 改错可能让系统难以调度普通任务。
- 多核下还要考虑绑核、NUMA、中断亲和性。

<br>

### 自测
为什么“把任务设成实时”不是所有低延迟场景的默认答案？

<br><br>

## 完整可复现示例：伪共享对比
***
```c
#include <stdio.h>
#include <pthread.h>
#include <stdalign.h>
#include <time.h>

#define N 100000000
alignas(64) volatile long a = 0;
alignas(64) volatile long b = 0;

void* fa(void* x){ for(long i=0;i<N;i++) a++; return 0; }
void* fb(void* x){ for(long i=0;i<N;i++) b++; return 0; }

int main(){
    pthread_t t1,t2;
    pthread_create(&t1,0,fa,0);
    pthread_create(&t2,0,fb,0);
    pthread_join(t1,0); pthread_join(t2,0);
    printf("a=%ld b=%ld\n", a, b);
    return 0;
}
```

编译运行：

```bash
gcc -O2 -pthread false_sharing.c -o fs
./fs
```

把 `alignas(64)` 去掉后再测，通常会看到差异。**热点变量分开到不同 Cache Line，可减少伪共享。**

<br><br>

## 自测三问
***
1. 伪共享和数据竞争的本质区别是什么？
2. CFS 为什么优先选 `vruntime` 少的任务？
3. `nice` 和 `chrt` 改的分别是什么？

<br><br>

## 一句话总结全章
***
**CPU 按 Cache Line 读写，热点变量同处一行会伪共享；任务调度先实时后普通，普通任务靠 CFS 和 nice 调权重。**



<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>
