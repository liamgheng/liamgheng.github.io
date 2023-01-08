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

那么在多队列情况下，触发的中断在哪个 CPU 响应呢？

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

CPU 响应硬中断，硬中断执行很快，主要就是设置了一个标志位让后续软中断可以执行。

在响应硬中断的 CPU 上响应软中断，之所以这么设计，猜测主要是为了利用好 CPU Cache，减少去内存取数据，提高处理效率。

软中断主要是 ksoftiqrd 线程执行，系统初始化时在每一个 CPU 上都创建了一个 ksoftiqrd 线程，该线程做的事情就是不断循环是否有软中断要处理（上面硬中断做的事情就是让这里判断为 true，才能执行接下来的逻辑），如果有则开始执行。

软中断做的事情，把网卡 DMA 过来的报文从内核 RingBuffer 取出，为的是标记 RingBuffer 元素空间为可用状态，可以再被网卡 DMA 过来的数据复用。之后就是各个协议栈的处理了：
- 注册抓包的钩子（交给各种协议 handler 之前会先执行这里），所以 tcpdump 抓包会执行这里。
- 三层协议，判断报文是 arp 或者 ip 协议，交给不同 handler 处理，iptables 规则也是在这里执行的。
- 如果是 ip 协议，则进一步交给四层（tcp，udp）handler 处理。

上面有两个注意点：
1. 避免设置太复杂的 iptables 规则，增加软中断处理时间。
2. iptables 拦截的包，也能正常被抓包，因为抓包是在协议 handler 处理之前执行的。

napi 的知识

## 容易出问题的点

不均衡，处理慢。irqbalance。
注意报文传输到网卡队列时的一个异常情况是报文满了。

内核 ring buffer 慢了？drooped？

## 参考