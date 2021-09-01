---
layout: post
title: "在wr841n上刷openwrt并使用openvpn自动VPN"
date: 2012-11-03 17:27
comments: true
tags:
 - OpenWRT
 - OpenVPN
---
 刚出来工作就买了一个WR841ND。最开始其实没咋倒腾，因为dd-wrt虽然功能多但是用到的不多，反而配置太多看着头晕，而开始倒腾openwrt的时候又觉得openwrt的界面太难看---不想动。所以其实一直是官方固件。不过最近由于斯巴达的即将召开，导致Google搜索基本上来讲完全瘫痪，而恰好又买了一个香港的VPS。所以决定倒腾倒腾让路由器直接连VPN然后智能路由。  
 硬件信息：TP-Link TL-WR841ND v7.1
 
 **注意：一下操作要求一定的linux基础和一定动手能力，所以请务必谋定而动。**  
 
##首先是刷上openwrt
 
刷新之前请注意：目前trunk中的openwrt默认没有启用web界面，虽然最新稳定版openwrt发布已经支持wr841n(d)了，但是我还是喜欢trunk……因为trunk里面没有我用不到的web界面……又可以节省好多K的空间装其他东西……

从官方固件刷到openwrt是很简单的。直接从trunk下载[适用于wr841nd的固件](http://downloads.openwrt.org/snapshots/trunk/ar71xx/openwrt-ar71xx-generic-tl-wr841nd-v7-squashfs-factory.bin) 然后通过官方固件的web界面中升级固件更新就行了。  
 现在你最好telnet过去修改密码，这样会禁用telnet而启用ssh。当然如果你是在太懒这个其实也无所谓啦。  
 
##配置网络
 
 现在你需要配置好网络，请参考：[openwrt 官方wiki关于网络配置的说明](http://wiki.openwrt.org/doc/howto/internet.connection) 。如果是 **静态IP** ，那么示例如下：  
 
	uci set network.wan.proto=static //表示使用静态IP。
	uci set network.wan.ipaddr=192.168.100.111 //我使用的是联通光猫。开了路由功能。所以可以直接静态IP。
	uci set network.wan.netmask=255.255.255.0
	uci set network.wan.gateway=192.168.100.1
	uci set network.wan.dns='8.8.8.8 8.8.4.4'
	uci commit network
	ifup wan

当然其实直接编辑/etc/config/network是一条更快的捷径,以下以 **pppoe拨号** 为例，与静态IP不同，我们一定要注意设定openwrt不通过ppp获得运营商的DNS，否则挂上VPN以后几乎一定会因为运营商DNS延迟更低而选择使用运营商DNS从而不能避免DNS污染，我们只修改wan段：

	config interface 'wan'
		option ifname 'eth1'
		option proto 'pppoe'  //表示使用pppoe拨号
		option username 'dsl11xxxxxxxx'   //pppoe帐号
		option password 'xxxxxx'  //pppoe密码
		option peerdns '0'   //表示不从运营商获取DNS，很重要！
		option dns '8.8.8.8 8.8.4.4'

配置好网络以后你可以试一下ping 8.8.8.8来测试一下。

##安装配置openvpn

你需要先更新下opkg的缓存：

	opkg update
然后安装openvpn，随着openwrt里面的openvpn更新到2.3，trunk里面openvpn已经被重新打包为两个包：openvpn-openssl和openvpn-polarssl。使用openvpn-polarssl要求你的服务器端也配置使用polarssl目前不大现实，同时openvpn-openssl因为依赖600多K的libopenssl直接安装肯定是要把路由器撑爆的。所以……如果铁了心要上，只能自己动手编译固件。对于ROM充裕的用户直接安装吧

	opkg install openvpn-openssl
安装好以后，你需要把你的配置文件，ca以及相关的认证文件放到/etc/openvpn下。放过去的方法：你要么在电脑上打开，然后粘贴到服务器上，要么像我一样在笔记本上开一个ftp然后在从路由器上wget。装完openvpn以后的剩余空间只有400多K了：

	Filesystem                Size      Used Available Use% Mounted on
	rootfs                    1.4M      1.1M    328.0K  78% /
所以请注意，一定要在配置文件中将日志设置为不记录，即：
	verb 0
否则路由器很快就会因为文件系统被撑爆而拒绝服务。

## 修改配置文件，配置路由行为

默认情况下，一般的openvpn提供商会路由所有流量这样会导致本地访问比较缓慢而且对VPN服务器造成比较大的压力。所以我们需要设置下路由让国内的流量不经过路由器。
现在提供比较成熟的路由方案的，一个是[chnroute](https://code.google.com/p/chnroutes/)。不过现在chnroute路由条目已经达到了3500多条。我从chnroute改出来的配置文件就达到了150多K。而当我开始连接以后添加路由表占了太多资源和时间，直接把我的路由器搞挂了。  
所以chnroute放到这么小个路由器上显然是不行的。  
鉴于此我看了下[autoddvpn](https://code.google.com/p/autoddvpn/) 的代码，发现只有150多行。相对chnroute简直是太小了，可惜我完全没有看明白它的脚本在干什么，而且没有人把它port到openwrt。所以我也就看看算了。
但是很幸运的是，google到chnroute之前衍生出了一个亚洲路由表，小很多。下载链接在这里：[routes-亚洲](http://pan.baidu.com/share/link?shareid=119635&uk=1409674396)。  
下载以后把内容增加到openvpn配置文件末尾。然后连接：

	/usr/sbin/openvpn --cd /etc/openvpn --config xxx.conf 
你可以先运行看看然后Ctrl+X关掉以后加上--daemon让它在后台运行。openvpn连接成功以后修改下转发规则，当然你也可以把下面这段丢到/etc/firewall.user里面去这样每次开机都会自动运行（而且对没有连接VPN时候的正常上网也没啥影响）：

	iptables -I FORWARD -o br-lan -j ACCEPT #允许br-lan端口流量被转发
	iptables -I FORWARD -o tun0 -j ACCEPT #允许tun0端口流量被转发
	iptables -t nat -I POSTROUTING -o tun0 -j MASQUERADE #tun0出口的流量SNAT出去
连接成功以后在路由上建议清空dns解析缓存：

	/etc/init.d/dnsmasq restart

电脑上建议先冲掉之前的dns解析：

	ipconfig /flushdns
	
经过我的测试，发现youtube，twitter，facebook无压力。完工。！  

参考链接：  
[OpenWRT设置Openvpn并自动智能翻墙](http://hugozhu.appspot.com/2011/07/17/opwrt%E8%AE%BE%E7%BD%AEopenvpn%E5%B9%B6%E8%87%AA%E5%8A%A8%E7%BF%BB%E5%A2%99/)。

update：2013-12-22 修正了对openvpn-polarssl的错误认识。