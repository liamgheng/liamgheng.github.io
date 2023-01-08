---
layout: post
title: 网络收包流程浅析 
categories: Blog
keywords: 计算机网络 
---

此文分析数据包从到达网卡开始，到唤起线程的整个过程。

## 流程解析

### 网卡部分

目前生产环境的网卡一般支持网卡多队列，不同队列绑定到不同 CPU 核心上面来处理网络中断，从而解决在大流量时网络包堆积到单核上的处理瓶颈。

查看队列个数:。
```
#ethtool -l eth0
Channel parameters for eth0:
Pre-set maximums:
RX:		0
TX:		0
Other:		0
Combined:	16
Current hardware settings:
RX:		0
TX:		0
Other:		0
Combined:	16
```
上面结果说明网卡最多支持 16 个队列，当前开启了 16 个队列。

数据包到达网卡之后，一般是按照四元组 Hash 到某一个队列上:
```
#ethtool -n eth0 rx-flow-hash tcp4
TCP over IPV4 flows use these fields for computing Hash flow key:
IP SA
IP DA
L4 bytes 0 & 1 [TCP/UDP src port]
L4 bytes 2 & 3 [TCP/UDP dst port]
```

数据报文到达网卡队列之后，还没办法被 CPU 处理，需要把报文数据传输到内存。这里通过 DMA 控制器把数据从网卡队列 DMA 到内存的一片 RingBuffer，通过 DMA 处理可以解放 CPU 搬迁数据的开销。

### CPU 处理

报文 DMA 到内存之后会触发一个硬中断（严格来说这么讲不太严谨，下文讲 NAPI 时会细说）。

那么在多队列情况下，会往哪个 CPU 触发硬中断呢？

每个队列都对应一个中断号，每个中断号可以设置亲和的 CPU，那么「网卡队列-中断号-CPU」就在网卡队列和 CPU 建立了映射关系。

网卡队列和中断号映射在 /proc/interrupts，通过 `grep 'TxRx' /proc/interrupts | awk '{print $1, $NF}'` 找到映射关系：
```
41: eth0-TxRx-0
42: eth0-TxRx-1
43: eth0-TxRx-2
44: eth0-TxRx-3
45: eth0-TxRx-4
46: eth0-TxRx-5
47: eth0-TxRx-6
...
```

中断号和 CPU 映射关系在 /proc/irq/${中断号}/smp_affinity_list：
```
# cat /proc/irq/41/smp_affinity_list
37
```
上面代表 41号 中断绑定到了 CPU37，综合前面结果，那么 eth0-TxRx-0(eth0 网卡 0 号队列) 的网络包触发的中断在 CPU37 响应。


napi 的知识

## 容易出问题的点

不均衡，处理慢。irqbalance。
注意报文传输到网卡队列时的一个异常情况是报文满了。

## 参考