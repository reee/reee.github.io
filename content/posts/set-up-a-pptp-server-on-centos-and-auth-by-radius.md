---
layout: post
title: "在CentOS上搭建pptp服务器并使用radius进行账户认证"
date: 2013-01-04 11:57
comments: true
tags:
 - Linux
 - PPTP
 - freeradius
---
感觉主流的VPN服务器里面pptp应该是最简单的了。总结下吧。本来没想搞这个的。可惜ipv4下的openvpn根本就是被我伟大的防火墙秒杀。所以先配个用着吧。

系统环境：centos 6.3 x86_64 xen虚拟机（我会炫耀说是樱花的么 = =）

## 安装相关软件
首先需要安装ppp和pptp以及radius认证库radiusclient-ng，ppp和radiusclient-ng在官方源里面有，所以直接安装就行了：

	yum -y install ppp radiusclient-ng
	
这里需要说一下的就是大部分教程都要求安装radiusclient而非radiusclient-ng。但是如果到radiusclient-ng的官方页面你会发现radiusclient-ng其实就是radiusclient的后续版本，最新的radiusclient已经是05年的了，虽然似乎不影响使用，但是真的是太老太老了。

pptpd直接安装sourceforge上提供的二进制版本就好了，这里是64位版本，如果你要安装32位版本把尾巴的x86_64改成i686就好了，如果是centos5的话，自己改咯：

	rpm -ivh http://poptop.sourceforge.net/yum/stable/packages/pptpd-1.3.4-2.el6.x86_64.rpm

## 修改相关配置
备份是个好习惯，不过其实这里没太大必要备份了= =：

	mv /etc/ppp/options.pptpd /etc/ppp/options.pptpd.bak
	vi /etc/ppp/options.pptpd
	
在option.pptpd里面写入相关的配置：

	name pptpd
	refuse-pap
	refuse-chap
	refuse-mschap
	require-mschap-v2
	require-mppe-128
	proxyarp
	lock
	nobsdcomp
	novj
	novjccomp
	nologfd
	idle 2592000
	ms-dns 8.8.8.8
	ms-dns 8.8.4.4
	#下面是radius认证需要的库以及配置文件
	#同样的，如果您是32位系统应该将lib64改为lib
	plugin /usr/lib64/pppd/2.4.5/radius.so
	plugin /usr/lib64/pppd/2.4.5/radattr.so
	radius-config-file /etc/radiusclient-ng/radiusclient.conf
	
修改/etc/pptpd.conf

	mv /etc/pptpd.conf /etc/pptpd.conf.bak
	vi /etc/pptpd.conf
	
其实需要改动的很少，其他还有一些譬如限制同时在线人数的，请自己阅读原始的pptpd.conf：

	option /etc/ppp/options.pptpd
	#下面是指定服务器使用的IP
	localip 192.168.12.1
	#下面是将要分配给客户端的IP段
	remoteip 192.168.12.2-245
	
使用iptables转发pptp的流量：

	iptables -t nat -A POSTROUTING -o eth0 -s 192.168.12.0/24 -j MASQUERADE
	
修改radiusclient配置：
	vi /etc/radiusclient-ng/radiusclient.conf
其中需要指定radius server的地址，修改如下两行，这里假设你已经搭好了radius server而且IP为1.2.3.4：

	authserver 1.2.3.4
	acctserver 1.2.3.4
	
如果你的radius server使用的是默认的端口，在上面其实就不用指定认证的端口了。另外还需要[注释掉bindaddr * （在77行）。否则pppd会报错](http://blog.chinaunix.net/uid-9509185-id-3060838.html) 。
接着你需要修改相关的radius server认证共享密钥，这里假设你已经更新了radius server上的设置允许了这台服务器的连接，仍然以1.2.3.4为例，且假设你的共享密钥为 some-pass
	vi /etc/radiusclient-ng/servers
在尾巴加上：

	1.2.3.4		some-pass
	
然后修改相关的dictionary文件，radiusclient的dictionary文件在/etc/radiusclient /下但是radiusclient-ng的文件到了/usr/share/radiusclient-ng/ 目录：

	vi /usr/share/radiusclient-ng/dictionary
	
在尾巴加上：

	INCLUDE /usr/share/radiusclient-ng/dictionary.merit
	INCLUDE /usr/share/radiusclient-ng/dictionary.microsoft
	
默认安装没有dictionary.microsoft，而一般教程让到[freeradius wiki](http://wiki.freeradius.org/PopTop)下载，但是相关链接已经挂掉了，用我从[某个地方](http://www.members.optushome.com.au/~wskwok/poptop_ads_howto_8.htm)挖出来的这个dictionary.microsoft吧：

	wget http://some-file.googlecode.com/git/pptp/dictionary.microsoft
	mv dictionary.microsoft /usr/share/radiusclient-ng/
	
然后启动pptpd server，并且加到服务器自启动列表中：

	/etc/init.d/pptpd start
	chkconfig --add pptpd
完工。
注释掉一些可能引起报错的地方，原因目前未知：

	 sed -i 's/logwtmp/\#logwtmp/g' /etc/pptpd.conf
	 sed -i 's/radius_deadtime/\#radius_deadtime/g' /etc/radiusclient-ng/radiusclient.conf
	 sed -i 's/bindaddr/\#bindaddr/g' /etc/radiusclient-ng/radiusclient.conf

## 其他问题
如果有其他问题，请直接查看/var/log/message
	cat /var/log/message
然后配合强大的Google吧。