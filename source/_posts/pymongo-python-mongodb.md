---
layout: post
section: Archive
category: default
date: 2012-11-23
title: "MongoDB的Python接口PyMongo"
description: "MongoDB的Python接口PyMongo"
tags: [mongodb]
---


*   创建Connection时，指定host及port参数

```python
>>> import pymongo  
>>> conn = pymongo.Connection(host='127.0.0.1',port=27017)
```perl

*   连接数据库

```python
>>> db = conn.ChatRoom  
或  
>>> db = conn['ChatRoom']
```perl

*   连接聚集

```python
>>> account = db.Account  
或  
>>> account = db["Account"]
```perl

*   查看全部聚集名称

```python
>>> db.collection_names()
```perl

*   查看聚集的一条记录

```python
>>> db.Account.find_one()  
>>> db.Account.find_one({"UserName":"keyword"})
```perl

*   查看聚集的字段

```python
>>> db.Account.find_one({},{"UserName":1,"Email":1})  
>>>    {u'UserName': u'libing', u'_id': ObjectId('4ded95c3b7780a774a099b7c'), u'Email': u'libing@35.cn'}  
>>> db.Account.find_one({},{"UserName":1,"Email":1,"_id":0})  
>>>    {u'UserName': u'libing', u'Email': u'libing@35.cn'}
```perl

*   查看聚集的多条记录

```python
>>> for item in db.Account.find():  
>>>    item  
>>> for item in db.Account.find({"UserName":"libing"}):  
>>>    item["UserName"]
```perl

*   查看聚集的记录统计

```python
>>> db.Account.find().count()  
>>> db.Account.find({"UserName":"keyword"}).count()
```perl

*   聚集查询结果排序

```python
>>> db.Account.find().sort("UserName")  --默认为升序  
>>> db.Account.find().sort("UserName",pymongo.ASCENDING)   --升序  
>>> db.Account.find().sort("UserName",pymongo.DESCENDING)  --降序
```perl

*   聚集查询结果多列排序

```python
>>> db.Account.find().sort([("UserName",pymongo.ASCENDING),("Email",pymongo.DESCENDING)])
```perl

*   添加记录

```python
>>> db.Account.insert({"AccountID":21,"UserName":"libing"})
```perl

*   修改记录

```python
>>> db.Account.update({"UserName":"libing"},{"$set":{"Email":"xx@126.com","Password":"123"}})
```perl

*   删除记录

```python
>>> db.Account.remove()   -- 全部删除  
>>> db.Test.remove({"UserName":"keyword"})
```perl

*   删除数据库

```python
>>> use test   -- 切换数据库  
>>> db.dropDatabase() -- 删除当前库
```perl
