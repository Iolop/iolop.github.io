---
layout: post
title: DPDK-初学(13)-link-status-interrupt
date: 2023-12-03 21:45 +0800
categories: [DPDK noob]
tags: [dpdk]
---

> mdns 完成了一个icmp ping request的示例，停一下。因为感觉后续的dns协议实际上是应用层的了，完整实现和dpdk的关系不大，还是先回头继续看dpdk。在mdns中用到的一些函数还不是很熟悉，还是得多用才行。不过mdns中用到的文件配置初始化的方法倒是挺有用的，后续可以持续的借鉴。
>
> 由于dpdk的各种示例排布比较紧凑，有些库的用法直接在一起，比如l2fwd-event就涉及到了event-dev这个库的使用，看的时候我就先跳过了，基础示例看完了再回头来看这些涉及到其他库的示例。

## 关键流程

这一章其实就一个重点，那就是`rte_eth_dev_callback_register`这个函数。函数原型如下，着重关注这个枚举类型[`rte_eth_event_type`](https://doc.dpdk.org/api/rte__ethdev_8h.html#a1e6788469a92700a583d06bf079d779d)。

```c
int
rte_eth_dev_callback_register(uint16_t port_id,
   enum rte_eth_event_type event,
   rte_eth_dev_cb_fn cb_fn, void *cb_arg)
```

从函数原型大概就能看出，通过注册不同的事件类型，给我们一个回调处理的机会。
