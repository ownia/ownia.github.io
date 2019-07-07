---
title: RaspberryPi3B的使用问题
date: 2018-07-19
tags:
comments: true
---

树莓派官网raspberrypi.org下载Raspbian stretch with desktop，通过Win32DiskImager烧录进16gSD卡中，出现如下错误：
<!--more-->
{% asset_img raspberry.jpg %}
在官网上找到类似问题，https://raspberrypi.stackexchange.com/questions/17023/panic-vfs-unable-to-mount-root-fs-on-unknown-block179-2， 但是没有得到解决。
通过百度得知可以使用`sudo fsck -f -v -r /dev/sdb2`之类的fsck命令解决。在deepin上使用此命令后错误没有得到解决。
尝试使用官网的noobs进行安装。
使用DiskGenius对已经烧录过的SD卡进行重新分区并格式化，烧录noobs系统。
安装时一直循环resizing fat partition。
解决方法：在使用DiskGenius之后，再使用电脑自带的格式化再次格式化，文件系统为FAT32，分配单元大小为16kb。最终成功安装。
使用noobs安装则不能开启expand Filesystem。
安装之后开启修改国内源，如果没有文件权限则使用chmod命令修改权限，在Raspberry Pi Configuration中开启SSH和VNC。
sudo ifconfig可以查看ip地址。

