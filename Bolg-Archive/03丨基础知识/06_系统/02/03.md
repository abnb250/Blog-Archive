# 02 CPU、存储、中断与浮点
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击返回目录](https://www.cnblogs.com/abnb250/articles/23347292)  

<br>

<br>


同样的二维数组遍历，`array[i][j]` 和 `array[j][i]` 结果一样，速度却可能差几倍；同样一个 `if`，数据随机和有序时速度也不同。**问题不在语法，而在 CPU Cache 和分支预测。**

<br><br>

## 痛点引入
***
你知道 CPU 有 L1/L2/L3 Cache，但不知道代码怎么写才能配合它。结果就是：逻辑没错，性能却被内存访问和缓存失效拖垮。核心目标只有一个：**提高缓存命中率。**

<br><br>

## 知识点 1：CPU Cache 按 Cache Line 读数据
***
<br>

### 问题
CPU 读 `array[0]` 时，真的只读 4 个字节吗？

<br>

### 方案
不是。CPU 从内存加载到 Cache 的基本单位是 **Cache Line**，常见 64 字节。读一个 `int` 可能顺带把后面十几个元素一起加载进来。**顺序访问连续内存，才能吃到这个红利。**

<br>

### 最小示例
```bash
getconf LEVEL1_DCACHE_LINESIZE
# 或
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
```

典型输出 `64`，表示 L1 数据缓存一次加载 64 字节。

<br>

### 一句话本质
**CPU 不是按元素搬数据，而是按 Cache Line 整块搬。**

<br>

### 易错点
- L1 常分数据缓存和指令缓存，不要混为一谈。
- L1/L2 每核私有，L3 多核共享。
- 小数组可能全在 Cache 里，差距不明显；大数据量才明显。

<br>

### 自测
一个 `int` 4 字节，64 字节 Cache Line 一次可能带入多少个连续 `int`？

<br><br>

## 知识点 2：数据缓存命中——按内存布局顺序访问
***
<br>

### 问题
二维数组 `array[i][j]` 和 `array[j][i]`，为什么前者通常快？

<br>

### 方案
C 语言二维数组按行连续存放。`array[i][j]` 顺着内存走，当前 Cache Line 里的后续元素会被继续命中；`array[j][i]` 跳跃访问，容易频繁 Cache Miss。


<br>

### 最小示例
```c
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        sum += a[i][j];   // 顺序访问，通常更快
```

而 `a[j][i]` 是跨行跳跃访问。**遇到遍历连续数据，优先按内存布局顺序访问。**

<br>

### 一句话本质
**数据缓存优化的第一招：顺着内存走，不要跳着走。**

<br>

### 易错点
- 编译器优化、数组大小、计时方式都会影响结果，实验时要注意。
- 不是所有语言二维数组都按行连续，要确认内存布局。
- 顺序访问不是玄学，本质是 Cache Line 预取式整块加载。

<br>

### 自测
Java 的二维数组是“数组的数组”，`array[i][j]` 一定连续吗？为什么不一定？

<br><br>

## 知识点 3：指令缓存与分支预测
***
<br>

### 问题
先排序再遍历，为什么有时比先遍历再排序更快？

<br>

### 方案
`if (x < 50)` 这类分支，CPU 分支预测器会根据历史预测走哪边，并提前把指令放进指令缓存。数据随机时预测难；数据有序时规律强，预测命中率高。C/C++ 可用 `likely/unlikely` 提示，但**不要滥用**。

<br>

### 最小示例
```c
if (likely(x < 50)) {
    a[i] = 0;
}
```

`likely` 告诉编译器这个分支更可能为真，便于布局指令。但现代 CPU 动态预测已很强，**只有非常确信且收益明确时才用**。

<br>

### 一句话本质
**有规律的分支让 CPU 猜得准，指令缓存命中率更高。**

<br>

### 易错点
- “先排序再遍历快”通常指遍历阶段；排序本身有成本，要算总账。
- `likely/unlikely` 不是性能银弹，滥用可能反优化。
- 分支预测属于指令缓存/执行层面，不是数据缓存。

<br>

### 自测
为什么随机数据上的 `if` 比有序数据上的同样 `if` 更难预测？

<br><br>

## 知识点 4：多核绑核提高私有缓存命中
***
<br>

### 问题
计算密集型线程在多核间来回切换，为什么性能会下降？

<br>

### 方案
L1/L2 每核私有。线程换核后，原核心 Cache 里的数据新核心不一定有，命中率下降。可用 `sched_setaffinity` 或 `taskset` 把线程绑定到某个核心。

<br>

### 最小示例
```bash
taskset -c 0 ./my_program
```

表示让程序主要在 0 号 CPU 上运行。**绑核能提高私有缓存命中率，但可能造成负载不均。**

<br>

### 一句话本质
**让线程别乱跑，是为了让它复用同一个核心的 L1/L2 Cache。**

<br>

### 易错点
- 绑核不是越多越好；核心少、任务多时可能排队。
- 实时性、NUMA、负载均衡都要综合考虑。
- 绑核解决的是缓存切换问题，不解决算法复杂度问题。

<br>

### 自测
一个 8 核机器上跑 8 个计算密集线程，是否一定要绑核？为什么？

<br><br>

## 完整可运行示例：二维数组行/列遍历计时
***
```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N 4096

static double now() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec + ts.tv_nsec / 1e9;
}

int main() {
    int *a = malloc(sizeof(int) * N * N);
    for (int i = 0; i < N * N; i++) a[i] = i;

    volatile long sum = 0;
    double t1 = now();
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++)
            sum += a[i * N + j];
    double t2 = now();

    for (int j = 0; j < N; j++)
        for (int i = 0; i < N; i++)
            sum += a[i * N + j];
    double t3 = now();

    printf("row-major: %.3f s\n", t2 - t1);
    printf("col-major: %.3f s\n", t3 - t2);
    printf("sum=%ld\n", sum);
    free(a);
    return 0;
}
```

编译运行：

```bash
gcc -O2 cache.c -o cache
./cache
```

现象：通常 `row-major` 明显快于 `col-major`。结论：**按内存布局顺序访问，能有效利用 Cache Line。**

<br><br>

## 自测三问
***
1. 为什么读一个元素可能顺带加载后面多个元素？
2. `likely/unlikely` 解决的是数据缓存还是指令/分支预测问题？
3. 多核绑核提升性能的前提和代价分别是什么？

<br><br>

## 一句话总结全章
***
**让 CPU 跑得快，不是背术语，而是让数据顺序访问、让分支有规律、让线程尽量复用同一个核心的 Cache。**



<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>
