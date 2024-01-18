---
layout: post
title: DPDK-初学(18)-ipsec-secgw
date: 2024-01-03 22:11 +0800
categories: [DPDK noob]
tags: [dpdk]
---

终于算忙到了头，可以继续看dpdk了。接着来看这个ipsec-gw，乍一看官网的Sample Guides，这一节基本不涉及怎么组织构建代码的，只给了几个配置文件语法，算了，凑合一样的看。

## 简介

开篇就提到了没实现Internet Key Exchange (IKE)，只提供手动设置安全策略和安全关联，这样也好，我们可以直奔大纲，弯弯绕绕的复杂可以略过。注意这里提到了几个RFC，如果想要完整的实现功能，还是要认真参考如下的文档：RFC4301, RFC4303, RFC3602，RFC2404。

安全策略(Security Policies)采用了ACL规则实现，安全关联(Security Associations)则是通过LPM实现了一张表来进行存储。

由于本示例对端口进行保护/未保护区分，所以其对应的流量被同等的视为入站/出站流量。

IPsec 入站流量路径

- 从指定port读取包
- 辨别包属性为IPv4或ESP(Encapsulating Security Payload)
- 根据ESP中的SPI(Security Parameter Index)执行入站SA查询
- 验证/解密（内联ipsec不需要此步骤）
- 移除ESP和出口IP头（协议卸载打开时不需要此步骤）
- 对解密的包和其他IPv4包通过ACL进行入站SP检查
- 路由
- 将包发送至对应port

IPsec 入站流量路径

- 从指定port读取包
- 对所有IPv4流量通过ACL进行出站SP检查
- 对需要进行IPsec保护的包进行出站SP查询
- 添加ESP和出口IP头（协议卸载打开时不需要此步骤）
- 执行加密/摘要（内联ipsec不需要此步骤）
- 路由
- 将包发送至对应port
