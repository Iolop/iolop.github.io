---
layout: post
title: DPDK 初学(8)-IPv4 Multicast Sample
date: 2023-11-03 23:11 +0800
categories: [DPDK noob]
tags: [dpdk]
---

## 关键函数

### rte_eth_allmulticast_enable

前面的流程和[ip_fragmentation](https://iolop.github.io/posts/dpdk-%E5%88%9D%E5%AD%A6-7-ip-fragmentation/)基本没差，这里也就不展开了，直到这个函数。

该函数用于启动某个port的多播功能。

### init_mcast_hash

结构很简单，创建一个`rte_fbk_hash_table`然后向里面添加key即可。fbk(four byte keys)

```c
mcast_hash_params.socket_id = rte_socket_id();
mcast_hash = rte_fbk_hash_create(&mcast_hash_params);
for (i = 0; i < RTE_DIM(mcast_group_table); i++) {
  if (rte_fbk_hash_add_key(mcast_hash,
   mcast_group_table[i].ip,
   mcast_group_table[i].port_mask) < 0) {
   return -1;
  }
 }
```

### mcast_forward

去掉以太网头，获取`dst_addr`，开始查表。检查是否需要clone

```c
if (!RTE_IS_IPV4_MCAST(dest_addr) ||
     (hash = rte_fbk_hash_lookup(mcast_hash, dest_addr)) <= 0 ||
     (port_mask = hash & enabled_port_mask) == 0) {
  rte_pktmbuf_free(m);
  return;
 }
...
use_clone = (port_num <= MCAST_CLONE_PORTS &&
     m->nb_segs <= MCAST_CLONE_SEGS);
```

开始进行多播转发，其中的`mcast_out_pkt`就是关键，通过m来构造mc这个pkt。其中通过参数use_clone来控制构造的细节，一种是clone一种是增加引用计数。

```c
dst_eth_addr.as_int = ETHER_ADDR_FOR_IPV4_MCAST(dest_addr);
/* >8 End of constructing destination ethernet address. */

/* Packets dispatched to destination ports. 8< */
for (port = 0; use_clone != port_mask; port_mask >>= 1, port++) {
 /* Prepare output packet and send it out. */
 if ((port_mask & 1) != 0) {
  if (likely ((mc = mcast_out_pkt(m, use_clone)) != NULL))
   mcast_send_pkt(mc, &dst_eth_addr.as_addr,
     qconf, port);
  else if (use_clone == 0)
   rte_pktmbuf_free(m);
 }
}
```

## 关键流程

没啥好说的，流程清晰易懂。完全可以对照ip_fragmentation。
