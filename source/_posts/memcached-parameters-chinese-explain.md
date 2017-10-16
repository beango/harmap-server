---
layout: post
section: Archive
category: default
date: 2011-09-11
title: "memcached参数中文解释"
description: "memcached参数中文解释"
tags: [memcached]
---

```perl
memcached 1.4.2  
-p <num>      监听的TCP端口(默认: 11211)  
-U <num>      监听的UDP端口(默认: 11211, 0表示不监听)  
-s <file>     用于监听的UNIX套接字路径（禁用网络支持）  
-a <mask>     UNIX套接字访问掩码，八进制数字（默认：0700）  
-l <ip_addr>  监听的IP地址。（默认：INADDR_ANY，所有地址）  
-d            作为守护进程来运行。  
-r            最大核心文件限制。  
-u <username> 设定进程所属用户。（只有root用户可以使用这个参数）  
-m <num>      单个数据项的最大可用内存，以MB为单位。（默认：64MB）  
-M            内存用光时报错。（不会删除数据）  
-c <num>      最大并发连接数。（默认：1024）  
-k            锁定所有内存页。注意你可以锁定的内存上限。  
              试图分配更多内存会失败的，所以留意启动守护进程时所用的用户可分配的内存上限。  
              （不是前面的 -u <username> 参数；在sh下，使用命令"ulimit -S -l NUM_KB"来设置。）  
-v            提示信息（在事件循环中打印错误/警告信息。）  
-vv           详细信息（还打印客户端命令/响应）  
-vvv          超详细信息（还打印内部状态的变化）  
-h            打印这个帮助信息并退出。  
-i            打印memcached和libevent的许可。  
-P <file>     保存进程ID到指定文件，只有在使用 -d 选项的时候才有意义。  
-f <factor>   块大小增长因子。（默认：1.25）  
-n <bytes>    分配给key+value+flags的最小空间（默认：48）  
-L            尝试使用大内存页（如果可用的话）。提高内存页尺寸可以减少"页表缓冲（TLB）"丢失次数，提高运行效率。  
              为了从操作系统获得大内存页，memcached会把全部数据项分配到一个大区块。  
-D <char>     使用 <char> 作为前缀和ID的分隔符。  
              这个用于按前缀获得状态报告。默认是":"（冒号）。  
              如果指定了这个参数，则状态收集会自动开启；如果没指定，则需要用命令"stats detail on"来开启。  
-t <num>      使用的线程数（默认：4）  
-R            每个连接可处理的最大请求数。  
-C            禁用CAS。  
-b            设置后台日志队列的长度（默认：1024）  
-B            绑定协议 - 可能值：ascii,binary,auto（默认）  
-I            重写每个数据页尺寸。调整数据项最大尺寸。
```perl
memcached在线地址：[https://github.com/liuxd/MyTranslation/blob/master/translation/memcached-1.4.man](https://github.com/liuxd/MyTranslation/blob/master/translation/memcached-1.4.man)