---
layout: post
title: DPDK-初学(17)-vmdq-forward
date: 2023-12-25 19:24 +0800
categories: [DPDK noob]
tags: [dpdk]
---

VMDq进行L2转发时，将进入流量拆分成多个队列。简单来说，入口的包通过各自的属性，比如MAC地址，VLAN ID来进行分配，划拨到具体的rx队列中去，而这些队列组成一个pool。常见的pool数量有8，16，32，64。

## 基本流程

前面的差别都不大，基础操作。初始EAL环境，检查可用的lcore，检查可用的ports，建立mbuf_pool，随后开始初始化port。即下方这个port_init函数。

```c
port_init(portid, mbuf_pool)
```

由于每一个网卡的配置可能有所不同，所以这里给出了一个默认配置，具体的选项得自己配置，从`rte_eth_dev_info_get`获取后来设置相应的结构体。

```c
static const struct rte_eth_conf vmdq_conf_default = {
 .rxmode = {
  .mq_mode        = RTE_ETH_MQ_RX_VMDQ_ONLY,
 },

 .txmode = {
  .mq_mode = RTE_ETH_MQ_TX_NONE,
 },
 .rx_adv_conf = {
  /*
   * should be overridden separately in code with
   * appropriate values
   */
  .vmdq_rx_conf = {
   .nb_queue_pools = RTE_ETH_8_POOLS,
   .enable_default_pool = 0,
   .default_pool = 0,
   .nb_pool_maps = 0,
   .pool_map = {{0, 0},},
  },
 },
};
```

随后通过获取的pools数量，开始配置。设置好每一个pool对应的vlan_id和所属的pool。这里的`conf.pool_map[i].pools`是一个掩码表示。

```c
struct rte_eth_vmdq_rx_conf conf;
for (i = 0; i < conf.nb_pool_maps; i++) {
  conf.pool_map[i].vlan_id = vlan_tags[i];
  conf.pool_map[i].pools = (1UL << (i % num_pools));
 }
```

随后拷贝默认设置到eth_conf(定义在port_init中的`struct rte_eth_conf`结构)

```c
(void)(rte_memcpy(eth_conf, &vmdq_conf_default, sizeof(*eth_conf)));
(void)(rte_memcpy(&eth_conf->rx_adv_conf.vmdq_rx_conf, &conf,
    sizeof(eth_conf->rx_adv_conf.vmdq_rx_conf)));
if (rss_enable) {
 eth_conf->rxmode.mq_mode = RTE_ETH_MQ_RX_VMDQ_RSS;
 eth_conf->rx_adv_conf.rss_conf.rss_hf = RTE_ETH_RSS_IP |
      RTE_ETH_RSS_UDP |
      RTE_ETH_RSS_TCP |
      RTE_ETH_RSS_SCTP;
}
return 0;
```

这样，就设置好了`rte_eth_vmdq_rx_conf`这个结构体。

再来看看后续的计算。5，6主要是因为起始值可能不是从0开始。

1. 计算pf(physical function)，rx可用queue减去vmdq的数量？没看懂为啥这么算。
2. 计算每一个pool里面的队列数量。
3. 将要使用的vmdq队列总数。
4. 总队列数(pf+上面的3计算结果)。
5. vmdq队列起始值。
6. vmdq池子起始值。

```c
num_pf_queues = dev_info.max_rx_queues - dev_info.vmdq_queue_num;
queues_per_pool = dev_info.vmdq_queue_num / dev_info.max_vmdq_pools;
num_vmdq_queues = num_pools * queues_per_pool;
num_queues = num_pf_queues + num_vmdq_queues;
vmdq_queue_base = dev_info.vmdq_queue_base;
vmdq_pool_base  = dev_info.vmdq_pool_base;
```

后面就是正常初始化，不过注意一点，是初始了所有的queue。

```c
rxRings = (uint16_t)dev_info.max_rx_queues;
txRings = (uint16_t)dev_info.max_tx_queues;
...
for (q = 0; q < rxRings; q++) {
  retval = rte_eth_rx_queue_setup(port, q, rxRingSize,
     rte_eth_dev_socket_id(port),
     rxconf,
     mbuf_pool);
  if (retval < 0) {
   printf("initialise rx queue %d failed\n", q);
   return retval;
  }
 }
```

最后，给pool设定好mac地址，结束初始化。

```c
for (q = 0; q < num_pools; q++) {
  struct rte_ether_addr mac;
  mac = pool_addr_template;
  mac.addr_bytes[4] = port;
  mac.addr_bytes[5] = q;
  printf("Port %u vmdq pool %u set mac " RTE_ETHER_ADDR_PRT_FMT "\n",
   port, q, RTE_ETHER_ADDR_BYTES(&mac));
  retval = rte_eth_dev_mac_addr_add(port, &mac,
    q + vmdq_pool_base);
  if (retval) {
   printf("mac addr add failed at pool %d\n", q);
   return retval;
  }
 }
```

然后开启lcore中的转发函数，简单结构如下

```c
for (q = startQueue; q < endQueue; q++) {
 const uint16_t rxCount = rte_eth_rx_burst(sport,
  q, buf, buf_size);
 if (unlikely(rxCount == 0))
  continue;
 rxPackets[q] += rxCount;
 for (i = 0; i < rxCount; i++)
  update_mac_address(buf[i], dport);
 const uint16_t txCount = rte_eth_tx_burst(dport,
  vmdq_queue_base + core_id,
  buf,
  rxCount);
 if (txCount != rxCount) {
  for (i = txCount; i < rxCount; i++)
   rte_pktmbuf_free(buf[i]);
 }
}
```
