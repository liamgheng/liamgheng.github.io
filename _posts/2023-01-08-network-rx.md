---
layout: post
title: 网络收包流程浅析 
categories: 计算机网络
---

此文分析数据包从到达网卡开始，到被协议栈处理的整个过程。

## 流程解析

### 网卡部分

目前生产环境的网卡一般支持网卡多队列，不同队列绑定到不同 CPU 核心上面来处理网络中断，从而解决在大流量时网络包堆积到单核上的处理瓶颈。

查看队列个数:
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

软中断做的事情，把网卡 DMA 过来的报文从内核 RingBuffer 取出，然后标记 RingBuffer 被取出的元素空间为可用状态，可以再被网卡 DMA 过来的数据复用。之后就是各个协议栈的处理了：
- 注册抓包的钩子（交给各种协议 handler 之前会先执行这里），所以 tcpdump 抓包会执行这里。
- 三层协议，判断报文是 arp 或者 ip 协议，交给不同 handler 处理，iptables 规则也是在这里执行的。
- 如果是 ip 协议，则进一步交给四层（tcp，udp）handler 处理。

上面有两个注意点：
1. 避免设置太复杂的 iptables 规则，增加软中断处理时间。
2. iptables 拦截的包，也能正常被抓包，因为抓包是在协议 handler 处理之前执行的。

上面提到报文 DMA 到内存之后会触发硬中断是不严谨的，如果这样的话在高速网卡中会频繁触发硬中断，导致浪费太多 CPU 在响应硬中断，所以采用 NAPI 机制结合中断和轮询来处理网络报文：
1. 有中断来了，驱动关闭硬中断通知内核收包。
2. 内核软中断轮询网卡，在规定时间（防止造成饥饿）内尽可能多收包。时间用尽，或者没有包可收，则再次开启硬中断准备下一次收包。

## 容易出问题的点

### 中断在多核分布不均衡

首先，如何判断中断分布均衡性？有多种办法，可以采集 /proc/interrupts 来对比一段时间的变化，也可以通过 kprobes 来分析，我一般用前者：
```
#cat /proc/interrupts | grep TxRx > file1;sleep 30;cat /proc/interrupts | grep TxRx > file2;awk 'NR==FNR{for(i=1;i++<(NF-1);)a[FNR,i]=$i;next}{for(i=1;i++<(NF-1);)$i=$i-a[FNR,i]}{print $0}' file1 file2 | awk '{for(i=2;i<=NF-1;i++)$i=(a[i]+=$i)}END{print}'

304: 2 20 143 2 39 2 4 2 0 75 3 244 9 65 5 50 15 4 50 44 25 15 2 2 28 59 2 4 6 13 103 3 5 1 5 13 11 26 92 46 2 61 213 64 30 28 63 11 38 54 48 179 4 15 85 35 120 6 22 9 11 8 9 16 77 54 21 24 63 50 8 19 67 16 9 12 52 70 10 14 0 eth1-TxRx-62
```
脚本做的事情是间隔 30s 采集两次 interrupts 数据，然后相减得到差值，最后在每个 CPU 上求和。上面结果代表 CPU0 在 30s 内收到了两次硬中断，CPU2 收到了 20 次硬中断... 

中断处理不均衡，一种极端情况是中断几乎都堆积到 CPU0 上了，这种情况主要是没有配置硬中断号和 CPU 映射关系，手工配置或者借助 irqbalance 配置就好了。

### RingBuffer 满，新数据包被丢弃

ifconfig 能看到 overruns 信息，如果数字在增长，则说明有包因为 RingBuffer 满了，导致被丢弃。
```
eth0: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 78:aa:82:d2:71:f4  txqueuelen 1000  (Ethernet)
        RX packets 396750651  bytes 176746356984 (164.6 GiB)
        RX errors 0  dropped 407201  overruns 0  frame 0
        TX packets 469712230  bytes 106158266844 (98.8 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions
```

这种情况需要有两种解决办法。

首先确认软中断是否执行太慢，导致没有及时从 RingBuffer 中取走报文导致堆积。

也可以可以增加 RingBuffer 长度，查看长度：
```
#ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:		4096
RX Mini:	0
RX Jumbo:	0
TX:		4096
Current hardware settings:
RX:		4096
RX Mini:	0
RX Jumbo:	0
TX:		512
```
上面说明 eth0 网卡 RB 最长为 4096，当前为 512。

## 参考

- http://blog.hyfather.com/blog/2013/03/04/ifconfig/
- 《深入理解 Linux 网络》第二章 