---
layout: post
title: DPDK-初学(15)-multi-process
date: 2023-12-05 21:46 +0800
categories: [DPDK noob]
tags: [dpdk]
---

## Basic multi-process

这一种multi-process实现起来最简单，启动参数直接指定即可。`-proc-type=primary`参数指定主从关系。primary和second之间，通过ring组成一个环形。通过这个共享的内存空间进行通信。缺点就是一旦primary挂了，必须连带着second一起重来。

```c
if (rte_eal_process_type() == RTE_PROC_PRIMARY){
  send_ring = rte_ring_create(_PRI_2_SEC, ring_size, rte_socket_id(), flags);
  recv_ring = rte_ring_create(_SEC_2_PRI, ring_size, rte_socket_id(), flags);
  message_pool = rte_mempool_create(_MSG_POOL, pool_size,
    STR_TOKEN_SIZE, pool_cache, priv_data_sz,
    NULL, NULL, NULL, NULL,
    rte_socket_id(), flags);
 } else {
  recv_ring = rte_ring_lookup(_PRI_2_SEC);
  send_ring = rte_ring_lookup(_SEC_2_PRI);
  message_pool = rte_mempool_lookup(_MSG_POOL);
 }
```

## Symmetric multi-process

这一个示例启用了`--proc-type=auto`这一个参数，第一个启动的进程会被视作primary，其他被视作second。和上一个不同的是，这里primary构造了一个`pktmbuf_pool`并且负责所有的port初始化。

### smp_port_init

这一函数就是primary用来初始化port的入口，里面的port_conf比较重要。在RX方向开启RSS(Receive Side Scaling)同时卸载ip/tcp层的checksum到硬件。TX方向上没啥说的。

```c
struct rte_eth_conf port_conf = {
  .rxmode = {
   .mq_mode = RTE_ETH_MQ_RX_RSS,
   .offloads = RTE_ETH_RX_OFFLOAD_CHECKSUM
  },
  .rx_adv_conf = {
   .rss_conf = {
    .rss_key = NULL,
    .rss_hf = RTE_ETH_RSS_IP,
   },
  },
  .txmode = {
   .mq_mode = RTE_ETH_MQ_TX_NONE,
  }
};
```

随后是一系列的硬件特性支持检查。打开无可用描述符就丢包、检查硬件的rss hash-function支持。后面就是开始配置eth设备，感觉作者开始考虑到默认配置里面的有些参数可能用户硬件不支持的情况了，比如这里的卸载checksum功能和硬件rss。最后正常流程启动nic。

```c
rte_eth_dev_info_get(port, &info);
info.default_rxconf.rx_drop_en = 1;
...
rss_hf_tmp = port_conf.rx_adv_conf.rss_conf.rss_hf;
port_conf.rx_adv_conf.rss_conf.rss_hf &= info.flow_type_rss_offloads;
...
retval = rte_eth_dev_configure(port, rx_rings, tx_rings, &port_conf);
if (retval == -EINVAL) {
  printf("Port %u configuration failed. Re-attempting with HW checksum disabled.\n",port);
  port_conf.rxmode.offloads &= ~(RTE_ETH_RX_OFFLOAD_CHECKSUM);
  retval = rte_eth_dev_configure(port, rx_rings, tx_rings, &port_conf);
}
if (retval == -ENOTSUP) {
  printf("Port %u configuration failed. Re-attempting with HW RSS disabled.\n", port);
  port_conf.rxmode.mq_mode &= ~(RTE_ETH_MQ_RX_RSS);
  retval = rte_eth_dev_configure(port, rx_rings, tx_rings, &port_conf);
}
```

### lcore_main

在上面的初始完成后，每一个核都启动。通过对应的port来传递消息。每一个lcore上运行的process，向自己对应的proc_id(下面的q_id)发送消息，通过port和queue的搭配完成消息传递。

```c
for (;;) {
  struct rte_mbuf *buf[PKT_BURST];
  for (p = start_port; p < end_port; p++) {
    const uint8_t src = ports[p];
    const uint8_t dst = ports[p ^ 1]; /* 0 <-> 1, 2 <-> 3 etc */
    const uint16_t rx_c = rte_eth_rx_burst(src, q_id, buf, PKT_BURST);
    if (rx_c == 0)
      continue;
    pstats[src].rx += rx_c;
    const uint16_t tx_c = rte_eth_tx_burst(dst, q_id, buf, rx_c);
    pstats[dst].tx += tx_c;
    if (tx_c != rx_c) {
      pstats[dst].drop += (rx_c - tx_c);
      for (i = tx_c; i < rx_c; i++)
        rte_pktmbuf_free(buf[i]);
    }
  }
}
```

## Client-Server muliti-process

和上面提到的smp类似，同样采用primary初始化所有接口和需要用到的ring、pktmbuf。关键代码位于`init`函数。由primary初始化完成后，second直接寻找即可使用。和smp不同的是，这里的server使用queue中的rx队列，具体的转发业务会转交给client完成，同时client只使用tx队列。
