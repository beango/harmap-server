---
title: Oracle账号密码过期
date: 2017-09-06 11:50:15
tags: [ORA-28000,Oracle账号被锁,Oracle账号过期]
---

Oracle 11g 之前默认的用户时是没有密码过期的限制的，在Oracle 11g 中默认的profile启用了密码过期时间是180天。

<!--more-->

``` bash
select * from dba_profiles where profile='DEFAULT' and resource_name='PASSWORD_LIFE_TIME';
```

如果是已经锁定了账户：

cmd 设置全局SID 

``` bash
set ORACLE_SID=orcl
```

登录数据库

``` bash
sqlplus /nolog
sql > conn orcl/pwd@orcl as SYSDBA
```

将用户密码过期时间改为不过期 

``` bash
alter profile default limit password_life_time unlimited;
commit;
```

将过期的密码重置为可用状态
 
 ``` bash
 ALTER USER sgdsproject IDENTIFIED BY sgdsproject_new;

 ALTER USER sgdsproject IDENTIFIED BY sgdsproject;
``` 
 
全部操作完成。
