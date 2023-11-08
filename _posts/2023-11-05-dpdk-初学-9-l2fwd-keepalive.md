---
layout: post
title: DPDK 初学(9)-L2fwd Keepalive
date: 2023-11-05 21:10 +0800
categories: [DPDK noob]
tags: [dpdk]
---
KeepAlive检测常被用在检测无声退出的情况，一般都是主进程main保持一定频率的想worker发送消息并获取回复，如果达到设定阈值没有回复就通告上级并处理。

## 关键函数

前面都是些见过好多次的常规操作了，不再赘述。

### rte_eth_tx_buffer_set_err_callback

原型如下。该函数用于为一个pkt缓冲区buffer注册一个callback，当尝试通过一个ethdev发送出全部的缓冲pkt时但是没有完全成功发送时，会调用这个callback。并且这个回调只会由 `rte_eth_tx_buffer()` 和`rte_eth_tx_buffer_flush()`触发。默认的回调函数会释放pkt回到mempool，如果要自定义callback，要记得做这个操作。

```c
int rte_eth_tx_buffer_set_err_callback(struct rte_eth_dev_tx_buffer *buffer, buffer_tx_error_fn callback, void *userdata)
```

### rte_timer*系列

初始化timer库

```c
rte_timer_subsystem_init();
rte_timer_init(&stats_timer);
```

### rte_keepalive_shm_create

创建一个共享内存并映射出来。通过如下三个函数，获取了一个main使用的`rte_keepalive_shm`结构。随后初始化这个结构

```c
fd = shm_open(RTE_KEEPALIVE_SHM_NAME,
  O_CREAT | O_TRUNC | O_RDWR, 0666);
ftruncate(fd, sizeof(struct rte_keepalive_shm)) != 0
ka_shm = (struct rte_keepalive_shm *) mmap(
   0, sizeof(struct rte_keepalive_shm),
   PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

memset(ka_shm, 0, sizeof(struct rte_keepalive_shm));
sem_init(&ka_shm->core_died, 1, 0)
for (idx_core = 0;
  idx_core < RTE_KEEPALIVE_MAXCORES;
  idx_core++) {
 ka_shm->core_state[idx_core] =
  RTE_KA_STATE_UNUSED;
 ka_shm->core_last_seen_times[idx_core] = 0;
}
```

### rte_keepalive_create

将上面获取的`ka_shm`绑定到`rte_global_keepalive_info`结构中来，并且注册好一个回调`dead_core`。

```c
rte_global_keepalive_info =
 rte_keepalive_create(&dead_core, ka_shm);
```

### rte_timer_reset

函数原型：

```c
int rte_timer_reset(struct rte_timer *tim, uint64_t ticks, enum rte_timer_type type, unsigned int tim_lcore, rte_timer_cb_t fct, void *arg)
```

> tim：注册好的timer
>
> ticks：时间间隔长度
>
> type：定时器命中后timer的下一个状态，PERIODICAL-重回pending、SIGLE-stop状态
>
> time_lcore：回调执行的lcore
>
> fct：回调函数
>
> arg：回调函数的参数

通过这个函数，注册一个定时器，每到设定时间触发一次keepalive检测。

```c
rte_timer_reset(&hb_timer,
 (check_period * rte_get_timer_hz()) / 1000,
 PERIODICAL,
 rte_lcore_id(),
 (void(*)(struct rte_timer*, void*))
 &rte_keepalive_dispatch_pings,
 rte_global_keepalive_info
 ) != 0 )
```

## 流程

来看一下输出，分别是`l2fwd-keepalive`和`ka-agent`。流程还是很清晰的，main这边ping没有接受到返回就通过共享内存上报至ka-agent，然后重启dead的lcore。

```bash
EAL: Core MIA. Last seen 10ms ago.
EAL: Core died. Last seen 20ms ago.
Dead core 1 - restarting..
L2FWD: entering main loop on lcore 1
L2FWD:  -- lcoreid=1 portid=0
L2FWD:  -- lcoreid=1 portid=1
EAL: Core MIA. Last seen 10ms ago.
EAL: Core died. Last seen 20ms ago.
Dead core 1 - restarting..
```

```bash
sudo ./build/ka-agent
1 dead cores: 1,
1 dead cores: 1,
1 dead cores: 1,
```
