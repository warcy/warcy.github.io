---
title: VXLAN在OpenStack中的应用
tags:
---
OpenStack作为时下最火的IaaS框架之一，其网络也支持使用VXLAN搭建overlay网络。比如在其文档[Scenario: Classic with Open vSwitch](http://docs.openstack.org/mitaka/networking-guide/scenario-classic-ovs.html)一文中，简要描述了在VXLAN网络模型下的流量分布以及各节点的功能。

简要来说，即所有节点默认部署软件VTEP以实现VXLAN。当然，为减少服务器封包解包带来的性能损失，可将VTEP部署至ToR。

<!-- more -->

### OpenStack在VXLAN的改进
对于基于组播的VXLAN方案，在上一篇笔记中也有提到在实际部署是有问题的。具体上来说有以下几点：
- Flood低效
- 组播部署困难
- 混合的数据和控制平面

对于这一系列问题，社区包括一些商用公司也给出了部分解决方案。

#### BaGPipe

#### L2 Population+ARP Responder

### 第三方改进方案
- Nuage Networks
- OpenContrail

此外部分商业厂商，如Cisco、BigSwitch给出的硬件方案。

### 参考资料
- http://www.sdnlab.com/16816.html