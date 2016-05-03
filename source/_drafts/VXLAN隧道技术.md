---
title: VXLAN隧道技术
tags:
---

VXLAN（Virtual eXtensible LAN，可扩展虚拟局域网络）是基于IP网络、采用MAC over UDP封装形式的二层 VPN技术。其本质与GRE类似，通过进一步封装实现隧道效果。

在学习VXLAN的报文结构等一系列具体知识之前，让我们看看RFC7348里是怎么说的：`Virtual eXtensible Local Area Network (VXLAN): A Framework for Overlaying Virtualized Layer 2 Networks over Layer 3 Networks`。
显然，相对于GRE是一项报文封装技术，VXLAN是一种overlay网络框架。这也就意味着他不仅仅是一种隧道技术，他将给出overlay网络下的一系列处理方案，比如以组播承载租户的ARP广播。

<!-- more -->

我们先来看看VXLAN的报文结构：

![vxlan-header-rfc7348.png](./vxlan-header-rfc7348.png)

通过`VXLAN Header`字段中的`VXLAN Network Identifier（VNI）`实现overlay网络间报文的区分。当指定VNI时，8 bits的Flag标志位中的I标志位需置1，该段中设下的保留位R的值**必须**置0，而Header中其余的保留部分也**必须**置0。此外，VXLAN要求UDP目的端口号应该使用默认值4789，源端口号建议使用49152-65535（详见RFC文档）。同时，VXLAN也支持使用IPv6作为外层封装。
前文提到VXLAN提供的是overlay网络，除了以隧道方式传输数据，还有以下处理方式：
- VTEP作为隧道的Endpoint在提供报文的封装及解封装之外，通过本地Mapping的方式减少了学习造成的隧道流量，此时VTEP可以任务实现了部分路由器的功能。
- 对于未知的用户，仍需要有效的学习机制以减少广播对网络造成的负载。VXLAN使用组播来限制ARP广播仅在overlay子网内被处理，且组播能够有效控制泛洪规模。
- 由于是overlay网络，在与公网连接时，需要通过VXLAN gateway与非VXLAN设备进行通信。

### 实验

本实验环境与前文[GRE隧道技术](http://warcy.github.io/2016/04/29/GRE%E9%9A%A7%E9%81%93%E6%8A%80%E6%9C%AF/)基本相同，实验拓扑、虚拟机信息等详细数据可见前文。区别在与隧道搭建时，使用VXLAN而不是GRE，包括流表设置命令也与之前相同，当然OVS在实际封装报文时设置的是VXLAN报文中的VNI字段。且本实验只对VXLAN报文封装进行了验证性试验。
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

在VM1 ping VM2，通过Wireshark在VM1 eth0抓取相关报文，结果如下列图例所示。
p.s. wireshark可能无法正常decode报文的VXLAN header，可通过在对应报文右键`Decode As...`手动设置。

VTEP为通过组播实现广播控制，所以需要使用IGMP协议加入组播组。如Figure 1所示，VTEP 192.168.5.23以EXCLUDE模式加入组播组。但是本实验中未配置组播控制器，所以并未启用组播功能。

![vxlan-experiment-wireshark-igmp.png](./vxlan-experiment-wireshark-igmp.png)

*Figure 1: IGMP packet of VTEP*

如Figure 2所示，VTEP对于转发网络可视为正常用户，在同一子网内直接进行交换。相对的，在VTEP间通过ARP学习保证之间的连通性后，VTEP对用户层面的ARP进行封装，并发送至对端VTEP。由于未启用组播功能，故外层封装报文仍未点对点的通信。

![vxlan-experiment-wireshark-vtep-arp.png](./vxlan-experiment-wireshark-vtep-arp.png)

*Figure 2: ARP packet of VTEP*

![vxlan-experiment-wireshark-vm-arp.png](./vxlan-experiment-wireshark-vm-arp.png)

*Figure 3: ARP packet of VM*

在VM间完成ARP学习后，ICMP报文的封装方式与ARP报文相同。其中`Flag=0x08`，即标志位中的I位置1，同时附加了自定义的VNI值为10，与流表设置内容相符。

![vxlan-experiment-wireshark-icmp.png](./vxlan-experiment-wireshark-icmp.png)

*Figure 4: ICMP packet of VM*

### 结论

#### 与GRE技术的比较

### 参考资料

- https://tools.ietf.org/pdf/rfc7348
- http://www.arista.com/assets/data/pdf/Whitepapers/VXLAN_Scaling_Data_Center_Designs.pdf
- http://bingotree.cn/?p=654
- https://ring0.me/2014/02/network-virtualization-techniques/
- http://www.sdnlab.com/15820.html
- http://www.sdnlab.com/16169.html

