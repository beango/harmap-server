---
layout: post
section: Archive
category: default
date: 2011-06-15
title: "MongoDB安装及使用"
description: "MongoDB安装及使用"
tags: [mongodb]
---


### 一、准备工作：

运行yum命令查看MongoDB的包信息

    [root@vm ~]# yum info mongo-10gen  

提示没有相关匹配的信息，说明你的centos系统中的yum源不包含MongoDB的相关资源，所以要在使用yum命令安装MongoDB前需要增加yum源，也就是在 /etc/yum.repos.d/目录中增加 *.repo yum源配置文件，以下分别是针对centos 64位和32位不同的系统的MongoDB yum源配置内容；我们这里就将该文件命名为：/etc/yum.repos.d/10gen.repo  

**For 64-bit yum源配置：**

    vi /etc/yum.repos.d/10gen.repo

    [10gen]  
    name=10gen Repository  
    baseurl=http://downloads-distro.mongodb.org/repo/redhat/os/x86_64  
    gpgcheck=0

**For 32-bit yum源配置：**

    vi /etc/yum.repos.d/10gen.repo  

    [10gen]  
    name=10gen Repository  
    baseurl=http://downloads-distro.mongodb.org/repo/redhat/os/i686  
    gpgcheck=0  

根据自己的系统选择相应的配置内容  
  
**查看系统是32位还是64位的方法：**
  
    $ uname -a  
  
含有x86_64的那说明是64位的，例如我的centos6.0 64bit系统执行这个命令后显示：  
  
> Linux vm.centos6 2.6.32-71.29.1.el6.x86_64 #1 SMP Mon Jun 27 19:49:27 BST 2011 x86_64 x86_64 x86_64 GNU/Linux  
   
  
做好yum源的配置后，如果配置正确执行下面的命令便可以查询MongoDB相关的信息：  
  
**查看mongoDB的服务器包的信息**

    [root@vm ~]# yum info mongo-10gen-server  
    ****(省略多行不重要的信息)*********  
    Available Packages  
    Name       : mongo-10gen-server  
    Arch       : x86_64  
    Version    : 1.8.2  
    Release    : mongodb_1  
    Size       : 4.7 M  
    Repo       : 10gen  
    Summary    : mongo server, sharding server, and support scripts   
    URL        : http://www.mongodb.org  
    License    : AGPL 3.0  
    Description: Mongo (from "huMONGOus") is a schema-free document-oriented  
               : database.  
               :  
               : This package provides the mongo server software, mongo sharding  
               : server softwware, default configuration files, and init.d scripts.  
      
    [root@vm ~]#  

**查看客户端工具的信息**

    [root@vm ~]# yum info mongo-10gen  
    Loaded plugins: fastestmirror  
    **（省略多行不重要的信息）**  
    Installed Packages  
    Name       : mongo-10gen  
    Arch       : x86_64  
    Version    : 1.8.2  
    Release    : mongodb_1  
    Size       : 55 M  
    Repo       : 10gen  
    Summary    : mongo client shell and tools   
    URL        : http://www.mongodb.org  
    License    : AGPL 3.0  
    Description: Mongo (from "huMONGOus") is a schema-free document-oriented  
               : database. It features dynamic profileable queries, full indexing,  
               : replication and fail-over support, efficient storage of large  
               : binary data objects, and auto-sharding.  
               :  
               : This package provides the mongo shell, import/export tools, and  
               : other client utilities.  
      
    [root@vm ~]#

### 二、安装MongoDB的服务器端和客户端工具
  
**安装服务器端：**

    [root@vm ~]# yum install mongo-10gen-server  
    [root@vm ~]# ls /usr/bin/mongo（tab键）  
    mongo         mongod        mongodump     mongoexport   mongofiles    mongoimport   mongorestore  mongos        mongostat  

这些就是MongoDB的程序文件  
因为mongo-10gen-server包依赖于mongo-10gen，所以安装了服务器后就不需要单独安装客户端工具包mongo-10gen了  
  
**单独安装可客户端：**

    [root@vm ~]# yum install mongo-10gen   

**检查**

    [root@vm ~]# /etc/init.d/mongod  
    Usage: /etc/init.d/mongod {start|stop|status|restart|reload|force-reload|condrestart}  
    [root@vm ~]# /etc/init.d/mongod status  
    mongod (pid 1341) is running...  
    [root@vm ~]#

