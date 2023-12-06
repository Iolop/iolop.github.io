---
layout: post
title: DPDK-初学(14)-service-cores
date: 2023-12-04 22:07 +0800
categories: [DPDK noob]
tags: [dpdk]
---
示例非常清晰易懂。这个例子展示了，怎么把一个service给注册起来并且map到对应的lcore中去。

## 基本流程

通过三个函数来注册并且启用service

```c
rte_service_component_register
rte_service_component_runstate_set
rte_service_runstate_set
```

随后，在`apply_profile`中，通过完全对称的三个函数，来将lcore加入到service可用core上，并且根据设置来判断，这个核要不要被service占用。

```c
rte_service_lcore_add
rte_service_lcore_start
rte_service_map_lcore_set
```

总的来说，service在这个示例上，被实现成了一种和lcore解耦开的样子。他们能够独立的设置自己的状态，然后再配置这些service能够再哪些核上运行。再来看dpdk中的解释，这一段就比较容易理解了，将硬件的问题抽象出来，可以不再关心硬件。二是可以任意配置service-function被多个cpu运行起来。
>The service cores infrastructure provides DPDK with two main features. The first is to abstract away hardware differences: the service core can CPU cycles to a software fallback implementation, allowing the application to be abstracted from the difference in HW / SW availability. The second feature is a flexible method of registering functions to be run, allowing the running of the functions to be scaled across multiple CPUs.
