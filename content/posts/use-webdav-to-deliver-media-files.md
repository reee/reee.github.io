---
title: "使用WebDAV来实现远程媒体播放"
date: 2019-08-18T21:38:27+08:00
tags: [webdav, plex, emby, nginx, "Solid Explorer", RaiDrive]
---

Plex无疑是媒体服务器中的佼佼者。作为一个商业方案，Plex比Emby及其衍生物要更加优雅一些。但是在经历过Plex某个版本会莫名其妙占满单个核心，以及不久前干脆占满所有核心导致整个服务器宕机以后。我决定借WebDAV来实现媒体服务器，而事实证明，效果非常好。

## 使用WebDAV的优势

利用WebDAV的精髓在于WebDAV可以被挂载为一个本地磁盘，下面的优势其实都是基于这一事实。

* 无需转码，因此服务器负担极低

WebDAV可以被映射为一个本地文件夹，调用本地播放器进行播放。因此不管是h.265、wmv或是其他各种奇葩编码方案，通过一个现代的本地播放器都能完美的支持播放，不存在需要转码的情况。而plex大体上是浏览器那一套解码器，绝大多数情况需要被转码为h.264才能开始播放，这就要求服务器本身具有很强的解码能力。使用Plex播放一个非h.264编码的文件，经历的流程为 服务端解码，再编码-客户端再解码播放。而如果使用WebDAV的话，直接在客户端对文件进行解码即可。

使用WebDAV，服务器的负担只有传输数据这一个任务，那么之前的矿渣蜗牛星际的J1900或者各位手中“价值几万”的斐讯N1就都可以轻轻松松串流几路1080P（前提是网络给力）。相对应的，（对于非h.264编码的视频）Plex要么需要GPU支持，要么需要至少i3级别的CPU才能达到实时播放的能力（参考这里： [Plex Media Server Requirements](https://support.plex.tv/articles/200375666-plex-media-server-requirements/) ），区别不可谓不大。

这里需要提到另外一个类似WebDAV的文件分享解决方案：[FileBrowser](https://github.com/filebrowser/filebrowser) 。由于FileBrowser也是通过浏览器提供的服务，因此也存在只要浏览器不能解码，那么该文件就没法播放的问题，局限比较大。

* 更快开始播放

Plex在播放开始前需要客户端判断先是否支持该视频，以及判断网络情况，然后再决定是否转码以及转码的码率。如果确实需要转码，还需要一定时间等待一定时间段的视频完成转码。导致Plex需要一定的时间才能开始播放。WebDAV则只需要从服务器端读取数据就可以开始解码播放。因此WebDAV开始播放需要等待的时间要少得多。

**特别提醒**：考虑到整个视频码率不是一定的，以及网络情况可能会有波动，因此建议通过WebDAV播放视频的，将播放器的缓存视频的时间长度设长一点，整个播放过程会流畅一些。从我的体验来讲，将播放器缓存时间设置为1min，基本上就不会感觉到有卡顿问题。

* 可以配合本地的一些其他服务，增强体验

比如 [Media Preview](http://www.babelsoft.net/products/mediapreview.htm) 增强缩略图体验。或者使用 [Everything](https://www.voidtools.com/zh-cn/) 对文件进行快速检索。反正挂载在本地，我就是可以为所欲为。。。

## 使用WebDAV相对Plex的不足

* 媒体组织方式的缺陷

Plex毕竟是一个媒体中心，对媒体信息尤其是电影电视剧信息的刮削能力相当强，会自动为文件配上海报简介无需人工干预。这点是WebDAV这个方案无法比的。不过因为我本身对这块要求不高，所以这并不算什么致命的问题。

* 文件体积过大以及对高码率文件支持效果欠佳

一部正常的720P电影大概是4G左右，如果直接在移动网络下播放，就是足足4G的流量消耗。另外，一个正常的4K视频码率在50mbps左右，在当前国内的网络环境下，不大可能有人能支持这么高码率视频的实时播放。而Plex因为可以在服务器端进行转码，就可以规避这两个问题。不过1080P视频普遍在10mbps左右，要正常播放并不是什么难事，所以这个缺陷也不算什么问题。

## 服务器端配置

* 相关软件的安装

现在一般习惯使用nginx来作为web服务器。通过nginx来提供WebDAV服务，需要加载dav-ext这个第三方模块。下面仅讨论使用nginx提供WebDAV服务的情况。

Ubuntu中，nginx-full这个包自带了dav-ext这个扩展，因此直接安装这个包即可。而arch的话，需要安装 nginx-mainline 和 nginx-mainline-mod-dav-ext。

* WebDAV服务的配置

配置部分可以参考 [ArchWiki: WebDAV](https://wiki.archlinux.org/index.php/WebDAV) 。配置过程分为两步：1. 是载入模块。需要注意的是，Ubuntu载入模块的方式略有不同，请参考Ubuntu的相关配置。2.是书写配置文件。由于我们需要面向整个互联网提供服务，因此配置文件需要改变一下，去掉allow，deny字段，然后加上认证部分。我的配置文件供参考：

    location /dav {
        root   /srv/http;

        auth_basic "Auth Require";
        auth_basic_user_file /etc/nginx/.htpasswd;

        dav_methods PUT DELETE MKCOL COPY MOVE;
        dav_ext_methods PROPFIND OPTIONS;

        dav_access user:rw group:rw all:r;
        client_max_body_size 0;
        create_full_put_path on;
        client_body_temp_path /srv/client-temp;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        charset utf-8,gbk;
    }
	
配置文件中，是将/srv/http/dav目录分享出来，请按自己情况修改。client_body_temp_path也需要自己根据实际情况修改。另外.htpasswd文件需要自行使用htpasswd或者其他方式生成。可以参考DO的这篇文章： [How To Set Up Password Authentication with Nginx on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-nginx-on-ubuntu-14-04)
	
## 客户端部分
	
Android上推荐使用Solid Explorer（收费软件，14天试用）挂载WebDAV文件夹。其他的Total Commander安装相关扩展以后可以正常挂载，但是无法调用本地播放器播放（事实上我猜Solid Explorer有一些小魔法在里面）。Ghost Commander界面实在一言难尽且挂载体验不佳。
	
Windows自身支持WebDAV，但是体验十分糟糕，这里推荐使用 [RaiDrive](https://www.raidrive.com/)（有收费版但事实上免费版已经够用了）挂载。由于相关配置都十分简单，这里就不多说了。