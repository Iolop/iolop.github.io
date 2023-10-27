---
categories: [DPDK noob]
tags: [dpdk]
---

# skeleton example

## 关键流程

- 初始化EAL环境
  形如`application -l 0-3 -n 4 -- -q`其中通过`--`分隔，前部分为EAL参数，后部分为应用所需参数

  ```c
  argc -= ret;
  argv += ret;
  ```

- 建立mbuf_pool
- 轮询所有ports，为每一个ports初始化
- 使用一个逻辑核开始收发包处理

## 关键函数

- rte_eth_dev_count_avail

  该函数用来获取当前所有可用端口，在dpdk中端口不是平时网络编程中常见的port，这里的port我觉得可以理解成一个可用的抽象NIC(Network Interface Card)。由于本示例是转发测试，所以至少需要两个ports来进行。

- rte_pktmbuf_pool_create

  函数原型

  ```c
  struct rte_mempool *
  rte_pktmbuf_pool_create(const char *name, unsigned int n,
   unsigned int cache_size, uint16_t priv_size, uint16_t data_room_size,
   int socket_id)
  ```

  name: 缓冲区名字

  n: 缓冲区内mbuf的数量

  cache_size: 每一个核应该缓存多少mbuf对象

  priv_size: 应用的私有数据大小，位于rte_mbuf struct和data buffer之间

  data_room_size: 每一个mbuf的data buffer大小

  socket_id: 一个socket标识符，这个参数和NUMA(Non-uniform memory access)有关，通常通过rte_socket_id函数来获取
  
  提前申请足够大的可用缓冲区，用在后续的收发包。

- rte_eth_dev_info_get
  
  获取一个NIC的配置信息，检查是否具有所需的功能。检查NIC是否支持MBUF快速释放功能卸载，如果支持就在port_conf中加上此标志位

  ```c
  if (dev_info.tx_offload_capa & RTE_ETH_TX_OFFLOAD_MBUF_FAST_FREE)
  port_conf.txmode.offloads |=
   RTE_ETH_TX_OFFLOAD_MBUF_FAST_FREE;
  ```

- rte_eth_dev_configure
  
  将port_conf应用到port所对应的rx/tx 收发队列上

- rte_eth_dev_adjust_nb_rx_tx_desc
  
  调整每一个rx/tx队列中的rxd/txd(收发描述符)数量

- rte_eth_rx_queue_setup
  
  分配并且设置好rx队列

- rte_eth_tx_queue_setup
  分配并且设置好tx队列
