---
title: oracle连接超时-windows平台日志导致监听异常
date: 2017-10-16 11:18:13
tags: oracle
---

操作系统：windows2008 R2 64bit

数据库版本：Oracle 11.2.0.1

文件系统：ntfs

故障现象：数据库反应非常慢，其他数据库通过db link访问非常慢，但无报错，处于hang住状态。使用tnsping 数据库一直hang住，不报错也不返回结果。在数据库服务器本机使用lsnrctl status也hang住。

<!--more-->

经检查，发现listener.log文件达到4G，但由于文件系统格式是ntfs，允许单个文件大于4G。


解决方法：

1.方法一：先lsnrctl stop停止监听，将listener.log改名，然后再lsnrctl start开启监听，此时会新生成一个空的listener.log文件；

2.方法二：执行lsnrctl set log_status off，停止产生监听日志。