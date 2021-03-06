---
layout: post
category: default
date: 2013-02-27
title: "可伸缩系统的架构经验"
description: "可伸缩系统的架构经验"
tags: [架构分析]
redirecturl: http://agiledon.github.com/blog/2013/02/27/scalability-system-architecture-lessons/
---


来源：[简单文本](http://agiledon.github.com/blog/2013/02/27/scalability-system-architecture-lessons/)

最近，阅读了Will Larson的文章[Introduction to Architecting System for Scale](http://lethain.com/introduction-to-architecting-systems-for-scale/)，感觉很有价值。作者分享了他在Yahoo!与Digg收获的设计可伸缩系统的架构经验。在我过往的架构经验中，由于主要参与开发企业软件系统，这种面向企业内部的软件系统通常不会有太大的负载量，太多的并发量，因而对于系统的可伸缩性考虑较少。大体而言，只要在系统部署上考虑集群以及负载均衡即可。本文给了我很多启发，现把本文的主要内容摘译出来，并结合自己对此的理解。

Larson首先认为，一个理想的系统，对于容量（Capacity）的增长应该与添加的硬件数是线性的关系。换言之，如果系统只有一台服务器，在增加了另一台同样的机器后，容量应该翻倍。以此类推。这种线性的容量伸缩方式，通常被称之为水平伸缩“Horizontal Scalability”。

[![可伸缩系统的架构经验](/post-images/2013-02/scalability.jpg "可伸缩系统的架构经验")](/post-images/2013-02/scalability.jpg "可伸缩系统的架构经验")

在设计一个健壮的系统时，自然必须首要考虑失败的情况。Larson认为，一个理想的系统是当失去其中一台服务器的时候，系统不会崩溃。当然，对应而言，失去一台服务器也会导致容量的响应线性减少。这种情况通常被称为冗余“Redundancy”。

**负载均衡**

无论是水平伸缩还是冗余，都可以通过负载均衡来实现。负载均衡就好似一个协调请求的调停者，它会根据集群中机器的当前负载，合理的分配发往Web服务器的请求，以达到有效利用集群中各台机器资源的目的。显然，这种均衡器应该介于客户端与Web服务器之间，如下图所示：[![可伸缩系统的架构经验](/post-images/2013-02/scalability01.png "可伸缩系统的架构经验")](/post-images/2013-02/scalability01.png "可伸缩系统的架构经验")

本文提到了实现负载均衡的几种方法。其一是Smart
Client，即将负载均衡的功能添加到数据库（以及缓存或服务）的客户端中。这是一种通过软件来实现负载均衡的方式，它的缺点是方案会比较复杂，不够健壮，也很难被重用（因为协调请求的逻辑会混杂在业务系统中）。对此，Larson在文章以排比的方式连续提出问题，以强化自己对此方案的不认可态度：

> Is it attractive because it is the simplest solution? Usually, no. Is
> it seductive because it is the most robust? Sadly, no. Is it alluring
> because it’ll be easy to reuse? Tragically, no.

第二种方式是采用硬件负载均衡器，例如[Citrix NetScaler](http://www.citrix.com/English/ps2/products/product.asp?contentID=21679)。不过，购买硬件的费用不菲，通常是一些大型公司才会考虑此方案。

如果既不愿意承受Smart Client的痛苦，又不希望花费太多费用去购买硬件，那就可以采用一种混合（Hybird）的方式，称之为软件负载均衡器（Software Load Balancer）。Larson提到了[HAProxy](http://haproxy.1wt.eu/)。它会运行在本地，需要负载均衡的服务都会在本地中得到均衡和协调。

**缓存**

为了减轻服务器的负载，还需要引入缓存。文章给出了常见的对缓存的分类，分别包括：预先计算结果（precalculating result，例如针对相关逻辑的前一天的访问量）、预先生成昂贵的索引（pre-generating expensive indexes，例如用户点击历史的推荐）以及在更快的后端存储频繁访问的数据的副本（例如[Memcached](http://memcached.org/)）。

**应用缓存**

提供缓存的方式可以分为应用缓存和数据库缓存。此二者各擅胜场。应用缓存通常需要将处理缓存的代码显式地集成到应用代码中。这就有点像使用代理模式来为真实对象提供缓存。首先检查缓存中是否有需要的数据，如果有，就从缓存直接返回，否则再查询数据库。至于哪些值需要放到缓存中呢？有[诸多算法](http://en.wikipedia.org/wiki/Cache_algorithms#Least_Recently_Used)，例如根据最近访问的，或者根据访问频率。使用Memcached的代码如下所示：

    key = "user.%s" % user_id
    user_blob = memcache.get(key)
    if user_blob is None:
        user = mysql.query("SELECT * FROM users WHERE user_id=\"%s\"", user_id)
        if user:
            memcache.set(key, json.dumps(user))
        return user
    else:
        return json.loads(user_blob)

**数据库缓存**

数据库缓存对于应用代码没有污染，一些天才的DBA甚至可以在不修改任何代码的情况下，通过数据库调优来改进系统性能。例如通过配置Cassandra行缓存。

**内存缓存**

为了提高性能，缓存通常是存储在内存中。常见的内存缓存包括Memcached和[Redis](http://redis.io/)。不过采用这种方式仍然需要合理的权衡。我们不可能一股脑儿的将所有数据都存放在内存中，虽然这会极大地改善性能，但比较起磁盘存储而言，RAM的代价更昂贵，同时还会影响系统的健壮性，因为内存中的数据没有持久化，容易丢失。正如之前提到的，我们应该将需要的数据放入缓存，通常的算法是[least recently used](http://en.wikipedia.org/wiki/Cache_algorithms#Least_Recently_Used)，即LRU。

**CDN**

提高性能，降低Web服务器负载的另一种常见做法是将静态媒体放入CDN（Content Distribution Network）中。如下图所示：[![可伸缩系统的架构经验](/post-images/2013-02/scalability02.png "可伸缩系统的架构经验")](/post-images/2013-02/scalability02.png "可伸缩系统的架构经验")

CDN可以有效地分担Web服务器的压力，使得应用服务器可以专心致志地处理动态页面；同时，CDN还可以通过地理分布来提高响应请求的性能。在设置了CDN后，当系统接收到请求时，首先会询问CDN以获得请求中需要的静态媒体（通常会通过HTTP Header来配置CDN能够缓存的内容）。如果请求的内容不可用，CDN会查询服务器以获得该文件，并在CDN本地进行缓存，最后再提供给请求者。如果当前网站并不大，引入CDN的效果不明显时，可以考虑暂不使用CDN，在将来可以通过使用一些轻量级的HTTP服务器如[Nginx](http://nginx.org/)，为静态媒体分出专门的子域名如static.domain.com来提供服务。

**缓存失效**

引入缓存所带来的问题是如何保证真实数据与缓存数据之间的一致性。这一问题通常被称之为缓存失效（Cache Invalidation）。从高屋建瓴的角度来讲，解决这一问题的办法无非即使更新缓存中的数据。一种做法是直接将新值写入缓存中（通常被称为write-through cache）；另一种做法是简单地删除缓存中的值，在等到下一次读缓存值的时候再生成。

整体而言，要避免缓存实效，可以依赖于数据库缓存，或者为缓存数据添加有效期，又或者在实现应用程序逻辑时，尽量考虑避免此问题。例如不直接使用DELETE FROM a WHERE…来删除数据，而是先查询符合条件的数据，再使得缓存中对应的数据失效，继而根据其主键显式地删除这些行。

**Off-Line处理**

这篇文章还提到了Off-Line的处理方式，即通过引入消息队列的方式来处理请求。事实上，在大多数企业软件系统中，这种方式也是较为常见的做法。在我撰写的文章《[案例分析:基于消息的分布式架构](http://agiledon.github.com/blog/2012/12/27/distributed-architecture-based-on-message/)》中，较为详细地介绍了这种架构。在引入消息队列后，Web服务器会充当消息的发布者，而在消息队列的另一端可以根据需要提供消费者Consumer。如下图所示。对于Off-Line的任务是否执行完毕，通常可以通过轮询或回调的方式来获知。[![可伸缩系统的架构经验](/post-images/2013-02/scalability03.png "可伸缩系统的架构经验")](/post-images/2013-02/scalability03.png "可伸缩系统的架构经验")

为了更好地提高代码可读性，可以在公开的接口定义中明确地标示该任务是On-Line还是Off-Line。

引入Message Queue，可以极大地缓解Web服务器的压力，因为它可以将耗时较长的任务转到专门的机器上去执行。

此外，通过引入定时任务，也可以有效地利用Web服务器的空闲时间来处理后台任务。例如，通过Spring Batch Job来执行每日、每周或者每月的定时任务。如果需要多台机器去执行这些定时任务，可以引入Spring提供的[Puppet](https://puppetlabs.com/)来管理这些服务器。Puppet提供了可读性强的声明性语言来完成对机器的配置。

**Map-Reduce**

对于大数据的处理，自然可以引入Map-Reduce。为整个系统专门引入一个Map-Reduce层来处理数据是有必要的。相对于使用SQL数据库作为数据中心的方式，Map-Reduce对可伸缩性的支持更好。Map-Reduce可以与任务的定时机制结合起来。如下图所示：[![可伸缩系统的架构经验](/post-images/2013-02/scalability04.png "可伸缩系统的架构经验")](/post-images/2013-02/scalability04.png "可伸缩系统的架构经验")

**平台层**

Larson认为，大多数系统都是Web应用直接与数据库通信，但如果能加入一个平台层（Platform Layer），或许会更好。[![可伸缩系统的架构经验](/post-images/2013-02/scalability05.png "可伸缩系统的架构经验")](/post-images/2013-02/scalability05.png "可伸缩系统的架构经验")

首先，将平台与Web应用分离，使得它们可以独立地进行伸缩。例如需要添加一个新的API，就可以添加新的平台服务器，而无需增加Web服务器。要知道，在这样一个独立的物理分层架构中，不同层次对服务器的要求是不一样的。例如，对于数据库服务器而言，由于需要频繁地对磁盘进行I/O操作，因此应保证数据库服务器的IO性能，如尽量使用固态硬盘。而对于Web服务器而言，则对CPU的要求比较高，尽可能采用多核CPU。

其次，增加一个额外的平台层，可以有效地提高系统的可重用性。例如我们可以将一些与系统共有特性以及横切关注点的内容（如对缓存的支持，对数据库的访问等功能）抽取到平台层中，作为整个系统的基础设施（Infrastructure）。尤其对于产品线系统而言，这种架构可以更好地为多产品提供服务。

最后，这种架构也可能对跨团队开发带来好处。平台可以抽离出一些与产品无关的接口，从而隐藏其具体实现的细节。如果划分合理，并能设计出相对稳定的接口，就可以使得各个团队可以并行开发。例如可以专门成立平台团队，致力于对平台的实现以及优化。

 
