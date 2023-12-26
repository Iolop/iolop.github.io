---
layout: post
title: DPDK-初学(16)-packet-ordering
date: 2023-12-06 21:55 +0800
categories: [DPDK noob]
tags: [dpdk]
---

将一个流中的包进行重排序。该示例至少需要三个lcore。

- Rx-core：负责从NIC接收数据包，通过软件队列发送给worker
- Worker-core：从软件队列接收包，并设置好转发port。
- Tx-core：从worker通过软件队列获取包，将乱序的包插入到重排序缓冲区，并取出排序后的包进行转发。

## 简要流程

前面都是基础的初始化操作，注意一下通过`rte_ring_create`建立了两个软件队列，用在后面的数据包传递上。然后用`rte_reorder_create`建立了一个重排序缓冲区。
