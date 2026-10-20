# 02 malloc 是如何分配内存的？
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击返回目录](https://www.cnblogs.com/abnb250/articles/23369031)  

<br>

<br>

## 痛点引入
***
你写 `void* p = malloc(1);`，它到底向操作系统要了 1 字节吗？为什么 `free(p)` 只传一个地址，就能知道释放多大？为什么申请 1 字节，进程堆却涨了 132KB？**malloc 不是系统调用，而是 C 库的内存分配器；它先管虚拟内存，再按 brk/mmap 策略向内核批发内存。**

<br>

<br>

## 知识点 1：malloc 分配的是虚拟内存
***
### 问题
***
`malloc` 返回成功，就代表物理内存已经到手了吗？

<br>

### 方案
***
不是。`malloc` 是 C 库函数，返回的是**虚拟内存**。只有读写这块地址触发缺页中断后，内核才建立虚拟页到物理页的映射。**malloc 成功 ≠ 物理内存已分配。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <cstdlib>

int main() {
    void* p = std::malloc(1024);
    std::cout << "p=" << p << "\n";
    std::free(p);
    return 0;
}
```
编译运行：
```bash
g++ malloc1.cpp -o malloc1
./malloc1
```
输出类似：
```text
p=0x55...
```
**易错点：** 地址打印出来不代表物理页已分配；不检查 `nullptr` 也是常见错误。

<br>

### 一句话总结
***
**malloc 先给你虚拟地址，真正碰这块内存时才可能分配物理页。**

<br>

### 自测
***
自测：`malloc(1GB)` 成功后立刻 `sleep`，进程 RSS 一定涨 1GB 吗？

<br>

<br>

## 知识点 2：brk 和 mmap 两条路
***
### 问题
***
malloc 怎么向操作系统要内存？都用同一种方式吗？

<br>

### 方案
***
glibc 常见策略：**小于 128KB 走 `brk()` 从堆顶扩展；大于等于 128KB 走 `mmap()` 私有匿名映射**。阈值随 glibc 版本可能不同。**小块走堆内存池，大块走 mmap。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <cstdlib>

int main() {
    void* small = std::malloc(1);
    void* big = std::malloc(128 * 1024);
    std::cout << "small=" << small << "\n";
    std::cout << "big=" << big << "\n";
    std::free(small);
    std::free(big);
    return 0;
}
```
编译运行：
```bash
g++ malloc2.cpp -o malloc2
./malloc2
```
输出类似：
```text
small=0x55...
big=0x7f...
```
**易错点：** 地址高低只能辅助判断，不能绝对证明走 brk 还是 mmap；要结合 `/proc/self/maps` 和 glibc 版本。

<br>

### 一句话总结
***
**malloc 小块用 brk 从堆里批发，大块用 mmap 单独映射。**

<br>

### 自测
***
自测：为什么大块内存更倾向于 mmap，而不是一直扩堆？

<br>

<br>

## 知识点 3：malloc(1) 可能预分配一大块
***
### 问题
***
只申请 1 字节，为什么堆可能变大很多？

<br>

### 方案
***
malloc 的分配器会预分配更大的内存池，减少频繁系统调用。以 Ptmalloc2 为例，`malloc(1)` 可能实际拿到 132KB 的堆空间。**你申请的是逻辑块，分配器批发的是内存池。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <cstdlib>
#include <unistd.h>

int main() {
    void* p = std::malloc(1);
    std::cout << "pid=" << getpid() << " p=" << p << "\n";
    std::cout << "run: cat /proc/" << getpid() << "/maps | grep heap\n";
    std::cin.get();
    std::free(p);
    return 0;
}
```
编译运行：
```bash
g++ malloc3.cpp -o malloc3
./malloc3
```
另开终端：
```bash
cat /proc/<pid>/maps | grep heap
```
输出类似：
```text
55...000-55...000 rw-p ... [heap]
```
**易错点：** 不同 glibc、不同分配器结果不同；不要把 132KB 当固定值。

<br>

### 一句话总结
***
**malloc(1) 不等于只向内核要 1 字节，分配器会批发内存池。**

<br>

### 自测
***
自测：为什么分配器不每次都严格按用户请求大小向内核申请？

<br>

<br>

## 知识点 4：free 后内存归不归还？
***
### 问题
***
`free(p)` 之后，内存一定还给操作系统了吗？

<br>

### 方案
***
看来源：`brk` 来的小块通常留在 malloc 内存池里复用，不立刻还给 OS；`mmap` 来的大块 `free` 后通常直接 `munmap` 归还。**小块缓存复用，大块真还。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <cstdlib>
#include <unistd.h>

int main() {
    void* p = std::malloc(128 * 1024);
    std::cout << "pid=" << getpid() << " p=" << p << "\n";
    std::cin.get();
    std::free(p);
    std::cout << "freed big block\n";
    std::cin.get();
    return 0;
}
```
编译运行后查看：
```bash
cat /proc/<pid>/maps
```
**易错点：** `free` 后继续用指针是 use-after-free；`free(nullptr)` 合法，重复 `free` 同一指针是未定义行为。

<br>

### 一句话总结
***
**brk 小块 free 后常进内存池，mmap 大块 free 后更可能归还系统。**

<br>

### 自测
***
自测：`free` 后堆内存还在 `/proc/self/maps` 里，是否说明 free 没生效？

<br>

<br>

## 知识点 5：free 为什么知道大小？
***
### 问题
***
`free(p)` 只给一个地址，它怎么知道要释放多大？

<br>

### 方案
***
malloc 返回给用户的地址前面通常有**块头/描述信息**，记录块大小等元数据。free 会从用户地址向左偏移读头部，从而知道释放范围。**大小信息藏在返回地址前面的头部里。**

<br>

### 最小示例
***
```cpp
#include <iostream>
#include <cstdlib>

int main() {
    int* p = static_cast<int*>(std::malloc(sizeof(int)));
    *p = 42;
    std::cout << "user ptr=" << p << " value=" << *p << "\n";
    std::free(p);
    return 0;
}
```
编译运行：
```bash
g++ malloc5.cpp -o malloc5
./malloc5
```
输出类似：
```text
user ptr=0x55... value=42
```
**易错点：** 不要自己改块头，也不要对非 malloc 返回的指针调用 free；C++ 更推荐 `new/delete` 或智能指针。

<br>

### 一句话总结
***
**free 靠返回地址前面的块头知道大小，所以只能释放 malloc 系返回的原始指针。**

<br>

### 自测
***
自测：`p + 1` 传给 `free` 会发生什么？为什么？

<br>

<br>

## 完整可运行/可复现示例
***
```cpp
#include <iostream>
#include <cstdlib>
#include <unistd.h>

int main() {
    void* a = std::malloc(1);
    void* b = std::malloc(128 * 1024);

    std::cout << "pid=" << getpid() << "\n";
    std::cout << "small=" << a << "\n";
    std::cout << "big=" << b << "\n";
    std::cout << "cat /proc/" << getpid() << "/maps\n";
    std::cin.get();

    std::free(a);
    std::free(b);
    std::cout << "freed\n";
    std::cin.get();
    return 0;
}
```
编译运行：
```bash
g++ malloc_all.cpp -o malloc_all
./malloc_all
```
另开终端：
```bash
cat /proc/<pid>/maps
```
**注意：** 依赖 Linux；结果受 glibc 版本和分配器影响。

<br>

<br>

## 自测三问
***
1. malloc 返回的是虚拟内存还是物理内存？
2. 小块和大块分别更可能走 brk 还是 mmap？
3. free 为什么必须传 malloc 返回的原始地址？

<br>

<br>

## 一句话总结全章
***
**malloc 是 C 库分配器：先给虚拟内存，小块用 brk 批发进内存池，大块用 mmap 单独映射，free 靠块头知道大小，小块缓存复用、大块更可能归还系统。**

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>
