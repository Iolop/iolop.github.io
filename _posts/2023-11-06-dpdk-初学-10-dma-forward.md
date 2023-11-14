---
layout: post
title: DPDK 初学(10)-DMA-forward
date: 2023-11-06 23:44 +0800
categories: [DPDK noob]
tags: [dpdk]
---

从dpdk的DMA模块看了下，下面这个函数调用返回可用dma数量，结果是0。怀疑是虚拟机没这个功能——DMA控制器芯片。所以这里先不写了，后面看能不能搞一个真机来看看。

```bash
uint16_t dma_count =  rte_dma_count_avail();
```
