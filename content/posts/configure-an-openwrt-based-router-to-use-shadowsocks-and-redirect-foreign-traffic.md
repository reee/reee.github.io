---
title: 配置一台基于openWRT的路由器使用shadowsocks并智能穿墙
date: 2014-03-16 21:07:48
tags:
 - openwrt
 - shadowsocks
 - pdnsd
---
本文开始写作时中使用的路由器为TP-LINK WR841N（D）V7（with openwrt 12.09 稳定版），到这次更新时已经换为了水星 4530R （with openwrt trunk）

在路由器上使用shadowsocks的**优势**：
0. 效率比较高，在我的不严谨测试中效率比ipsec和pptp都略高。
1. 目前为止比较稳定（较少受到干扰）而且比较安全。
2. 服务器端和客户端的配置都相对来说比较简单，不容易出错。
3. 路由器下面的所有设备都可以**0配置自动穿墙**，你懂的。
4. 相比goagent（基本阵亡？默哀）而言，要求的包都很小。

**劣势**
0. 暂时没有发现。（是的。本来的问题我貌似解决了。）

本文的基本目的是在openwrt上使用pdnsd通过tcp查询规避DNS污染，通过iptables转发到端口的功能转发特定流量给跑在路由器上的shadowsocks来访问某些特定IP段达到一定程度无视某墙的存在的目的。

本文分为三个部分：
0. **相关包的安装和shadowsocks的配置**
1. **pdnsd的配置**
2. **使用iptables对流量进行重定向**

## 在openwrt上安装相关的包并配置shadowsocks

其中shadowsocks我们使用shadowsocks-libev-polarssl（openssl的lib比较大，塞不下……），推荐从这里获取：
http://sourceforge.net/projects/openwrt-dist/files/shadowsocks-libev/

然后在路由器端刷新opkg缓存包并安装shadowsocks：

	opkg update
	opkg install shadowsocks-libev-polarssl_1.4.5_ar71xx.ipk

我们还需要安装额外的包，其中我们使用*pdnsd*（如果后面直接配置dnsmasq转发请求给opendns则不需要）来净化部分国外域名解析，用*iptables-mod-nat-extra*实现iptables流量转发到端口的功能：

	opkg update
	opkg install pdnsd  
	opkg install pdnsd  iptables-mod-nat-extra

我们需要编辑shadowsocks的配置信息 */etc/config/shadowsocks.json*（新版默认的配置文件移动到了/etc/shadowsocks.json 不过在后面启动的时候指定就好了，无影响。）：
格式如下：

	{
		"server":"[服务器IP地址]",
		"server_port":[服务器端口],
		"local_port":[本地端口,稍后iptables会用到],
		"password":"[密码]",
		"timeout":600,
		"method":"[加密方式]"
	}

在12.09上shadowsocks会因为缺少libpolarssl.so.3而无法启动，我们可以使用ln“欺骗”一下shadowsocks：

*注：我目前一切切换到trunk版本，不知道新版是否还存在这个BUG，请自行测试能否启动。*

	ln -s /usr/lib/libpolarssl.so /usr/lib/libpolarssl.so.3


## 配置pdnsd对某些域名进行净化
我采用的基本思路是通过pdnsd使用TCP协议向国外的上级DNS查询而避过DNS污染，然后在本地提供一个1053端口的DNS供dnsmasq使用。如果全局使用pdnsd转发的国外DNS会导致国内某些网站或者服务访问较慢，不推荐。
另外一个思路是使用非标准端口查询，那么就可以不需要配置pdnsd，直接在dnsmasq配置中将相关域名查询请求转发给支持非标准端口的DNS就行了，目前已知的是opendns支持5353端口和443端口。（即在dnsmasq段配置将127.0.0.1#1053 替换为 208.67.222.222#5353 或 208.67.222.222#443 **注意：未测试，仅理论分析**）。

