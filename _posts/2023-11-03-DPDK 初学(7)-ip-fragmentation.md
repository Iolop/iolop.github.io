---
categories: [DPDK noob]
tags: [dpdk]
---

# Ip_fragmentation example

到现在为止，有一点不理解，那就是使用vfio-pci驱动的网卡，怎么ifconfig看不到。在stackoverflow看到了[一个解答](https://stackoverflow.com/questions/31097809/interface-goes-missing-from-ifconfig-in-dpdk)，不知道对不对，留待后续验证。

## 关键函数

### init_mem

进行一些内存初始化，这里第一次出现了`rte_lcore_to_socket_id`，函数原型如下

```c
unsigned int rte_lcore_to_socket_id(unsigned int lcore_id)
Get the ID of the physical socket of the specified lcore
```

获取特定逻辑核的物理端口id，这里应该是对[NUMA架构](https://en.wikipedia.org/wiki/Non-uniform_memory_access)中的物理节点。NUMA架构可以简化的理解成，每一个或者一组CPU现在拥有了部分很靠近自己的内存，这能让它们在访问这部分内存时拥有更快的速度，同样的，如果出现了miss，那么跨结点访问内存也会花费更多的时间。上文中提到CPU+RAM的组合，被称为NUMA的一个node。如果想更详细的了解，可以[阅读一下这里](https://zhuanlan.zhihu.com/p/62795773)

然后对每一个socket建立所属的mbuf_pool，

```c
socket = rte_lcore_to_socket_id(lcore_id);
mp = rte_pktmbuf_pool_create(buf, NB_MBUF, 32,
    0, RTE_MBUF_DEFAULT_BUF_SIZE, socket);
socket_direct_pool[socket] = mp;

mp = rte_pktmbuf_pool_create(buf, NB_MBUF, 32, 0, 0,
    socket);
socket_indirect_pool[socket] = mp;
```

初始化lpm和lpm6

```c
lpm_config.max_rules = LPM_MAX_RULES;
lpm_config.number_tbl8s = 256;
lpm_config.flags = 0;

lpm = rte_lpm_create(buf, socket, &lpm_config);
socket_lpm[socket] = lpm;
lpm6 = rte_lpm6_create(buf, socket, &lpm6_config);
```

### main

`init_mem`之后开始进行port的初始化，这里用到了一个rte_eth_conf结构，之前也有用这个结构[配置对应的eth_dev](https://iolop.github.io/posts/DPDK-%E5%88%9D%E5%AD%A6(5)-flow-filtering/#init_port)，下面的配置中mq_mode没见过，搜索了一下，这是用来配置发送队列模式。在dpdk中有这么几个选项

- RTE_ETH_MQ_TX_NONE

  最基本的情况，NIC 使用单个发送队列来处理所有传出的数据包。

- RTE_ETH_MQ_TX_DCB

  数据中心桥接（DCB，Data Center Bridging）模式，适用于数据中心网络。在此模式下，NIC 根据 DCB 配置信息分配数据包到发送队列，以确保流量的优先级和服务质量（QoS）。

- RTE_ETH_MQ_TX_VMDQ_DCB

  结合了 VMDq（Virtual Machine Device Queue）和 DCB 的模式，用于虚拟化环境，其中虚拟机需要特定的网络带宽和优先级。

- RTE_ETH_MQ_TX_VMDQ_ONLY

  仅使用 VMDq 的模式，用于虚拟化环境，NIC 会根据虚拟机的 MAC 地址将数据包分配到不同的发送队列。

```c
static struct rte_eth_conf port_conf = {
 .rxmode = {
  .mtu = JUMBO_FRAME_MAX_SIZE - RTE_ETHER_HDR_LEN -
   RTE_ETHER_CRC_LEN,
  .offloads = (RTE_ETH_RX_OFFLOAD_CHECKSUM |
        RTE_ETH_RX_OFFLOAD_SCATTER),
 },
 .txmode = {
  .mq_mode = RTE_ETH_MQ_TX_NONE,
  .offloads = (RTE_ETH_TX_OFFLOAD_IPV4_CKSUM |
        RTE_ETH_TX_OFFLOAD_MULTI_SEGS),
 },
};
```

以下代码对`lcore_queue_conf`进行初始化，实际上就是对每一个lcore上的每一个rx_queue进行设置。

```c
qconf = &lcore_queue_conf[rx_lcore_id];
while (rte_lcore_is_enabled(rx_lcore_id) == 0 ||
         qconf->n_rx_queue == (unsigned)rx_queue_per_lcore) {

   rx_lcore_id ++;
   if (rx_lcore_id >= RTE_MAX_LCORE)
    rte_exit(EXIT_FAILURE, "Not enough cores\n");

   qconf = &lcore_queue_conf[rx_lcore_id];
  }
socket = (int) rte_lcore_to_socket_id(rx_lcore_id);

rxq = &qconf->rx_queue_list[qconf->n_rx_queue];
rxq->portid = portid;
rxq->direct_pool = socket_direct_pool[socket];
rxq->indirect_pool = socket_indirect_pool[socket];
rxq->lpm = socket_lpm[socket];
rxq->lpm6 = socket_lpm6[socket];
qconf->n_rx_queue++;//设置完一个就++，为了上面的while能进入到下一个lcore
```

随后开始设置rte_eth_dev、mtu、rxd、txd、queue等部分。这里记得检查下自己的网卡到底支不支持`static struct rte_eth_conf port_conf`这个变量内定义的`offlaods`

```c
ret = rte_eth_dev_configure(portid, 1, (uint16_t)n_tx_queue,
         &local_port_conf);

ret = rte_eth_dev_set_mtu(portid, local_port_conf.rxmode.mtu);

ret = rte_eth_dev_adjust_nb_rx_tx_desc(portid, &nb_rxd,
         &nb_txd);

// rx_queue这里是每个port设置一个
rxq_conf = dev_info.default_rxconf;
rxq_conf.offloads = local_port_conf.rxmode.offloads;
ret = rte_eth_rx_queue_setup(portid, 0, nb_rxd,
        socket, &rxq_conf,
        socket_direct_pool[socket]);
// tx_queue则是每一个port要设置多个
//使用nb_tx_queues的原因是之前的代码设置rte_eth_dev时确认了队列的可用数量，而这个数量实际上和我们的参数 -l保持一致。这意味着我们使用n个lcores，就会在每一个port上设置相应数量的tx_queue
queueid = 0;
for (lcore_id = 0; lcore_id < RTE_MAX_LCORE; lcore_id++) {
 if (rte_lcore_is_enabled(lcore_id) == 0)
  continue;

 if (queueid >= dev_info.nb_tx_queues)
  break;

 ...

 txconf = &dev_info.default_txconf;
 txconf->offloads = local_port_conf.txmode.offloads;
 ret = rte_eth_tx_queue_setup(portid, queueid, nb_txd,
         socket, txconf);
 ...
 qconf = &lcore_queue_conf[lcore_id];
 qconf->tx_queue_id[portid] = queueid;
 queueid++;
}
```

然后老操作了，启动port，开混杂模式。随即检查port能不能支持ipv4/6的ptype，顺便注册一个回调。这个check_ptype就不展开了，内容很简单。我的vmxnet3不支持IPV6 ptype。

```c
if (check_ptype(portid) == 0) {
   rte_eth_add_rx_callback(portid, 0, cb_parse_ptype, NULL);
   printf("Add Rx callback function to detect L3 packet type by SW :"
    " port = %d\n", portid);
  }
```

### init_routing_table

函数倒是简明，但是没理解为啥能注册成功。这里的`l3fwd_ipv4_route_array[i].if_out`从0-7，在`rte_lpm_addr`函数原型中，最后一个参数是`next_hop`，我的port哪里有8个，一共才2个。甚至启动参数`sudo ./build/ip_fragmentation -l 0-3 -n 3 -- -p 3 -q 2`规定了使用4lcores、0，1ports，2tx_queues。

```c
ret = rte_lpm_add(lpm,
  l3fwd_ipv4_route_array[i].ip,
  l3fwd_ipv4_route_array[i].depth,
  l3fwd_ipv4_route_array[i].if_out);
```

### main_loop

整体结构和之前的l2fwd类似，[参考这篇](https://iolop.github.io/posts/DPDK-%E5%88%9D%E5%AD%A6(6)-l2fwd/)。只是最后调整了转发函数，这里使用的是`l3fwd_simple_forward`。

### l3fwd_simple_forward

有点懵逼，这个`qconf->tx_mbufs[port_out].m_table`是怎么引入的来着。

移除Ethernet头

```c
rte_pktmbuf_adj(m, (uint16_t)sizeof(struct rte_ether_hdr));
```

## 关键流程

流程大体上和l2fwd比较接近，只是多了一个mbuf_pool的过程。还是没理解怎么给些个port设置ip才能让包进来，现在虽然能build出来跑，但是都是没有包进入的状态。