说明安后服务器端已经在运行了  
  
  
**服务器配置: /etc/mongod.conf**

    [root@vm ~]# cat /etc/mongod.conf  
    # mongo.conf  
      
    #where to log  
    logpath=/var/log/mongo/mongod.log  
      
    logappend=true #以追加方式写入日志  
      
    # fork and run in background  
    fork = true  
      
    #port = 27017 #端口  
      
    dbpath=/var/lib/mongo #数据库文件保存位置  
      
    # Enables periodic logging of CPU utilization and I/O wait  
    #启用定期记录CPU利用率和 I/O 等待  
    #cpu = true  
      
    # Turn on/off security.  Off is currently the default  
    # 是否以安全认证方式运行，默认是不认证的非安全方式  
    #noauth = true  
    #auth = true  
      
    # Verbose logging output.  
    # 详细记录输出  
    #verbose = true  
      
    # Inspect all client data for validity on receipt (useful for  
    # developing drivers)用于开发驱动程序时的检查客户端接收数据的有效性  
    #objcheck = true  
      
    # Enable db quota management 启用数据库配额管理，默认每个db可以有8个文件，可以用quotaFiles参数设置  
    #quota = true  
    # 设置oplog记录等级  
    # Set oplogging level where n is  
    #   0=off (default)  
    #   1=W  
    #   2=R  
    #   3=both  
    #   7=W+some reads  
    #oplog = 0  
    
    # Diagnostic/debugging option 动态调试项  
    #nocursors = true  
      
    # Ignore query hints 忽略查询提示  
    #nohints = true  
    # 禁用http界面，默认为localhost：28017  
    # Disable the HTTP interface (Defaults to localhost:27018).这个端口号写的是错的  
    #nohttpinterface = true  
      
    # 关闭服务器端脚本，这将极大的限制功能  
    # Turns off server-side scripting.  This will result in greatly limited  
    # functionality  
    #noscripting = true  
    # 关闭扫描表，任何查询将会是扫描失败  
    # Turns off table scans.  Any query that would do a table scan fails.  
    #notablescan = true  
    # 关闭数据文件预分配  
    # Disable data file preallocation.  
    #noprealloc = true  
    # 为新数据库指定.ns文件的大小，单位:MB  
    # Specify .ns file size for new databases.  
    # nssize = <size>  
      
    # Accout token for Mongo monitoring server.  
    #mms-token = <token>  
    # mongo监控服务器的名称  
    # Server name for Mongo monitoring server.  
    #mms-name = <server-name>  
    # mongo监控服务器的ping 间隔  
    # Ping interval for Mongo monitoring server.  
    #mms-interval = <seconds>  
      
    # Replication Options 复制选项  
      
    # in replicated mongo databases, specify here whether this is a slave or master 在复制中，指定当前是从属关系  
    #slave = true  
    #source = master.example.com  
    # Slave only: specify a single database to replicate  
    #only = master.example.com  
    # or  
    #master = true  
    #source = slave.example.com  
    [root@vm ~]#  