*提醒一下：最近对Google的干扰已经全面升级，单纯解决DNS污染没法愉快的撸youtube了。*

修改pdnsd的配置文件 */etc/pdnsd.conf*：
注意关注中文注释部分，如果复制记得把中文注释删掉。。。其他部分如果您需要，再自行修改：

	# 这一段全局配置需要修改：

	global {
		# debug = on;
		perm_cache=1024;
		cache_dir="/var/pdnsd";
		run_as="nobody";
		server_port = 1053;    # ！！！使用 1053 作为 dns 端口, 默认是 53，一定要修改否则会跟默认dnsmasq冲突
		server_ip = 127.0.0.1;  #我们只需要处理本机转发的DNS查询，所以不需要更改
		status_ctl = on;
		query_method=tcp_only; # ！！！最重要的配置, 只使用 tcp 查询上级 dns
		min_ttl=15m;
		max_ttl=1w;
		timeout=10;
	}

	#……

	# 自行增加下面这一段，pdnsd默认是没有提供上游DNS服务器的（默认配置文件中用各种注释方式把自带的注释掉了）：

	server {
		label= "googledns";           # 这个label随便写
		ip = 8.8.8.8; # 这里为上级 dns 的 ip 地址，要求必须支持TCP查询，相关说明见后文注解
		root_server = on;        # 设置为 on。
		uptest = none;           # 不去检测 dns 是否无效.
	}
			# …… 后面不需要修改的就不贴出来了。

注：DNS地址如果不愿意倒腾可以使用Google Public DNS。如果需要使用其他DNS，请参考：http://public-dns.tk/ ，为了配合后面的重定向，建议使用与VPS地区相同的DNS，譬如我现在使用的服务器在日本，这里的DNS同样使用日本的DNS，一定程度上可以提高连接速度。

启用pdnsd，并设置为开机启动：

	/etc/init.d/pdnsd enable
	/etc/init.d/pdnsd start

设置dnsmasq对特定域名使用本地的pdnsd进行解析：
为了保持配置文件整洁，建议在 */etc/dnsmasq.conf* 最后加入：

	conf-dir=/etc/dnsmasq.d

然后新建目录 */etc/dnsmasq.d*  ，在里面加入一个conf，名字任选。譬如 */etc/dnsmasq.d/fuckgfw.conf* ,下面是我的文件内容，你可以按自己需要整理自己的：

	#Google and Youtube
	server=/.google.com/127.0.0.1#1053
	server=/.google.com.hk/127.0.0.1#1053
	server=/.gstatic.com/127.0.0.1#1053
	server=/.ggpht.com/127.0.0.1#1053
	server=/.googleusercontent.com/127.0.0.1#1053
	server=/.appspot.com/127.0.0.1#1053
	server=/.googlecode.com/127.0.0.1#1053
	server=/.googleapis.com/127.0.0.1#1053
	server=/.gmail.com/127.0.0.1#1053
	server=/.google-analytics.com/127.0.0.1#1053
	server=/.youtube.com/127.0.0.1#1053
	server=/.googlevideo.com/127.0.0.1#1053
	server=/.youtube-nocookie.com/127.0.0.1#1053
	server=/.ytimg.com/127.0.0.1#1053
	server=/.blogspot.com/127.0.0.1#1053
	server=/.blogger.com/127.0.0.1#1053

	#FaceBook
	server=/.facebook.com/127.0.0.1#1053
	server=/.thefacebook.com/127.0.0.1#1053
	server=/.facebook.net/127.0.0.1#1053
	server=/.fbcdn.net/127.0.0.1#1053
	server=/.akamaihd.net/127.0.0.1#1053

	#Twitter
	server=/.twitter.com/127.0.0.1#1053
	server=/.t.co/127.0.0.1#1053
	server=/.bitly.com/127.0.0.1#1053
	server=/.twimg.com/127.0.0.1#1053
	server=/.tinypic.com/127.0.0.1#1053
	server=/.yfrog.com/127.0.0.1#1053

