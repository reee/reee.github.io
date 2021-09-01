---
title: 配置一台基于openwrt的无线路由器作为纯无线AP使用
date: 2013-08-05 13:46:38
tags:
- openwrt
- mw4530r
- dumb-ap
- ipv6
---

## 什么是dumb AP以及为什么需要dumb AP

我觉得第一个要解释的是，什么是AP，也就是或者说无线接入点（Wireless Access Point）。 

无线接入点可以简单的看做是通过无线提供的集线器。无线路由器会自己构造一个子网然后所有接入用户共享一个出口IP。而无线接入点只提供接入服务（无线的网线），接入客户的路由服务由接入点的上层设备提供。举个栗子：某正常大学某楼分配的是59.64.32.0/24 的一个子网。如果用无线路由器，那么路由器下所有用户没有公网IP，共享一个公网IP出口（59.64.32.x）。而如果使用无线AP，那么通过无线AP连接的每一个用户都能分配到一个公网IP（59.64.32.x）。

对于教育网用户还有一个好处，因为目前大多数路由器并不支持ipv6，所以如果通过无线路由器的方式没办法获得IPV6地址也就上不了传说中的六维什么的少数教育网福利。而通过无线AP模式，则完全由上层设备路由，所以每个通过无线AP连接的用户都能正常获得IPV6地址（猜测，没有实践过）。

另外一个补充就是，因为无线路由器方式共享一个出口IP，所以说适用于共享上网的场景。而无线AP方式因为每个用户都是独立获取IP，所以适合于每个用户都用自己的账号进行上网的场景。

openwrt上对[Dumb AP](http://wiki.openwrt.org/doc/recipes/dumbap)的定义，事实上就是我们所说的AP。这里就不再多说了。

## 配置openwrt的dumb AP 模式

怎么刷openwrt我就不说了吧……= =

主要参考 [OpenWRT 官方Wiki：Dumb AP](http://wiki.openwrt.org/doc/recipes/dumbap)，但是官方Wiki有少许语焉不明。当然Wiki肯定会被继续完善，不过现在还是先看哥的吧。

### 修改网络配置文件(/etc/config/network)

因为现在纯无线AP反而比无线路由器贵，所以说一般来讲看这个教程的应该是无线路由器的用户（非无线路由器用户请参考openwrt 官方教程），那么我们需要把WAN口和LAN口桥接起来（如果我没理解错的话）：

    config interface lan
            option type     'bridge'
            option ifname   'eth0.1 eth1'  # 将vlan1 和 wan桥接起来
            option proto    'dhcp'         
            
这里需要注意的是，官方教程是把eth0.1和eth1桥接起来，但是事实上很多设备的WAN口不一定是eth1，譬如水星4530r的wan事实上却是eth0.2，所以请自行参考wan口的配置文件，譬如如果wan段如下，那么在lan段就应该是将eth0.1和eth0.2桥接起来，也就是： option ifname   'eth0.1 eth0.2'

    config interface wan
            ...
            option ifname   'eth0.2'
            ...

桥接起来以后，我们需要禁用本身路由器关于wan口的配置：直接把所有wan，wan6的配置文件部分都注释掉，譬如wan6的配置部分：

    #config interface 'wan6'
    #    option proto 'dhcpv6'
    #    option ifname '@wan'

            
### 修改无线配置文件(/etc/config/wireless)

修改完成以后的文件看起来像下面这样：

    config 'wifi-iface'
            option device  'radio0' # 这个不要动
            option network 'lan'  # 酌情修改，可能不需要更改
            option mode    'ap' # 酌情修改，可能不需要更改
            option ssid    'ap_myaccesspoint' # 无线的名字
            option encryption 'psk2'  # 无线加密方式
            option key     'ap_password' # 无线密码 psk2要求8位以上。
            
### 关闭DHCP服务和防火墙

通过uci来关闭DHCP服务：

    uci set dhcp.lan.ignore=1
    uci commit dhcp
    /etc/init.d/dnsmasq restart

关闭防火墙：

    /etc/init.d/firewall disable
    /etc/init.d/firewall stop
    
然后载入新的配置：

    /etc/init.d/network reload

## 其他问题

在配置为无线AP模式以后，如果还需要访问路由器，需要将电脑的IP地址手动设置为：

    ip：192.168.1.x
    mask: 255.255.255.0
    gateway: 192.168.1.1
    
的方式才能正常访问。