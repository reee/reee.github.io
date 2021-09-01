---
title: 在centOS 7上面，从源码包重新编译软件包
date: 2015-12-27 11:32:36
tags:
- Centos 7
- 编译
- 源码包
---
众所周知，ocserv需要重新编译才能支持比较详细的路由设置。这里简单总结一下如何在centOS 7 上面，从srpm出发，重新编译RPM包。

基本参考的资料就是官方的构建环境搭建指南和RPM重构指南：
https://wiki.centos.org/HowTos/SetupRpmBuildEnvironment
https://wiki.centos.org/HowTos/RebuildSRPM

## 搭建构建环境

1. 首先安装必备的软件包：

    yum install rpm-build redhat-rpm-config make gcc

2. 建立相应的用户，并切换到相应用户：

    user add mockbuild
    su mockbuild

需要注意的是，官方建议不使用root账户来进行编译，而看起来EPEL默认的编译用户是mockbuild，上一步建立任意非root账户其实也可以但是可能会在安装srpm的时候遇到类似于下面的警告：

    warning: user mockbuild does not exist - using root
    warning: group mockbuild does not exist - using root

从个人实践的情况来看，似乎上面的警告也不会影响最后的编译，所以看你自己咯。

3. 建立相应的文件夹：

    mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}

4. 设定RPM build变量：

    echo '%_topdir %(echo $HOME)/rpmbuild' > ~/.rpmmacros

## 取得源代码包进行重编译

以epel为例，可以从这里取得对应的SRPM：
https://dl.fedoraproject.org/pub/epel/7/SRPMS/

将对应的SRPM下载下来，然后使用rpm进行安装：

    rpm -i /path/to/the/rpm

相关的文件会被解压到上一步新建的文件夹。

以ocserv为例，我们需要更改源码包中的vpn.h。所以需要先从SOURCES目录下的ocserv-0.10.8.tar.xz解压出源码，修改vpn.h以后打包回去：

    cd SOURCES
    tar -Jxf ocserv-0.10.8.tar.xz
    rm ocserv-0.10.8.tar.xz
    tar -Jcf ocserv-0.10.8.tar.xz ocserv-0.10.8/

然后重新编译rpm包：

    rpmbuild -ba SPECS/ocserv.spec

然后在RPMS下就会生成对应的rpm包了。