## 使用iptables对流量进行重定向

之前犯了一个错误，采用了默认流量重定向，特定流量（亚太流量）穿透的思路。这样相对来说有很多不必要的流量被重定向到了shadowsocks的服务器端，尤其是在本路由下跑PT的情况。这几天想了下，为什么不只定向某些流量呢。

下面脚本的思路是所有流量默认穿透，只有符合条件的流量才被重定向。这样显得“智能”多了。

您可以直接复制下面的脚本，跟我一样保存为*/usr/bin/ss-black.sh*，注意运行前要给它运行权限：

	chmod +x /usr/bin/ss-black.sh

以下为脚本内容：


	#!/bin/sh

	#create a new chain named SHADOWSOCKS
	iptables -t nat -N SHADOWSOCKS

	#Redirect what you want

	#Google
	iptables -t nat -A SHADOWSOCKS -p tcp -d 74.125.0.0/16 -j REDIRECT --to-ports 1080
	iptables -t nat -A SHADOWSOCKS -p tcp -d 173.194.0.0/16 -j REDIRECT --to-ports 1080

	#Youtube
	iptables -t nat -A SHADOWSOCKS -p tcp -d 208.117.224.0/19 -j REDIRECT --to-ports 1080
	iptables -t nat -A SHADOWSOCKS -p tcp -d 209.85.128.0/17 -j REDIRECT --to-ports 1080

	#Twitter
	iptables -t nat -A SHADOWSOCKS -p tcp -d 199.59.148.0/22 -j REDIRECT --to-ports 1080

	#Shadowsocks.org
	iptables -t nat -A SHADOWSOCKS -p tcp -d 199.27.76.133/32 -j REDIRECT --to-ports 1080

	#1024
	iptables -t nat -A SHADOWSOCKS -p tcp -d 184.154.128.246/32 -j REDIRECT --to-ports 1080

	#Anything else should be ignore
	iptables -t nat -A SHADOWSOCKS -p tcp -j RETURN

	# Apply the rules
	iptables -t nat -A PREROUTING -p tcp -j SHADOWSOCKS

注1：以前的暴力重定向所有流量（除亚洲流量以外）的版本在这里：https://gist.github.com/reee/fe174cfd8985273bc478
注2：如果需要添加你自己需要访问的域名很简单。首先使用dig或者nslookup获取域名对应的**正确IP**，然后借助APNIC的IP WHOIS工具：(http://wq.apnic.net/apnic-bin/whois.pl) 可以轻松的获得大部分IP段。以facebook为例：

dig获取的正确IP为：173.252.110.27。
通过APNIC查询到173.252.64.0/18均属于facebook。则添加

	iptables -t nat -A SHADOWSOCKS -p tcp -d 173.252.64.0/18 -j REDIRECT --to-ports 1080

到上面脚本 倒数第二条前面就可以了。

然后就是见证奇迹的时刻：

	#设置路由：
	/usr/bin/ss-black.sh
	#启动shadowsocks
	/usr/bin/ss-redir -c /etc/config/shadowsocks.json &


## 其他问题

查看iptables的NAT表来检查路由表是否已经成功加载：

	iptables -t nat --list

停止服务器：

	killall ss-redir  # 关闭shadowsocks。
	/etc/init.d/firewall restart # 清除流量重定向配置。

参考文章：
[openwrt 上通过 pdnsd 和 dnsmasq 解决 dns 污染](https://wido.me/sunteya/use-openwrt-resolve-gfw-dns-spoofing)
[https://github.com/haohaolee/shadowsocks-openwrt](https://github.com/haohaolee/shadowsocks-openwrt)

更新历史：
0. 2014-05-25 小幅修正某些说法，话说乃们在twitter上收藏那么多吓到我了。
1. 2014-06-12 更改路由方式，现在科学多了。
