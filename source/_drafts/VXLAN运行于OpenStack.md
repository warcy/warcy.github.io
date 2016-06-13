---
title: VXLAN运行于OpenStack
tags:
---
已经将VXLAN部署到了实验室的OpenStack平台，下面就来聊聊实际使用中遇到的一些问题吧。

环境相关：

- OpenStack Juno
- 基于Open vSwitch的Neutron方案
- 未启用DVR

<!-- more -->

### 1. 报文分片而无法正常访问
[`tl,dr`]修改方案可直接看本章的**Fix**

搭完VXLAN后进行了简单的ping测试，包括到公网都没问题。但在建立尝试通过浏览器访问`Baidu`时，发现浏览器无法正常访问，只是加载条持续缓慢加载。测试时使用的windows是没有装任何工具的裸镜像，于是先测试了几个网页，发现只有`cn.bing.com`可以不完全加载（如缺少背景图片），但搜索功能正常。
确认防火墙的配置没有问题，OpenStack的安全组也没问题，于是尝试通过在外部（Ubuntu）搭建简单服务器把`Wireshark`传进去：
```bash
$ python -m SimpleHTTPServer
```
发现服务器能收到`GET`但页面依旧无法正常加载，便换了台Windows尝试使用python3：
```cmd
C:\Windows\system32>python3.exe -m http.server
```
装完`Wireshark`后访问`Baidu`的抓包结果如下：

![vxlan_note_get_baidu_with_mtu_1500.jpg](./vxlan_note_get_baidu_with_mtu_1500.jpg)

结果可见，浏览器在不停重传报文，并且提示

> [Reassembly error, protocol TCP: New fragment overlaps old data (retransmission?)]

显然报文被分片了，且无法正常重组。看了端口发现是**443**，才反应过来是通过`HTTPS`访问的百度。
作为处在重力井底的地球人，也能够想起曾几何时看到过一篇运维经验介绍，讲的就是由于VXLAN导致`https`报文分片而无法正常重组的问题。

##### Fix
由于IPv4场景下将会在增加50 Bytes的报文头（包括外部报文及VXLAN字段），所以尝试修改将系统的`MTU`值由默认的**1500**修改为**1450**：

```cmd
C:\Windows\system32>netsh interface ipv4 show subinterfaces

   MTU  MediaSenseState   传入字节  传出字节      接口
------  ---------------  ---------  ---------  -------------
4294967295                1          0    5539223  Loopback Pseudo-Interface 1
  1500                1  139545430   60761452  本地连接

C:\Windows\system32>netsh interface ipv4 set subinterface "本地连接" mtu=1450 st
ore=persistent
确定。  
```

通过浏览器访问测试，问题解决~

##### Question
虽然能正常使用了，但是由于报文头部的增加使得可传输数据减少，肯定是影响网络性能的。那么，反过来增大所有设备的`MTU`应该是能够减少分片带来的性能损失的，但在应用中又会遇到哪些问题呢？
