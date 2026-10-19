# 02 CPU、存储、中断与浮点
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击返回目录](https://www.cnblogs.com/abnb250/articles/23347278)  

<br>

<br>

## 01 CPU 如何执行程序
***
图灵机→冯诺依曼→指令周期，讲清 a = 1 + 2 怎么跑、32/64 位差在哪。  
适合：想补“代码之下”基础的人 · 约 10 分钟 · #CPU #指令周期 #位宽  
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23363583)

<br>

<br>

## 02 存储器层次：磁盘比内存慢几万倍？
***
SRAM/DRAM/SSD/HDD 的速度数量级与分级逻辑：越近越快越贵越小。  
适合：分不清缓存/内存/硬盘差距的人 · 约 8 分钟 · #存储器 #Cache #性能  
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23363584)

<br>

<br>

## 03 如何写出让 CPU 跑得更快的代码
***
围绕 Cache Line 讲数据缓存、指令缓存、分支预测与绑核。  
适合：想不改算法就拿到性能收益的人 · 约 10 分钟 · #CacheLine #缓存命中 #分支预测  
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23363585)

<br>

<br>

## 04 CPU 缓存一致性：MESI
***
写回、写传播、事务串行化，以及 MESI 四状态如何保证多核一致。  
适合：想理解多核“为什么能同时跑还对”的人 · 约 12 分钟 · #缓存一致性 #MESI #写回  
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23363586)

<br>

<br>

## 05 CPU 如何执行任务：伪共享与调度
***
伪共享怎么拖慢多线程，以及 CFS、nice、chrt、绑核怎么用。  
适合：写多线程、做性能调优的人 · 约 12 分钟 · #伪共享 #CFS #调度  
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23363587)

<br>

<br>

## 06 什么是软中断
***
硬中断/软中断上下半部，ksoftirqd 与 NET_RX 过高怎么排查。  
适合：遇到 si 高却不知从何下手的人 · 约 10 分钟 · #软中断 #ksoftirqd #性能排查  
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23363588)

<br>

<br>

## 07 为什么 0.1 + 0.2 不等于 0.3
***
补码、小数转二进制、IEEE 754，讲清浮点精度损失与比较方法。  
适合：被浮点精度坑过的人 · 约 10 分钟 · #IEEE754 #浮点数 #精度  
[点击进入文章](https://www.cnblogs.com/abnb250/articles/23363589)


<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>
