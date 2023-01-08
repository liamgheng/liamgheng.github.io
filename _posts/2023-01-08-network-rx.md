---
layout: post
title: 网络收包流程浅析 
categories: Blog
keywords: 计算机网络 
---

此文分析数据包从到达网卡开始，到唤起线程的整个过程。

## 准备阶段

1. 创建 ksoftirqd 内核线程。

    创建 ksoftirqd 内核线程，每个 CPU 上创建一个，为的是充分利用多核并行处理软中断，提高处理效率。

    每个 ksoftirqd 线程做的事情就是 while 循环中不断判断是否有软中断需要处理，有则执行。

    注意软中断不只是网络包软中断，还有其他的：
    ```
    enum
    {
        HI_SOFTIRQ=0,
        TIMER_SOFTIRQ,
        // 下面两行是网络包软中断
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,
        BLOCK_IOPOLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
        HRTIMER_SOFTIRQ,
        RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

        NR_SOFTIRQS
    };
    ```

2. 为 NET_TX_SOFTIRQ，NET_RX_SOFTIRQ 这两种软中断注册处理函数。
3. 注册协议栈（tcp，udp，icmp）处理函数，软中断的处理函数会调用这里。
4. 启动 arp，ip，tcp，udp 模块。
5. 网络驱动初始化，调用 ethtool 采集网卡信息，修改网卡参数，背后的逻辑是这里的网路驱动实现的。
6. 启动网卡。



## 数据到来

## 容易遇到的问题

napi 机制