以上是默认的配置文件中的一些参数，更多参数可以用 mongod -h 命令来查看  

    [root@vm ~]# mongod -h  
    Allowed options:  
      
    General options:  
      -h [ --help ]          show this usage information  
      --version              show version information  
      -f [ --config ] arg    configuration file specifying additional options 指定启动配置文件路径  
      -v [ --verbose ]       be more verbose (include multiple times for more  
                             verbosity e.g. -vvvvv)  
      --quiet                quieter output  
      --port arg             specify port number 端口  
      --bind_ip arg          comma separated list of ip addresses to listen on -  
                             all local ips by default 绑定ip，可以多个  
      --maxConns arg         max number of simultaneous connections 最大并发连接数  
      --logpath arg          log file to send write to instead of stdout - has to  
                             be a file, not directory 日志文件路径  
      --logappend            append to logpath instead of over-writing 日志写入方式  
      --pidfilepath arg      full path to pidfile (if not set, no pidfile is  
                             created) pid文件路径  
      --keyFile arg          private key for cluster authentication (only for  
                             replica sets)集群认证私钥，仅适用于副本集  
      --unixSocketPrefix arg alternative directory for UNIX domain sockets  
                             (defaults to /tmp)替代目录  
      --fork                 fork server process  
      --auth                 run with security 使用认证方式运行  
      --cpu                  periodically show cpu and iowait utilization 定期显示的CPU和IO等待利用率  
      --dbpath arg           directory for datafiles 数据库文件路径  
      --diaglog arg          0=off 1=W 2=R 3=both 7=W+some reads oplog记录等级  
      --directoryperdb       each database will be stored in a separate directory  
                             每个数据库存储到单独目录  
      --journal              enable journaling 记录日志，建议开启，在异常宕机时可以恢复一些数据  
      --journalOptions arg   journal diagnostic options  
      --ipv6                 enable IPv6 support (disabled by default)  
      --jsonp                allow JSONP access via http (has security  
                             implications)允许JSONP通过http访问，该方式存在安全隐患  
      --noauth               run without security 不带安全认证的方式  
      --nohttpinterface      disable http interface 禁用http接口  
      --noprealloc           disable data file preallocation - will often hurt  
                             performance 禁用数据文件的预分配，往往会损害性能  
      --noscripting          disable scripting engine 禁用脚本引擎  
      --notablescan          do not allow table scans 不允许表扫描  
      --nounixsocket         disable listening on unix sockets禁止unix sockets监听  
      --nssize arg (=16)     .ns file size (in MB) for new databases 为新数据设置.ns文件的大小  
      --objcheck             inspect client data for validity on receipt 检查在收到客户端的数据的有效性  
      --profile arg          0=off 1=slow, 2=all  
      --quota                limits each database to a certain number of files (8  
                             default)启用数据库配额管理，默认每个db可以有8个文件，可以用quotaFiles参数设置  
      --quotaFiles arg       number of files allower per db, requires --quota  
      --rest                 turn on simple rest api 开启rest api  
      --repair               run repair on all dbs 修复所有数据库  
      --repairpath arg       root directory for repair files - defaults to dbpath修复文件的根目录，默  
                             认为dbpath指定的目录  
      --slowms arg (=100)    value of slow for profile and console log  
      --smallfiles           use a smaller default file size  
      --syncdelay arg (=60)  seconds between disk syncs (0=never, but not  
                             recommended)与硬盘同步数据的时间，默认60秒，0表示不同步到硬盘（不建议）  
      --sysinfo              print some diagnostic system information打印一些诊断系统信息  
      --upgrade              upgrade db if needed 如果必要，将数据库文件升级到新的格式  
                            （<=1.0到1.1+升级时所需的）  
      
    Replication options:    复制选项  
      --fastsync            indicate that this instance is starting from a dbpath  
                            snapshot of the repl peer 从一个dbpath快照开始同步  
      --autoresync          automatically resync if slave data is stale 自动同步，如果从机的数据不是新的  
                            自动同步  
      --oplogSize arg       size limit (in MB) for op log oplog的大小  
      
    Master/slave options:   主/从配置选项  
      --master              master mode 主模式  
      --slave               slave mode  从属模式  
      --source arg          when slave: specify master as <server:port>从属服务器上指定主服务器地址  
      --only arg            when slave: specify a single database to replicate从属服务器上指定要复制的  
                            数据库  
      --slavedelay arg      specify delay (in seconds) to be used when applying  
                            master ops to slave 指定从主服务器上同步数据的时间间隔 单位秒  
      
    Replica set options:    副本集选项  
      --replSet arg         arg is <setname>[/<optionalseedhostlist>]  
                            参数：<名称>[<种子主机列表>]  
      
    Sharding options:       分片设置选项  
      --configsvr           declare this is a config db of a cluster; default port  
                            27019; default dir /data/configdb 声明这是一个集群的配置数据库，  
                            默认的端口是27019 默认的路径是/data/configdb  
      --shardsvr            declare this is a shard db of a cluster; default port  
                            27018 声明这是集群的一个分片数据库，默认端口为27018  
      --noMoveParanoia      turn off paranoid saving of data for moveChunk.  this  
                            is on by default for now, but default will switch  
                            关闭偏着保存大块数据。现在它是默认的，但是会变换  
      
    [root@vm ~]#  

### 三、基本操作###
  
**连接到数据库：**

    [root@vm ~]# mongo  
    MongoDB shell version: 2.2.1  
    connecting to: test


**insert操作：**

    db.customer.insert({"name":"jack","age":21})  
    db.customer.insert({"name":"joe","age":24})

**find操作：**

    db.customer.find()  
    db.customer.find({"name":"joe"})


**update操作：**

    db.customer.update({"name":"joe"},{"name":"joe","age":28})

**remove操作：**

    db.customer.remove({"name":"joe"})  
    db.customer.remove()#删除所有

**索引：**

    db.customer.ensureIndex({"name":1}) #添加索引
    db.customer.getIndexes() #查看所有索引
    db.customer.dropIndex({"name":1}) #删除索引
    db.customer.ensureIndex({"name":1},{"unique":true}) #唯一索引

**数据备份mongodump/恢复mongorestore：**

    ./mongodump -d 数据库名  #将数据库备份在dump目录，-o my_mongodb_dump将数据库备份在my_mongodb_dump目录
    ./mongorestore -d 数据库名 备份路径/*   #将备份路径下的文件恢复为数据库

**数据导出 mongoexport/导入mongoimport：**

    ./mongoexport -d 数据库名 -c 表名 -o 表名.dat   #将数据库中的表数据导出到dat文件
    ./mongoimport -d 数据库名 -c 表名 表名.dat      #将dat文件导入到数据库中的表

**其他：**

    show dbs;  
    use MyTest;  
    show collections;  
    db.charpter.count({$where: "this.content.length>0"})  
    use test;  
    db.dropDatabase();//删除数据库
