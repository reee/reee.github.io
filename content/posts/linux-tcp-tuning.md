---
title: 高流量大并发Linux TCP 性能调优
date: 2013-04-20 00:23:43
tags:
 - linux
---
呃……标题比较隐晦。其实主要是手里面的跑openvpn服务器。因为并没有明文禁p2p（哎……想想那么多流量好像不跑点p2p也跑不完），所以造成有的时候如果有比较多人跑BT的话，会造成VPN速度急剧下降。

本文参考文章为：

[优化Linux下的内核TCP参数来提高服务器负载能力](http://blog.renhao.org/2010/07/setup-linux-kernel-tcp-settings/)

[Linux Tuning](http://fasterdata.es.net/host-tuning/linux/)

本文所面对的情况为：

1. 高并发数
2. 高延迟高丢包（典型的美国服务器）

值得注意的是，因为openvz的VPS权限比较低，能够修改的地方比较少，所以使用openvz的VPS作VPN服务器是非常不推荐的。

我们通过修改 /etc/sysctl.conf 来达到调整的目的，注意修改完以后记得使用：
	
	sysctl -p
	
来使修改生效。

首先，针对高并发数，我们需要提高一些linux的默认限制:

	fs.file-max = 51200
	#提高整个系统的文件限制
	net.ipv4.tcp_syncookies = 1
	#表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；
	net.ipv4.tcp_tw_reuse = 1
	#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
	net.ipv4.tcp_tw_recycle = 0
	#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭；
	#为了对NAT设备更友好，建议设置为0。
	net.ipv4.tcp_fin_timeout = 30
	#修改系統默认的 TIMEOUT 时间。
	net.ipv4.tcp_keepalive_time = 1200
	#表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为20分钟。
	net.ipv4.ip_local_port_range = 10000 65000 #表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为10000到65000。（注意：这里不要将最低值设的太低，否则可能会占用掉正常的端口！）
	net.ipv4.tcp_max_syn_backlog = 8192
	#表示SYN队列的长度，默认为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数。
	net.ipv4.tcp_max_tw_buckets = 5000
	#表示系统同时保持TIME_WAIT的最大数量，如果超过这个数字，TIME_WAIT将立刻被清除并打印警告信息。

	#额外的，对于内核版本新于**3.7.1**的，我们可以开启tcp_fastopen：
	net.ipv4.tcp_fastopen = 3
	
其次，针对大流量高丢包高延迟的情况，我们通过增大缓存来提高TCP性能，自己看E文注释吧……感觉我翻译出来各种味道不对 = =：

	# increase TCP max buffer size settable using setsockopt()
	net.core.rmem_max = 67108864 
	net.core.wmem_max = 67108864 
	# increase Linux autotuning TCP buffer limit
	net.ipv4.tcp_rmem = 4096 87380 67108864
	net.ipv4.tcp_wmem = 4096 65536 67108864
	# increase the length of the processor input queue
	net.core.netdev_max_backlog = 250000
	# recommended for hosts with jumbo frames enabled
	net.ipv4.tcp_mtu_probing=1

这里面涉及到一个TCP拥塞算法的问题，你可以用下面的命令查看本机提供的拥塞算法控制模块：

	sysctl net.ipv4.tcp_available_congestion_control

如果没有下文提到的htcp,hybla算法，你可以尝试通过modprobe启用模块：
	
	/sbin/modprobe tcp_htcp
	/sbin/modprobe tcp_hybla
	
对于几种算法的分析，详情可以参考下：[TCP拥塞控制算法 优缺点 适用环境 性能分析](http://blog.csdn.net/zhangskd/article/details/6715751)，但是这里面没有涉及到专门为卫星通讯设计的拥塞控制算法：[Hybla](http://en.wikipedia.org/wiki/Hybla)。根据各位大神的实验，我们发现Hybla算法恰好是最适合美国服务器的TCP拥塞算法，而对于日本服务器，个人想当然的认为htcp算法应该可以比默认的cubic算法达到更好的效果。但是因为htcp算法恰好没有编入我们所使用的VPS中，所以没办法测试。

	#设置TCP拥塞算法为 hybla
	net.ipv4.tcp_congestion_control=hybla
	
更新：2014-06-22 修改TW快速回收的问题以更好的兼容移动设备。