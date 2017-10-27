---
layout: post
title: "不用yum解决 lib/ld linux.so.2: bad ELF interprete问题"
author: 王一刀
description: ""
category: 
tags: []
---
在centos6.9 下安装jdk1.6 时遇到“lib/ld linux.so.2: bad ELF interprete”问题，机器是隔离的不能通过yum直接安装gclib。只能通过rpm包安装。通过以下地址下载glibc-2.12-1.209.el6.i686.rpm和nss-softokn-freebl-3.14.3-23.3.el6_8.i686.rpm
ftp://fr2.rpmfind.net/linux/centos/6.9/os/i386/Packages/glibc-2.12-1.209.el6.i686.rpm

ftp://rpmfind.net/linux/centos/6.9/os/x86_64/Packages/nss-softokn-freebl-3.14.3-23.3.el6_8.i686.rpm

将两个rpm包放到同一目录，执行命令：

rpm -ivh *.rpm

两个rpm一起安装，一个一个安装，因为相互依赖会安装失败。

安装完成后，再执行java -version命令就ok了。