---
layout: post
title: "tcpdump 抓包 & wireshark 分析"
---

最近两年做网络安全产品，那么排查问题时自然少不了抓包及其分析。

从我经验看，要想利用抓包高效解决问题，需要合理的抓包思路以及善用工具分析。

## 抓包思路

首先知道在哪里抓包。比如 A 服务访问 B 服务出现 Timeout，那么我们一般现在 A 抓包，过滤目的 IP 和端口只抓到 B 的流量。如果分析抓包结果发现 Timeout 是因为 TCP 建联失败，那么我们再同时抓 A 和 B，看下 B 是否收到了 SYN 还是会有回复 SYN ACK。

然后要注意抓包参数，常用的：
- 制定网卡。不指定默认抓 localhost，但也不要指定 any，因为大部分机器做了网卡聚合，比如 eth0 eth1 聚合为了 bond0 提供服务，如果抓 any 的话会同时抓到 ethx 和 bond0 的流量，一个包抓两份，在 wireshark 上表现出来大量重传，影响分析。
- 设置最大抓的数量，避免抓太多报文。
- 如果不关心 body 内容，可以设置最大抓包深度，避免抓包文件中 body 太大。

## wireshark 分析

一般是先过滤（可以结合排序，比如找大报文情况），然后使用各种统计工具从宏观上统计，或者采用流追踪（右键点击报文 Follow/TCP Stream）方法分析单个连接。

### 过滤

常用的一般是基于 7 层（http method，url），4 层（tcp header，比如 syn 标志位），3 层信息（ip）过滤。

![具体参考](https://www.wireshark.org/docs/man-pages/wireshark-filter.html)

### 统计

下面的统计方法，既可以选择统计所有抓取的报文，也可以统计过滤之后的报文。

####  Analyze/Expert Information

可以快速浏览基本统计信息，比如下图看到发送了 176 SYN，但是只收到了 75 个 SYN ACK，那么说明在 TCP 建连有丢包，是值得关注的异常。

![summary](assets/image/wireshark/summary.png)

#### Statistics/Conversations

可以根据 2/3/4 层聚合统计报文。

比如观察到 A 服务往 B 服务发送有 TCP 建连重传，A 服务持有 B 服务的地址池，轮询往 B 服务不同 IP 发送请求。

那么我们可以在 A 根据端口抓到 B 服务的报文，然后在 Wireshark 中过滤 `tcp.flags.syn==1 && tcp.flags.ack==0 && tcp.analysis.retransmission` 筛选出重传的 SYN 报文。

然后根据 IP 统计出到哪些 B 服务的 IP 存在重传，从而进一步定位问题。下图表明只到这 5 个 IP 存在重传，那么可以进一步分析这 5 台机器。

![conversation](assets/image/wireshark/conversations.png)

### Statistics/IO Graphs

以折线图观察报文趋势。

比如为了复用连接减少 TCP 握手次数而修改了 Web 框架的连接池参数，那么怎么直观衡量效果呢？

可以抓包然后过滤 TCP 握手报文，然后在 IO/Graphs 中看下 SYN 报文发送趋势。

### Statistics/HTTP/Request

统计 HTTP 报文。

![http](assets/image/wireshark/http.png)