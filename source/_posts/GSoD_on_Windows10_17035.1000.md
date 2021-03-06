---
title: Windows10 17035.1000的绿屏问题解决
date: 2018-01-12
---

XPS13使用Windows10开发者预览版快速通道更新17035.1000之后，打开电脑一段时间后会出现BAD_POOL_CALLER绿屏，只能通过重启解决。

在设置->开发者预览里把快速通道调至慢速通道后，再次绿屏，错误代码为`SYSTEM_THREAD_EXCEPTION_NOT_HANDLED`，此时不能进入系统主界面。在安全模式中删除Realtek的驱动，还是无效。备份系统所有数据到移动硬盘。尝试不删除数据重置电脑，出现“计算机意外地重新启动或遇到错误，windows安装无法继续”。回滚系统。尝试删除个人数据重置电脑，再次出现“计算机意外地重新启动或遇到错误，windows安装无法继续”，此时无法回滚。尝试运行oobe里的msoobe.exe，还是无效。

最终只有用u盘重装系统一条路。最后使用一个16g的HP u盘进行了恢复盘安装。通过dell微信客服问来https://www.microsoft.com/zh-cn/software-download/windows10?linkId=44645463 网址获取镜像。安装u盘时，因为是通过uefi格式安装，所以不修改bios设置会出现问题，修改uefi secure boot和legacy boot都为off，重新进入F12，开始重装系统。过程基本顺利，删除所有分区之后新建分区。从1507更新到1709系统。驱动根据服务标签在http://www.dell.com/support/home/cn/zh/cndhs1?app=drivers&~ck=mn上查找，或者直接下载最新的Dell update应用程序自动查找所需驱动。

退出win10开发者预览通道，慢速和快速都退出，恢复稳定版本。

