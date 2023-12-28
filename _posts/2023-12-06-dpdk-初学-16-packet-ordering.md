---
layout: post
title: DPDK-初学(16)-packet-ordering
date: 2023-12-06 21:55 +0800
categories: [DPDK noob]
tags: [dpdk]
---

将一个流中的包进行重排序。该示例至少需要三个lcore。

- Rx-core：负责从NIC接收数据包，通过软队列发送给worker
- Worker-core：从软队列接收包，并设置好转发port。
- Tx-core：从worker通过软队列获取包，将乱序的包插入到重排序缓冲区，并取出排序后的包进行转发。

## 简要流程

前面都是基础的初始化操作，注意一下通过`rte_ring_create`建立了两个软件队列，用在后面的数据包传递上。然后用`rte_reorder_create`建立了一个重排序缓冲区。
随即开始分别启动rx-thread、worker-thread、tx-thread。

rx-thread：通过`rte_eth_rx_burst`接收包，随即通过`rte_reorder_seqn`为这些包标记顺序。注意这里实际上应该是利用了mbuf中的`dynfield1`字段来标记，`seqn`是一个不断自增的数据。随后通过软队列发送给worker-thread。

```c
for (i = 0; i < nb_rx_pkts; )
    *rte_reorder_seqn(pkts[i++]) = seqn++;
...
...
__rte_experimental
static inline rte_reorder_seqn_t *
rte_reorder_seqn(struct rte_mbuf *mbuf)
{
    return RTE_MBUF_DYNFIELD(mbuf, rte_reorder_seqn_dynfield_offset,
        rte_reorder_seqn_t *);
}
...
...
ret = rte_ring_enqueue_burst(ring_out,
    (void *)pkts, nb_rx_pkts, NULL);
```

worker-thread：从软队列中取出包，修改port，然后通过软队列给tx-thread。

```c
burst_size = rte_ring_dequeue_burst(ring_in,
    (void *)burst_buffer, MAX_PKTS_BURST, NULL);
for (i = 0; i < burst_size;)
    burst_buffer[i++]->port ^= xor_val;
ret = rte_ring_enqueue_burst(ring_out, (void *)burst_buffer,
    burst_size, NULL);
```

tx-thread：从软队列中获取数据包，多一个操作，`rte_reorder_insert`。然后通过`rte_reorder_drain`获取到之前插入的mbuf重排序后的结果，随即将它们发送。

```c
nb_dq_mbufs = rte_ring_dequeue_burst(args->ring_in,
    (void *)mbufs, MAX_PKTS_BURST, NULL);
for (i = 0; i < nb_dq_mbufs; i++) {
    /* send dequeued mbufs for reordering */
    ret = rte_reorder_insert(args->buffer, mbufs[i]);
...
}
dret = rte_reorder_drain(args->buffer, rombufs, MAX_PKTS_BURST);
for (i = 0; i < dret; i++) {
    ...
    sent = rte_eth_tx_buffer(outp1, 0, outbuf, rombufs[i]);
    ...
}
```

以上就是一个简单的重排序应用。
