---
title: GRE隧道技术
date: 2016-04-29 00:26:08
tags:
---


GRE（Generic Routing Encapsulation，通用路由协议封装）协议是通过对已有报文进一步封装实现在另一网络层的传输，常见应用为IP-over-IP，是一种隧道（Tunnel）技术。

Tunnel是一个虚拟的点对点连接，提供了一条通路使封装的数据报文能够在这个通路上传输,并且在一个Tunnel的两端分别对数据报进行封装及解封装。图中的路由器A和B作为Tunnel Endpoint实现对报文的封装工作，而在应用中为便于部署也有通过软件方式实现的封装技术。

{% asset_img gre-tunnel.png %}

\* 本图片来源于[GRE技术介绍](http://www.h3c.com.cn/Products___Technology/Technology/Security_Encrypt/Other_technology/Technology_recommend/200805/605933_30003_0.htm)

<!-- more -->

上图IP network在对GRE进行转发时，将根据报文外层的`Delivery Header`部分进行转发，被封装在内部的`Payload packet`可视为GRE报文的数据字段。通用GRE报文格式如下：

![gre-packet](./gre-packet.png)

其中`GRE Header`字段在**RFC2784**标准中定义如下，后在**RFC2890**中提出改进方案。在**RFC2890**中，当设置Key标志位（报文中的2 bit）为1时，增加的Key Field（32 bits）可用于同一个隧道内的多个私有网络/流量的隔离。

#### RFC2784

![gre-header-rfc2784](./gre-header-rfc2784.png)

#### RFC2890

![gre-header-rf2890](./gre-header-rfc2890.png)

### OpenStack neutron网络中的应用

在OpenStack neutron网络下，GRE作为一种隧道技术被应用于实例跨节点通信时的租户流量隔离。不同租户的报文在经由OVS Tunnel Bridge转发时，将设置不同的Key，以用于在对端GRE Endpoint实现对解封装报文的转发。

而在使用隧道（包括GRE和VXLAN）作为租户隔离技术时，节点间的转发设备将无法区分节点内具体租户以及实例的流量，即在转发时一个计算节点即一个“用户”。所以对于ToR来说，MAC表项由所有实例的MAC信息减少到Tunnel Endpoint（物理机节点）的MAC信息，以避免大规模数据中心应用时ToR表项空间不足以及路由器表项空间不足等问题。

![scenario-classic-ovs-flowew2](./scenario-classic-ovs-flowew2.png)
\* 本图片来源于[Scenario: Classic with Open vSwitch](http://docs.openstack.org/mitaka/networking-guide/scenario-classic-ovs.html)

### 实验

在OpenStack neutron模型中，有通过Open vSwitch实现的GRE的方案，所以本实验尝试通过OVS复现该功能，并测试转发网络对GRE报文的处理。

#### 实验环境

实验环境与[搭建基于Open vSwitch的GRE隧道实验](http://www.sdnlab.com/5889.html)一文类似，通过2台虚拟机实现模拟Tunnel Endpoint以及用户，另外在虚拟机间的转发使用OVS替代网桥。

实验拓扑：

![experiment-topology](./experiment-topology.png)

设备信息：

| 设备 | IP | MAC |
|--------|--------|
| VM1-br0 | 192.168.4.10 | 9e:7c:b1:71:f8:49 |
| VM1-br1 | 192.168.5.23 | 08:00:27:54:e2:e4 |
| VM2-br0 | 192.168.4.11 | 82:2a:2c:98:c1:4b |
| VM2-br1 | 192.168.5.238 | 08:00:27:f4:36:8d |
| VM1-eth0 | Null | 08:00:27:54:e2:e4 |
| VM2-eth0 | Null | 08:00:27:f4:36:8d |

#### 实验流程

利用Mininet默认命令在物理机快速创建OVS s1，并将虚拟机分别加入该OVS的**s1-eth1**和**s1-eth2**（默认生成的h1/h2在10.0.0.0/8网段，不会对实验造成干扰）。为便于调试该OVS，使用Ryu作为控制器。

```bash
$ sudo mn --controller=remote
```

虚拟机的网桥设置与原文相同，分别配置网桥br0和br1的IP地址及网关信息。
VM1：

```bash
$ ovs-vsctl add-br br0
$ ovs-vsctl add-br br1
$ ifconfig br1 192.168.5.23/24 up
$ route add default gw 192.168.5.1 dev br1
$ ifconfig br0 192.168.4.10/24 up
```

VM2:

```bash
$ ovs-vsctl add-br br0
$ ovs-vsctl add-br br1
$ ifconfig br1 192.168.5.238/24 up
$ route add default gw 192.168.5.1 dev br1
$ ifconfig br0 192.168.4.11/24 up
```

在建立GRE隧道时，添加`in_key`和`out_key`参数，以便通过流表来设置Tunnel id（即GRE报文的Key Field）。对应所加流表项在封装报文时设置Tunnel id为10，并从2端口发出（即本实验中eth0的端口号，可通过`ovs-ofctl show br0`命令查看）。
VM1:

```bash
$ ovs-vsctl add-port br0 gre1 -- set interface gre1 type=gre \
option:remote_ip=192.168.5.238 option:local=192.168.5.23 \
option:in_key=flow option:out_key=flow
$ ovs-ofctl add-flow br0 'priority=10,in_port=local,actions=set_tunnel:10,output=2'
```

VM2:

```bash
$ ovs-vsctl add-port br0 gre1 -- set interface gre1 type=gre \
option:remote_ip=192.168.5.23 option:local=192.168.5.238 \
option:in_key=flow option:out_key=flow
$ ovs-ofctl add-flow br0 'priority=10,in_port=local,actions=set_tunnel:10,output=2'
```

为实现通信，通过控制器在物理机的s1下发以下流表，实现对Tunnel Endpoint的ARP处理及GRE报文的转发。为便于在控制器端分析报文，对于未匹配报文统一将其发送至控制器。

```bash
$ ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=14.398s, table=0, n_packets=1, n_bytes=140, idle_age=3, priority=10,ip,in_port=1,nw_proto=47 actions=resubmit(,10)
 cookie=0x0, duration=14.398s, table=0, n_packets=1, n_bytes=140, idle_age=3, priority=10,ip,in_port=2,nw_proto=47 actions=resubmit(,20)
 cookie=0x0, duration=14.398s, table=0, n_packets=0, n_bytes=0, idle_age=14, priority=5,arp actions=ALL
 cookie=0x0, duration=14.398s, table=0, n_packets=0, n_bytes=0, idle_age=14, priority=0 actions=CONTROLLER:65535
 cookie=0x0, duration=14.398s, table=10, n_packets=1, n_bytes=140, idle_age=3, priority=10,dl_src=08:00:27:54:e2:e4 actions=output:2
 cookie=0x10, duration=14.398s, table=10, n_packets=0, n_bytes=0, idle_age=14, priority=0 actions=CONTROLLER:65535
 cookie=0x0, duration=14.398s, table=20, n_packets=1, n_bytes=140, idle_age=3, priority=10,dl_src=08:00:27:f4:36:8d actions=output:1
 cookie=0x10, duration=14.398s, table=20, n_packets=0, n_bytes=0, idle_age=14, priority=0 actions=CONTROLLER:65535
```

在VM1 ping VM2，观察VM1的ARP信息，并通过Wireshark抓取通过VM1 eth0的ICMP报文。
```bash
$ ping 192.168.4.11 -c1
PING 192.168.4.11 (192.168.4.11) 56(84) bytes of data.
64 bytes from 192.168.4.11: icmp_seq=1 ttl=64 time=0.977 ms

--- 192.168.4.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.977/0.977/0.977/0.000 ms
root@ovs-VirtualBox:/home/ovs# arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.5.238            ether   08:00:27:f4:36:8d   C                     br1
192.168.4.11             ether   82:2a:2c:98:c1:4b   C                     br0
```

![gre-experiment-wireshark](./gre-experiment-wireshark.png)

由抓包结果可见，相较于原实验，GRE Header的`Flags and Version`值为0x2000而非0x0，即Key标志位被置1。同时Key Field被流表设为0xa，与GRE桥br0配置的流表相吻合。
若隧道未设置流表，则发送的报文Key标志位仍置1，但Key Field为0x0。

### 扩展实验
#### 实验一

在配置GRE隧道时，可通过以下命令指定Tunnel id，此时封装的GRE报文Key Field将被设置为0xa。由于指定了Key值，在若VM2未配置key使用缺省0x0或者值的内容不为0xa时，Endpoint将会丢弃该报文。

```bash
$ ovs-vsctl add-port br0 gre1 -- set interface gre1 type=gre \
option:remote_ip=192.168.5.23 option:local=192.168.5.238 \
option:key=10
```

#### 实验二

根据报文的Protocol字段值为47可以确定报文为GRE报文，便想到OVS能否进一步匹配GRE的Tunnel id，修改s1的部分流表如下：

```bash
$ ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=13.235s, table=10, n_packets=0, n_bytes=0, idle_age=13, priority=10,tun_id=0xa,dl_src=08:00:27:54:e2:e4 actions=output:2
 cookie=0x0, duration=13.235s, table=20, n_packets=0, n_bytes=0, idle_age=13, priority=10,tun_id=0xa,dl_src=08:00:27:f4:36:8d actions=output:1
```

虽然流表可见，然而并没有什么用处，该条流表并不会匹配任何报文。因为tun_id的匹配是建立在GRE Endpoint能解封装GRE报文的情况下的，此时s1的端口未配置GRE隧道，将不会对报文内的GRE Header做处理。

### 结论

- OpenStack neutron中使用GRE实现租户间流量隔离，默认是在计算节点的OVS Tunnel Bridge上实现，为减轻CPU负担可以使用边缘的交换机实现offload，比如H3C的[混合方案](http://www.h3c.com.cn/About_H3C/Company_Publication/IP_Lh/2014/07/Home/Catalog/201501/852548_30008_0.htm)。
- GRE是一种隧道技术，既然是隧道那么就必须存在Endpoint实现封装与解封装，payload对于“正常”的网络设备应该是透明的，实验二中OVS显然无法正常匹配Tunnel id。
- 但是因为本质上来说GRE并没有对payload加密，被截获的报文还是能够被解封装的，所以可通过IPSec等技术进行加密。

### 参考资料

- https://tools.ietf.org/html/rfc1701
- https://tools.ietf.org/html/rfc2784
- https://tools.ietf.org/html/rfc2890
- https://github.com/yeasy/openstack_understand_Neutron/tree/master/gre_mode
- https://github.com/openstack/neutron/blob/dd4f1253c951d78a5b497680dfb31317ba469a58/neutron/plugins/ml2/drivers/openvswitch/agent/openflow/native/br_tun.py
- http://www.sdnlab.com/5889.html
- http://openvswitch.org/pipermail/dev/2013-February/025591.html
