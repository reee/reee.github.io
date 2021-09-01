---
title: 使用ipset让openwrt上的shadowsocks更智能的重定向流量
date: 2014-07-08 15:55:59
tags:
 - openwrt
 - shadowsocks
 - ipset
---
之前看到有人分享通过dnsmasq的ipset功能简化流量重定向试了下发现很不错。这里分享一下。

使用ipset的主要优势在于直接将所有被污染的域名解析结果交给ipset，不需要动态维护IP列表，在路由上更智能。主要适用于访问和谐站点较少，或者较固定的人群。

本文默认为ar71xx平台，并使用openwrt trunk版本，请按照您的需要自行更改。

本文包括三个部分：
0. 安装相应软件包
2. 配置dnsmasq和ipset
3. 【可选】使用pdnsd获取更优的解析结果。
4. DEBUG

## 一、安装需要的软件包

需要从外部下载以后安装的：
shadowsocks：推荐从http://sourceforge.net/projects/openwrt-dist/files/shadowsocks-libev/ 下载。
·
我的做法是下载下来在本地用nginx建一个http服务器，然后在路由器上去wget。另外的普遍做法是用winscp直接上传到路由器，共参考选择。


开始安装：

	opkg update //安装前必须更新包数据库缓存。
	opkg install ipset iptables-mod-nat-extra
	opkg install dnsmasq-full --force-overwrite //官方提供的dnsmasq-full提供了ipset相关 的功能，需要加后面的参数进行强制覆盖安装。

**某些** 版本ipset安装后会报类似下面的错误：

	kmod: failed to insert /lib/modules/3.10.44/ip_set.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_bitmap_ip.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_bitmap_ipmac.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_bitmap_port.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_hash_ip.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_hash_ipport.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_hash_ipportip.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_hash_ipportnet.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_hash_net.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_hash_netiface.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_hash_netport.ko
	kmod: failed to insert /lib/modules/3.10.44/ip_set_list_set.ko
	kmod: failed to insert /lib/modules/3.10.44/xt_set.ko

需要重新启动以后ipset 才能正常使用。

这里假设你已经下好上文中的dnsmasq和shadowsocks并且进入了存放这两个文件的文件夹。

安装shadowsocks：

	opkg install shadowsocks-libev-polarssl_1.4.5_ar71xx.ipk

新版本不需要处理libpolarssl的问题，老版本shadowsocks编译的时候链接的是libpolarssl.so.5，所以我们需要做一个libpolarssl.so.5软链接出来，否则shadowsocks不能启动且不会有任何报错。

	ln -s /usr/lib/libpolarssl.so  /usr/lib/libpolarssl.so.5

同时你需要自行编辑下/etc/shadowsocks.json，把正确的shadowsocks服务器信息填进去。

另外因为我们需要用到的是ss-redir而不是ss-local。所以请用编辑器打开/etc/init.d/shadowsocks 然后把里面所有的ss-local全部换成ss-redir（一共两处），然后：

	/etc/init.d/shadowsocks enable //让shadowsocks自动启动

考虑到之前ipset的问题。到这里我们先重启一下吧，咱们重启以后见 丷丷。

## 二、配置dnsmasq和ipset

我们先用ipset创建一个set，这里我创建的set名字为setmefree（没有逼格啊没有逼格）,然后将这个set中所有IP均转发到shadowsocks（这里本机的shadowsocks监听的是默认的1080端口）。
建议将下面的命令写入 /etc/rc.local 。每次开机自动运行。

	ipset -N setmefree iphash
	iptables -t nat -A PREROUTING -p tcp -m set --match-set setmefree dst -j REDIRECT --to-port 1080

设置 dnsmasq 对某些域名使用opendns进行解析并且加入setmefree这个set：
为了保持配置文件整洁，建议在 /etc/dnsmasq.conf 最后加入：

	conf-dir=/etc/dnsmasq.d

然后新建目录 /etc/dnsmasq.d ，在里面加入一个 conf，名字任选。譬如 /etc/dnsmasq.d/fuckgfw.conf , 下面是我的文件内容，你可以按自己需要整理自己的：

#Google and Youtube
server=/.google.com/208.67.222.222#443
server=/.google.com.hk/208.67.222.222#443
server=/.gstatic.com/208.67.222.222#443
server=/.ggpht.com/208.67.222.222#443
server=/.google.com/208.67.222.222#443
server=/.google.com.hk/208.67.222.222#443
server=/.gstatic.com/208.67.222.222#443
server=/.ggpht.com/208.67.222.222#443
server=/.googleusercontent.com/208.67.222.222#443
server=/.appspot.com/208.67.222.222#443
server=/.googlecode.com/208.67.222.222#443
server=/.googleapis.com/208.67.222.222#443
server=/.gmail.com/208.67.222.222#443
server=/.google-analytics.com/208.67.222.222#443
server=/.youtube.com/208.67.222.222#443
server=/.googlevideo.com/208.67.222.222#443
server=/.youtube-nocookie.com/208.67.222.222#443
server=/.ytimg.com/208.67.222.222#443
server=/.blogspot.com/208.67.222.222#443
server=/.blogger.com/208.67.222.222#443
server=/.google.co.jp/208.67.222.222#443
server=/.google.co.uk/208.67.222.222#443

