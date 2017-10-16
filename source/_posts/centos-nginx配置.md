---
title: centos+nginx配置
date: 2017-08-29 10:29:57
tags: [centos, nginx]
---

## 系统环境

搬瓦工2.99/年 72M内存，安装Centos 6 x86_64

### 显示中文

locale查看当前编码。

更改root下的.bash_profile文件，添加：

``` bash
LANG=zh_CN
LC_ALL=zh_CN.gbk
LC_CTYPE=zh_CN.gbk
export LANG
export LC_ALL
export LC_CTYPE
```

FTP上传文件改成UTF-8编码。

<!--more-->

### 安装 Nginx

``` bash
wget http://nginx.org/packages/centos/6/noarch/RPMS/nginx-release-centos-6-0.el6.ngx.noarch.rpm
rpm -ivh nginx-release-centos-6-0.el6.ngx.noarch.rpm 
yum install nginx
```

### nginx的几个默认目录

配置所在目录：/etc/nginx/
PID目录：/var/run/nginx.pid
错误日志：/var/log/nginx/error.log
访问日志：/var/log/nginx/access.log
默认站点目录：/usr/share/nginx/html

启动nginx：nginx
