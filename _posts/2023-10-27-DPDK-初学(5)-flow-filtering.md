---
categories: [DPDK noob]
tags: [dpdk]
--- 

# flow_filtering example

重复函数不再赘述，只会涉及到新增的部分

## 关键流程

- 初始化EAL
- 对端口进行设置
- 构造flow rule并且应用

## 关键函数

- init_port

  定义了需要的卸载功能，后续通过与网卡能提供的能力做`&`即可。

  这里有个坑，默认的`nr_queues`为5，但是我本机的vm虚拟机网卡(e1000)提供的max_rx_queues为2，注意修改否则会初始化失败。真实的物理网卡为RTL8168H，搜了下没找到硬件手册。

  > vmvare workstation 默认启动的时候，网卡采用的是e1000，而这个配置是可以修改的，具体的解释可以[参考这里](https://kb.vmware.com/s/article/1001805?lang=zh_CN)
  >
  > 我们可以手动的修改vmx文件来启动另一个网络适配器VMXNET3
  >
  > 在vmx文件内，找到你想要修改的网络适配器编号，随后修改即可，例如：ethernet0.virtualDev = "vmxnet3"

  我自己的网卡(e1000)只提供了0x800f的功能，即MULTI_SEGS、VLAN_INSET、IPV4&UDP&TCP的CKSUM。

  修改成vmxnet3之后，提供0x802d，即MULTI_SEGS、TCP_TSO、VLAN_INSET、UDP&TCP的CKSUM。

  ```c
  struct rte_eth_conf port_conf = {
    .txmode = {
     .offloads =
      RTE_ETH_TX_OFFLOAD_VLAN_INSERT |
      RTE_ETH_TX_OFFLOAD_IPV4_CKSUM  |
      RTE_ETH_TX_OFFLOAD_UDP_CKSUM   |
      RTE_ETH_TX_OFFLOAD_TCP_CKSUM   |
      RTE_ETH_TX_OFFLOAD_SCTP_CKSUM  |
      RTE_ETH_TX_OFFLOAD_TCP_TSO,
    },
   };
  ...
  port_conf.txmode.offloads &= dev_info.tx_offload_capa;
  rte_eth_dev_configure(port_id, nr_queues, nr_queues, &port_conf);
  ```

  随即开始初始化队列，这里直接使用了网卡的默认设置。对每一个port的每一个rx_queue，设置他们分配的描述符数量和配置。下面的tx_queue设置同理，只有一个区别。rx队列需要一个mbuf_pool来存储接收的frame data。

  ```c
  rxq_conf = dev_info.default_rxconf;
  rxq_conf.offloads = port_conf.rxmode.offloads;
  for (i = 0; i < nr_queues; i++) {
    ret = rte_eth_rx_queue_setup(port_id, i, 512,
           rte_eth_dev_socket_id(port_id),
           &rxq_conf,
           mbuf_pool);
  }
  
  txq_conf = dev_info.default_txconf;
  txq_conf.offloads = port_conf.txmode.offloads;
  
  for (i = 0; i < nr_queues; i++) {
    ret = rte_eth_tx_queue_setup(port_id, i, 512,
      rte_eth_dev_socket_id(port_id),
      &txq_conf);
  ```

  设置完成后，开启网卡混杂模式并且启动

  ```c
  rte_eth_promiscuous_enable(port_id);
  rte_eth_dev_start(port_id);
  ```

- generate_ipv4_flow

  函数原型如下，从参数能大概推测是为了在port_id指定的nic上对指定rx_queue开启某种规则

  ```c
  struct rte_flow *
  generate_ipv4_flow(uint16_t port_id, uint16_t rx_q,
    uint32_t src_ip, uint32_t src_mask,
    uint32_t dest_ip, uint32_t dest_mask,
    struct rte_flow_error *error)
  {
  ```

  如何设置filter主要是利用下面的几个结构

  ```c
  struct rte_flow_attr attr;
  struct rte_flow_item pattern[MAX_PATTERN_NUM];
  struct rte_flow_action action[MAX_ACTION_NUM];
  ```

  rte_flow_attr: 设定包的流向属性(group、优先级、出入或转发)

  rte_flow_item: 设定过滤的包具体需要满足什么条件

  rte_flow_action: 设定过滤的操作是什么

  其中pattern和action都是一个结构体数组，他们的最后一个成员都是一个结尾标志，如下所示

  ```c
  action[1].type = RTE_FLOW_ACTION_TYPE_END;
  pattern[2].type = RTE_FLOW_ITEM_TYPE_END;
  ```

  其中pattern有一点特殊，需要按照分层的思路来进行每一层的设定，如以下代码所示，从以太帧层到IP层，分别设置。

  ```c
  pattern[0].type = RTE_FLOW_ITEM_TYPE_ETH;
  pattern[1].type = RTE_FLOW_ITEM_TYPE_IPV4;
  pattern[1].spec = &ip_spec;
  pattern[1].mask = &ip_mask;
  ```

  IP层的设置如下，通过`pattern[1].type`可知是需要进行IPV4的配置，所以`pattern[1].spec`是`rte_flow_item_ipv4`结构

  ```c
  memset(&ip_spec, 0, sizeof(struct rte_flow_item_ipv4));
  memset(&ip_mask, 0, sizeof(struct rte_flow_item_ipv4));
  ip_spec.hdr.dst_addr = htonl(dest_ip);
  ip_mask.hdr.dst_addr = dest_mask;
  ip_spec.hdr.src_addr = htonl(src_ip);
  ip_mask.hdr.src_addr = src_mask;
  pattern[1].type = RTE_FLOW_ITEM_TYPE_IPV4;
  pattern[1].spec = &ip_spec;
  pattern[1].mask = &ip_mask;
  ```

  最终检查filter并创建

  ```c
  rte_flow_validate(port_id, &attr, pattern, action, error);
  rte_flow_create(port_id, &attr, pattern, action, error);
  ```

  这里还有个小问题，在e1000模式下，由于不支持MSI-X会报错，需要更换到vmxnet3下

  > MSI-X是一种PCIe的中断模式，更详细的说明可以[参考此链接](https://blog.csdn.net/weixin_38387929/article/details/115325613)

  可惜的是目前切换到vmxnet3后，出现新的报错，如下所示

  > Flow can't be created 1 message: Function not implemented, cause: (nil)

  跟踪了一下源代码然后gdb看了看，问题出在`dev->dev_ops->flow_ops_get`，这玩意是空的，vmxnet3没实现这个地方，导致没法应用。有兴趣的可以跟一下，调用链如下

  ```bash
  0x00007ffff7eb4a10 in rte_flow_ops_get () 
  0x00007ffff7eb5fbf in rte_flow_validate () 
  0x00005555555564b5 in generate_ipv4_flow ()
  0x00005555555554fa in main ()
  ```