#FaceBook
server=/.facebook.com/208.67.222.222#443
server=/.thefacebook.com/208.67.222.222#443
server=/.facebook.net/208.67.222.222#443
server=/.fbcdn.net/208.67.222.222#443
server=/.akamaihd.net/208.67.222.222#443

#Twitter
server=/.twitter.com/208.67.222.222#443
server=/.t.co/208.67.222.222#443
server=/.bitly.com/208.67.222.222#443
server=/.twimg.com/208.67.222.222#443
server=/.tinypic.com/208.67.222.222#443
server=/.yfrog.com/208.67.222.222#443

#Dropbox
server=/.dropbox.com/208.67.222.222#443

#1024
server=/.t66y.com/208.67.222.222#443

#shadowsocks.org
server=/.shadowsocks.org/208.67.222.222#443

#btdigg
server=/.btdigg.org/208.67.222.222#443

#sf.net
server=/.sourceforge.net/208.67.222.222#443

#feedly
server=/.feedly.com/208.67.222.222#443

#github and fastly
server=/.github.com/208.67.222.222#443
server=/.fastly.com/208.67.222.222#443

#wikipedia
server=/.wikipedia.org/208.67.222.222#443
server=/.wikimedia.org/208.67.222.222#443
server=/.wiktionary.org/208.67.222.222#443

#xda
server=/.xda-developers.com/208.67.222.222#443

# Here Comes The ipset

#Google and Youtube
ipset=/.google.com/setmefree
ipset=/.google.com.hk/setmefree
ipset=/.gstatic.com/setmefree
ipset=/.ggpht.com/setmefree
ipset=/.googleusercontent.com/setmefree
ipset=/.appspot.com/setmefree
ipset=/.googlecode.com/setmefree
ipset=/.googleapis.com/setmefree
ipset=/.gmail.com/setmefree
ipset=/.google-analytics.com/setmefree
ipset=/.youtube.com/setmefree
ipset=/.googlevideo.com/setmefree
ipset=/.youtube-nocookie.com/setmefree
ipset=/.ytimg.com/setmefree
ipset=/.blogspot.com/setmefree
ipset=/.blogger.com/setmefree
ipset=/.google.co.jp/setmefree
ipset=/.google.co.uk/setmefree

#FaceBook
ipset=/.facebook.com/setmefree
ipset=/.thefacebook.com/setmefree
ipset=/.facebook.net/setmefree
ipset=/.fbcdn.net/setmefree
ipset=/.akamaihd.net/setmefree

#Twitter
ipset=/.twitter.com/setmefree
ipset=/.t.co/setmefree
ipset=/.bitly.com/setmefree
ipset=/.twimg.com/setmefree
ipset=/.tinypic.com/setmefree
ipset=/.yfrog.com/setmefree

#Dropbox
ipset=/.dropbox.com/setmefree

#1024
ipset=/.t66y.com/setmefree

#shadowsocks.org
ipset=/.shadowsocks.org/setmefree

#btdigg
ipset=/.btdigg.org/setmefree

#sf.net
ipset=/.sourceforge.net/setmefree

#feedly
ipset=/.feedly.com/setmefree

#github&fastly
ipset=/.github.com/setmefree
ipset=/.fastly.com/setmefree

#wikipedia
ipset=/.wikipedia.org/setmefree
ipset=/.wikimedia.org/setmefree
ipset=/.wiktionary.org/setmefree

#xda
ipset=/.xda-developers.com/setmefree

注意：
1. 目前公共DNS有opendns监听非标准端口（443和5353），你也可以到这里查看由opennic提供的DNS：http://meo.ws/dnsrec.php （均监听以下端口54, 443, 1053, 1194, 5353, 8080，27015），当然你也可以自己建DNS服务器。关于DNS优化的讨论请参考下一节。
2. 每条记录都需要跟一条ipset设置，不要忘了。

然后重启 dnsmasq：

	/etc/init.d/dnsmasq restart

或者直接重新启动，顺便测试rc.local中的命令有没有成功运行。

## 三、【可选】使用pdnsd获取更优的解析结果。

注意：通过上面的配置您的openwrt穿墙已经应该正常了，下面仅作优化方面的讨论。

由于opendns是根据来源IP给出结果，导致的结果是直接在路由器上查询返回的是对应中国的IP，以youtube为例：

	dig www.youtube.com @208.67.222.222 -p 443

给出的其中一个结果为：173.194.72.102，在VPS上ping（你应该能理解，所有过shadowsocks的流量事实上是由VPS中转的，所以我们测试的是VPS与该IP的连接情况）约**33ms**。而换用VPS当地的DNS可以将延迟降到**1.45ms**左右。个人认为还是很有意义的。当然其实如果不怕麻烦自建DNS是最好啦。

关于pdnsd的配置请参考[上一篇](http://hong.im/2014/03/16/configure-an-openwrt-based-router-to-use-shadowsocks-and-redirect-foreign-traffic/)。

## 四、DEBUG

1. 通过下面的命令查看set中的IP，这样可以确定解析是否正常，并且查看某网站是否正确的被加到了ipset：

	ipset list setmefree

通过下面的命令可以清理掉set中所有ip。更多的ipset用法请查看*ipset help*

	ipset flush setmefree

2. 以daemon模式运行shadowsocks前建议先试一下确保配置文件无误：

	ss-redir -c /etc/shadowsocks.json

## 更新记录：
2015年8月9日 修正一些说法，更新DNS配置文件。
