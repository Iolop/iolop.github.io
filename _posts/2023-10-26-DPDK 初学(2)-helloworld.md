---
categories: [DPDK noob]
tags: [dpdk]
---

# helloworld example

## 关键函数

### rte_eal_init
  
  这个函数就是入口初始化所有环境，初始化所有EAL参数，具体的可选参数可以参考[DPDK的官方文档](https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html)

### rte_eal_remote_launch
  
  函数原型

  ```c
  int rte_eal_remote_launch(lcore_function_t *f, void *arg, unsigned int worker_id)
  ```

  该函数经常和`RTE_LCORE_FOREACH_WORKER(lcore_id)`搭配使用，用以从各逻辑核启动`f`

## 关键流程

- 初始化EAL(Environment Abstract Layer)
- 轮询worker，对每个逻辑核远程绑定一个入口`lcore_hello`
