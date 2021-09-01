---
title: "Qbittorrent vs Transmission vs Rtorrent"
date: 2019-08-16T12:33:19+08:00
tags: ["bittorrent client", qbittorrent, rtorrent, transmission]
---
一直在来回倒腾这三个客户端，所以很早就想写个总结。然后终于写出来了- 。-

## 基础对比

* 资源占用以及下载效率： rtorrent > qbittorrent > transmission。

在种子数比较少（大概1000以下）的情况下，rtorrent和qbittorrent的表现差别不是很大。根据其他人测试的数据，在种子数特别多的情况下，rtorrent能跟qbittorrent拉开差距。transmission的效率则最低，在我当前150mbps的环境下，rtorrent和qbittorrent能迅速占满带宽。而transmission即使占满带宽流量曲线也并不平滑，有比较大的波动。 

* 配置难度：rtorrent >> transmission > qbittorrent。

rtorrent之所以难配置的主要原因在于后端（rtorrent）和前端( [ruTorrent](https://github.com/Novik/ruTorrent) 或 [Flood](https://github.com/Flood-UI/flood) )分别都需要配置。比较好的是，从0.9.7开始rtorrent开始支持demon mode，相对之前版本要更好配置一些（老版本需要把rtorrent跑在tmux或者screen里面）。transmission由单一配置文件控制，直接编辑文件就能完成配置。而qbittorrent的绝大多数配置可以通过webUI完成，难度相对来说比transmission还要更低一些。但是——qbittorrent默认没有提供systemd的service文件，需要自己写一个，不过难度不大。另外需要注意的是Archlinux在装rtorrent和qbittorrent（仅远程使用的时候，安装qbittorrent-nox就可以）的时候，由于这两个软件本身对daemon模式支持不好，所以软件包本身不会添加相关用户，需要自己添加。

* 稳定性

从我使用的情况来看，qbittorrent相对来说稳定性要差一些，transmission稳定性尚可，rtorrent理论上来讲应该是最稳定。不过考虑到qbittorrent开发非常活跃，不稳定也是预料之中的事情。

## 其他方面

* WebUI相关。这个比好直接给结论，分开说吧。

单论特性的丰富程度，rtorrent(与ruTorrent配合，后面简称rutorrent)要远远优于后两者。rutorrent支持媒体文件自动截图，自动解压，查看媒体信息等等。你甚至可以直接通过rutorrent把种子文件下载到本机。可以看出来，rutorrent的主要设计宗旨就是为了在服务器上大量的做种而设计的。
另外，rtorrent的两个主流webUI（rutorrent和flood）对移动端支持都很不错。不过比较可惜的是，不管是rutorrent和flood都不支持批量添加种子文件。

qbittorrent的webUI特性也比较丰富。你甚至可以通过qbittorrent的webUI完成ddns更新（虽然我不知道有什么意义……）。qbittorrent WebUI的最大问题是不（tai）好（chou）看，感觉像上一代的网页，毫无设计感。另外qbittorrent的移动版页面也相当不好用。qbittorrent的webUI的另一个问题在于不支持对种子内容进行排序，也没有层级结构，在筛选大包里面的文件的时候相当不便。

transmission比较特殊。它自带的webUI功能极少，但是可以改为 [transmission-web-control](https://github.com/ronggang/transmission-web-control) 。transmission-web-control 的特性基本跟qbittorrent相当，但是比qbittorrent的漂亮很多。不过不管是transmission自带的webUI还是transmission-web-control本身都不支持https，而且如果用nginx代理的话，则transmission就不能设置密码，只能通过nginx控制。

另外，transmission可以通过第三方的跨平台远程控制客户端 [transgui](https://github.com/transmission-remote-gui/transgui) 控制。transgui支持文件关联，你完全可以像操作本机上的transmission一样控制远程服务器上的transmission，这点上看，完爆前面两个。

* 其他方面

三个客户端里面只有rtorrent不支持在未完成文件后面添加后缀，如果配合Plex等流媒体服务，会带来一些困扰。另外，qbittorrent针对大包里面没有选中的文件，会放入一个隐藏文件夹中，这里特别好评。顺便说下，其实这三个软件在安卓上都有第三方客户端可以支持，但是只有Transmission的客户端功能最全。

另外，这三个包如果都最小安装的话，qbittorrent的依赖最多，安装后所占体积最大，让人感觉不够优雅。

最后，之所以没有讨论deluge，主要是国内一些主流的PT站点并不支持deluge，且deluge也已经停止开发好长时间了。而没有讨论utorrent的主要原因是，我目前的主要平台是Linux，目前为止utorrent还没有Linux下的客户端。

综上，下面列一个表吧：

----

|-| rtorrent | qbittorrent| transmission|
|:---- | :---- |:---- | :---- |
|资源占用及下载效率|最优|优|略差|
|配置难度|最难|较简单|较简单|
|稳定性|稳定|较不稳定|稳定|
|移动端支持（WebUI）|良好|较差|尚可|
|批量添加种子（WebUI）|不支持|支持|支持|
|HTTPS支持（WebUI）|通过Web服务器支持|原生支持、也可通过Web服务器支持|如果使用Web服务器代理，则UI本身不能设有密码|
|WebUI综合评价|优秀|较差|原生WebUI较差，transmission-web-control 优秀|
|种子包内容控制|分文件夹，可按属性排序|不分文件夹，不可排序|分文件夹，可按属性排序|
|为未完成文件添加后缀|不支持|支持|支持|
|Android客户端|功能少|功能少|功能完备|
|Windows客户端|无|无|功能完备|

----

事实上，如果喜欢按挑三拣四的模式下大包或者对速度和资源占用要求没那么高，Transmission是最佳选择。如果有大批量发布种子的需求，那么rtorrent则是最佳选择。如果追求速度，不希望太操心，那可以试试qbittorrent。

以上，本文部分内容靠回忆书写，如有遗漏的地方，请留言指出。