---
title: VXLAN隧道技术
tags:
---

VXLAN（Virtual eXtensible LAN，可扩展虚拟局域网络）是基于IP网络、采用MAC over UDP封装形式的二层 VPN技术。其本质与GRE类似，通过进一步封装实现隧道效果。

<!-- more -->

### 实验

本实验环境与前文[GRE隧道技术](http://warcy.github.io/2016/04/29/GRE%E9%9A%A7%E9%81%93%E6%8A%80%E6%9C%AF/)基本相同，实验拓扑、虚拟机信息等详细数据可见前文。区别在与隧道搭建时，使用VXLAN而不是GRE，包括流表设置命令也与之前相同，当然OVS在实际封装报文时设置的是VXLAN报文中的VNI字段。
VM1:

```bash
$ ovs-vsctl add-port br0 vtep1 -- set interface vtep1 type=vxlan \
option:remote_ip=192.168.5.238 option:local=192.168.5.23 \
option:in_key=flow option:out_key=flow
$ ovs-ofctl add-flow br0 'priority=10,in_port=local,actions=set_tunnel:10,output=2'
```

VM2:

```bash
$ ovs-vsctl add-port br0 vtep1 -- set interface vtep1 type=vxlan \
option:remote_ip=192.168.5.23 option:local=192.168.5.238 \
option:in_key=flow option:out_key=flow
$ ovs-ofctl add-flow br0 'priority=10,in_port=local,actions=set_tunnel:10,output=2'
```

在VM1 ping VM2，通过Wireshark抓取通过VM1 eth0的ICMP报文。
p.s. wireshark可能无法正常decode报文的VXLAN header，可通过在对应报文右键`Decode As...`手动设置。

![vxlan-experiment-wireshark.png](./vxlan-experiment-wireshark.png)


### 结论


### 参考资料
