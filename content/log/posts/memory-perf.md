+++
title = '内存带宽测试'
date = 2022-10-20
draft = false
+++

这次实验有些迷惑，看似很简单的测试，怎么测都不一致，也没有成型的结论。但在实验中学习到很多知识，在实验报告中进行记录。

## 理论带宽

所使用的计算平台是 2 路 AMD 7H12 服务器，各有 8 个内存插槽，插满 16 条 3200MHz DDR4 32G 内存。理论最大内存带宽为 3200 MB * 16 * 8 = 400 GB/s.

## MLC

[Intel MLC](https://www.intel.com/content/www/us/en/download/736633/intel-memory-latency-checker-intel-mlc.html) (Memory Latency Checker) 是一款常用的内存测速软件，可以测试内存的延迟与带宽等数据，肯定比我写的靠谱，运行结果摘录如下

```
Using traffic with the following read-write ratios
ALL Reads        :	280874.6	
3:1 Reads-Writes :	267733.3	
2:1 Reads-Writes :	267746.0	
1:1 Reads-Writes :	263776.1	
Stream-triad like:	269764.7

Using Read-only traffic type
		Numa node
Numa node	     0	     1	
       0	139164.7	80525.8	
       1	80283.2		140104.7

Using Read-only traffic type
Inject	Latency	Bandwidth
Delay	(ns)	MB/sec
==========================
 00000	381.02	 280811.8
 00002	378.50	 280630.5
 00008	398.20	 280348.6
 00015	397.16	 280182.5
 00050	401.18	 280396.7
 00100	404.43	 280489.0
 00200	409.52	 280979.2
 00300	291.79	 283690.9
 00400	157.82	 237132.0
```

相比理论带宽只有 70%，确认了内存工作在 3200MHz 下

```
Speed: 3200 MT/s
Configured Memory Speed: 3200 MT/s
```

换用另一台 2 路 7H12 同内存配置服务器，速度同样为 280GB/s，同 node 下 NUMA 140GB/s. 换用 2 路 Intel 8368 同内存配置服务器，同 node 下 NUMA 带宽为 177GB/s，为理论值的 88%，仍然无法到达理论值，但却相比 AMD 有所提升。

经过讨论，认为可能存在两种情况：

- 运行时 CPU 核心均被 100% 占用，可能为计算性能瓶颈
- 在 AMD 上可能需要把 NUMA 进一步细分，参考 [AMD](https://developer.amd.com/wp-content/resources/56338_1.00_pub.pdf) 分为 NPS2 等进一步尝试

关于 Intel 为何无法跑满理论带宽，没有头绪。

## STREAM

> https://www.cs.virginia.edu/stream/FTP/Code/stream.c

使用方式

```bash
gcc -mtune=native -march=native -O3 -Ofast -mcmodel=large -fopenmp stream.c -DSTREAM_ARRAY_SIZE=600000000 -DNTIMES=30 -o stream
```

其中 STREAM_ARRAY_SIZE 按程序内说明，应至少为四倍 LLC 大小，其余为优化参数。

测试结果

```
-------------------------------------------------------------
Number of Threads requested = 128
Number of Threads counted = 128
-------------------------------------------------------------
Function    Best Rate MB/s  Avg time     Min time     Max time
Copy:          293256.7     0.033332     0.032736     0.036266
Scale:         303951.1     0.032861     0.031584     0.042619
Add:           311243.1     0.047557     0.046266     0.056702
Triad:         312568.8     0.047457     0.046070     0.060070
-------------------------------------------------------------
```

与 MLC 结果基本一致。

## 自研程序的带宽测试

使用基础带宽测试，基于如下顺序读取代码片段，写入同理

```c
const uint64_t N = ...;
const uint64_t K = ...;

double seq(volatile uint64_t *buf) {
  for (uint64_t i = 0; i < N; ++i)
    buf[i] = ...;
  uint64_t tmp;
  // for (uint64_t i = 0; i < K; ++i) {
  clock_t tic = clock();
#pragma unroll(16)
  for (uint64_t j = 0; j < N; ++j)
    tmp = buf[j];
  clock_t toc = clock();
  // }
  free((void *)buf);
  return (double)(toc - tic) / CLOCKS_PER_SEC;
}
```

在测试中发现，把 N 设置为足够大的值，即可测试内存带宽，单线程约为 10 GB/s，但多线程 K = 32 时仍只能达到 160 GB/s 左右速度，猜测多线程轮换造成了部分 unseq 读取，但是没有想到好的解决办法。

[Zongy](https://blog.csdn.net/weixin_43614211/article/details/123607543) 提到了很好的阶梯状 L1, L2, L3 带宽测试方式，复现结果很扭曲，猜测是当前服务器还有其他用户，正在运行一些 memory bound 程序，使得 Cache 无法被独占使用。

在讨论时发现 mmap 后是否 memset 显著影响带宽测试结果，然而相似代码在我处无法复现。后与更多人讨论，发现以下特性：[Intel Zero Opt](https://travisdowns.github.io/blog/2020/05/13/intel-zero-opt.html)，简单来说 Intel 在已经为全 0 的 cache line 中的一些 store 会被优化，但 AMD 未观察到类似优化，因此结果无法互相验证。但测试过程中还发现，无论 memset 0 或 1，带宽测试结果相似，与不 memset 产生较大不一致，这与文中 zero-over-zero stores 又产生了不一致。

## 粗略结论

跑不到理论带宽，AMD 只有 70%，Intel 损失了大约 10% 理论值。芯片都有一些优化，测试内存、缓存带宽需要明确使用场景，从而得到更准确的结果。