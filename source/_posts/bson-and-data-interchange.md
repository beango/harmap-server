---
layout: post
category: default
date: 2013-01-12
title: "BSON特性探讨及基于其特性的MongoDB优化"
description: "BSON特性探讨及基于其特性的MongoDB优化"
tags: [bson, mongodb, 优化]
redirecturl: http://blog.nosqlfan.com/html/2914.html
---


[BSON](http://bsonspec.org/)是由10gen开发的一个数据格式，目前主要用于[MongoDB](http://blog.nosqlfan.com/tags/mongodb "查看 MongoDB 的全部文章")中，是MongoDB的数据存储格式。BSON基于JSON格式，选择JSON进行改造的原因主要是JSON的通用性及JSON的schemaless的特性。

![](/post-images/2013-01/UIQZU.gif "BSON")

BSON主要会实现以下三点目标：

### 1.更快的遍历速度

对JSON格式来说，太大的JSON结构会导致数据遍历非常慢。在JSON中，要跳过一个文档进行数据读取，需要对此文档进行扫描才行，需要进行麻烦的数据结构匹配，比如括号的匹配，而BSON对JSON的一大改进就是，它会将JSON的每一个元素的长度存在元素的头部，这样你只需要读取到元素长度就能直接seek到指定的点上进行读取了。

**MongoDB优化**：对于MongoDB来说，由于采用了MMAP来做内存与数据文件的映射，在更新或者获取Document的某一个字段时，如果需要先读取其前面的所有字段，会导致物理内存由于读操作被加载到不必要的字段上，导致资源的不合理分配。而采用BSON只需要读到相应的位置然后跨过无用内容读取需要内容即可。

### 2.操作更简易

对JSON来说，数据存储是无类型的，比如你要修改基本一个值，从9到10，由于从一个字符变成了两个，所以可能其后面的所有内容都需要往后移一位才可以。而使用BSON，你可以指定这个列为数字列，那么无论数字从9长到10还是100，我们都只是在存储数字的那一位上进行修改，不会导致数据总长变大。当然，在MongoDB中，如果数字从整形增大到长整型，还是会导致数据总长变大的。

**MongoDB优化**：所以使用MongoDB的一个技巧是将长度可能变化的字段尽量命名靠后（MongoDB在update操作后会将字段按key值按字母顺序重排，所以靠后的意思是按a－z的顺序取名）。这样在更新的时候如果导致数字变长，不需要移动大量数据。一个典型的例子是如果用二进制类型存储文件时，如果文件名或者文件描述可能会变长，那么尽量将这个字段取名靠后是一个明智的选择，否则在文件名或文件描述字段变化时，会导致移动很长的二进制数据，造成不必要的浪费。

### 3.增加了额外的数据类型

JSON是一个很方便的数据交换格式，但是其类型比较有限。BSON在其基础上增加了“byte
array”数据类型。这使得二进制的存储不再需要先base64转换后再存成JSON。大大减少了计算开销和数据大小。

当然，在有的时候，BSON相对JSON来说也并没有空间上的优势，比如对{“field”:7}，在JSON的存储上7只使用了一个字节，而如果用BSON，那就是至少4个字节（32位）

**MongoDB优化**：在MongoDB中，如果你的字段是数字型，并且涉及到数据加减操作的，那么建议存在int型，但如果是一个固定不变的数字，并且在四位以下的话，可以考虑存成字符串类型。这样会节省空间。

目前在10gen的努力下，BSON已经有了针对多种语言的编码解码包。并且都是Apache 2 license下开源的。并且还在随着MongoDB进一步地发展。关于BSON，你可以在其官方网站 [bsonspec.org](http://bsonspec.org/)上获取更多信息。

来源：[blog.mongodb.org](http://blog.mongodb.org/post/9333386434/bson-and-data-interchange)
