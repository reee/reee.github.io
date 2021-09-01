---
title: 一台基于APU的HTPC
date: 2013-06-13 09:30:25
tags:
 - HTPC
 - APU
 - PotPlayer
 - XBMC
 - foobar2000
---
## 一、概述

本文主要定位于如何获得一个可工作的基于APU的HTPC，本文使用的硬件组装于2012年9月16，所以对各位而言，有可能硬件信息已经比较过时。

## 二、硬件

使用到的硬件如下：

*  主板：华擎 双核APU平台E350M1（集成AMD E-350 双核1.6G CPU，集成ATI Radeon HD6310显卡） 价格：￥505（含快递15块～）
*  机箱：酷冷至尊 魔方 MINI-ITX小机箱USB3.0拉丝铝面板 价格：￥299
*  内存：南亚易胜（elixir）DDR3 1333 4G 台式机内存 价格：￥99（已大幅涨价）
*  硬盘1：镁光 32G 固态硬盘 SSD 1.8寸半高 价格：￥188
*  硬盘2：希捷2TB 2T硬盘64M 7200转 ST2000DM001台式机硬盘 价格：￥660（已大幅降价）
*  遥控器：挥挥鼠M3 价格：￥158（已降价）
*  键盘鼠标：Logitech 罗技 MK240 无线键鼠套装 价格：￥99
*  电源：本地电脑城买的最便宜的电源 价格：￥99

合计：2107。败笔感觉在那小块固态硬盘上……感觉没太大必要。同时在有遥控器的情况下，无线键鼠套装基本就是在角落积灰，也没太大必要。这套系统的性能瓶颈应该在CPU上……真是……伤神呢。不过那个遥控器还真是满好用的，推荐。

个人认为优势在于：

* 系统自由度更高一些，可以自主选择Linux或者Windows。
* 采用魔方机箱的好处在于盘位较多，可以自由升级。而且散热什么的也比较给力。
* 采用APU一体板的好处在于，在能耗相对较低的情况（没有实测，但是整机功耗应该在30W左右）下，用比较低的价格做到了HTPC和NAS的合体。

优势其实一句话说到底就是自由度吧……DIY的，谁不是想自由度高一些呢……

相对来说，因为全部采用新硬件，可能性价比并不是那么突出。当然肯定是可以通过旧硬件重用或者淘宝二手CPU什么的做到更高的性价比。不过一是图一个新，另一个是新硬件相对来说功耗更低一点（没有考虑过Intel平台，求不喷），要更稳定一些……

## 三、软件解决方案

### 1、系统：Windows 8

在一开始尝试过蛮多系统（好吧，就是win 7，Win 8 和一堆linux发行版），其实如果主要应用是NAS，Linux应该是一个很不错的选择，但是一旦涉及到高清播放……我只能说Linux完完全全是渣渣啊。不过随着[AMD发布开源 UVD 支持](http://linuxtoy.org/archives/amd-releases-open-source-uvd-support.html)，Linux下面的高清播放支持倒是值得期待（表示我还没有尝试。）

然后win7 和win 8之间，因为个人认为win 8的欢迎屏幕更适合我的电视……加上win 8确实有一些驱动，性能上的提升，所以最终确定下来就是win 8了。

### 2、视频播放：PotPlayer+XBMC

这里采用[XBMC](http://xbmc.org/download/)解决在线视频播放的问题。为了播放国内视频网站的视频，XBMC中文插件库是一定要安装的，请自行参考：[xbmc-addons-chinese](https://code.google.com/p/xbmc-addons-chinese/)。除此之外，XBMC还真没啥好说的……不过优酷，搜获超清什么的，在我的51寸大电视上那是相当爽啊。

然后采用[Potplayer](http://pan.baidu.com/share/link?shareid=536881&uk=2636462713)解决本地高清视频播放的问题。这里多说一句，本来PotPlayer国内有一个非常不错的交流论坛：(http://potplayer.5d6d.net/) ，可惜随着5D6D官方宣布不再提供免费论坛服务，实在是前途未卜呢。

推荐的滤镜组合为：

* 源分离滤镜： [LAV Splitter](https://code.google.com/p/lavfilters/)
* 视频解码滤镜： [LAV Video Decoder](https://code.google.com/p/lavfilters/)
* 视频渲染： 高清视频 EVR(CP)，低清晰度视频 [MadVR](http://forum.doom9.org/showthread.php?t=146228)

源分离滤镜没啥好说的，其实PotPlayer目前自带的源分离滤镜已经能够提供比较好的拖放性能了，这个随意吧。

至于视频解码滤镜LAV Video Decoder的理由，主要是相对其他视频滤镜来说，LAV Video Decoder 一是开源免费，二是开发活跃，三是支持各种硬解方式。从我用下来的情况看，也比较稳定，没什么严重BUG。

而视频渲染其实如果性能足够当然是完全推荐MadVR一路走到黑的。不过因为组的HTPC性能太差……采用MadVR打开高清（720P以上）视频需要等待较长时间而且容易假死。所以只能退而求其次使用EVR(CP)。不过说起来，高清的视频上MadVR区别肉眼倒是满难看出来，但是如果视频本身质量比较差，采用MadVR可以很明显的观察到视频锐度和色彩还原度的大幅提升。所以MadVR非常推荐啦。

### 3、音频播放：foobar2000

基本上Windows平台上音频播放的选择是唯一的吧：[foobar2000](http://www.foobar2000.org/)，采用到的插件有：

* DTS解码支持：[DTS Decoder](http://www.foobar2000.org/components/view/foo_input_dts)
* WASAPI 输出支持：[WASAPI output support](http://www.foobar2000.org/components/view/foo_out_wasapi)
* ape解码支持：[Monkey's Audio Decoder](http://www.foobar2000.org/components/view/foo_input_monkey)
* HTTP远程控制，因为主要是配合Android上的一个程序foobarcon使用，所以直接从foobarcon主页下载最新的：[foobar2000 http control](https://sites.google.com/site/foobarcon/)

WASAPI输出可以感觉到有明显的音质提升，不过我那小破音箱……不过因为WASAPI是独占的，所以在打开其他需要声音输出的程序时，必须将foobar2000完全停止或者退出。

而http control除了可以使用手机上的foobarcon控制以外，还可以从其他电脑的浏览器控制，请自行查看项目主页：[foo-httpcontrol](https://code.google.com/p/foo-httpcontrol/)。说起来foobarcon真心不错，还可以在手机端显示歌词而且基本上都能抓取……一个人寂寞的在家K歌什么的毫无鸭梨了。

### 4、BT下载：utorrent

Windows下BT下载的选择也比较唯一：[utorrent](http://www.utorrent.com/)。

需要注意的是：

* utorrent 3去广告：将 设置 - 高级 中的offers.sponsored_torrent_offer_enabled改为false重启
* 启用远程控制：展开 设置 - 高级 - 远程 。勾选启用网页界面，在下面输入用户名，密码。你可以在下面定义端口和下载目录。自己倒腾吧。