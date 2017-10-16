---
layout: post
category: default
date: 2012-02-20
title: "Java堆内存的10个要点"
description: "Java堆内存的10个要点"
tags: [java]
redirecturl: http://blog.jobbole.com/13373/
---


当我开始学习Java编程时，我不知道什么是堆内存或堆空间，我甚至不知道当对象创建时，它们被放在了哪里。当我开始正式写一些程序后，我会经常遇到java.lang.outOfMemoryError的报错，之后我才开始关注什么是堆内存或者说堆空间(heap space)。对大多数程序员都经历过这样的过程，因为学习一种语言是非常容易来的，但是学习基础是非常难的，因为没有什么特定的流程让你学习编程的每个基础，使你发觉编程的秘诀。对于程序员来说，知道堆空间，设置堆空间，处理堆空间的outOfMemoryError错误，分析heap dump是非常重要的。这个关于Java堆的教程是给我刚开始学编程的兄弟看的。如果你知道这个基础知识或者知道底层发生了什么，当然可能帮助不是那么大。除非你知道了对象被创建在堆中，否则你不会意识到OutOfMemoryError是发生在堆空间中的。我尽可能的将我所知道的所有关于堆的知识都写下来了，也希望你们能够尽可能多的贡献和分享你的知识，以便可以让其他人也受益。

**Java中的堆空间是什么？**

当Java程序开始运行时，JVM会从操作系统获取一些内存。JVM使用这些内存，这些内存的一部分就是堆内存。堆内存通常在存储地址的底层，向上排列。当一个对象通过new关键字或通过其他方式创建后，对象从堆中获得内存。当对象不再使用了，被当做垃圾回收掉后，这些内存又重新回到堆内存中。要学习垃圾回收，请阅读”Java中垃圾回收的工作原理”。

**如何增加Java堆空间**

在大多数32位机、Sun的JVM上，Java的堆空间默认的大小为128MB，但也有例外，例如在32未Solaris操作系统(SPARC平台版本)上，默认的最大堆空间和起始堆空间大小为 -Xms=3670K 和 -Xmx=64M。对于64位操作系统，一般堆空间大小增加约30%。但你使用Java 1.5的throughput垃圾回收器，默认最大的堆大小为物理内存的四分之一，而起始堆大小为物理内存的十六分之一。要想知道默认的堆大小的方法，可以用默认的设置参数打开一个程序，使用JConsole(JDK 1.5之后都支持)来查看，在VM Summary页面可以看到最大的堆大小。

用这种方法你可以根据你的程序的需要来改变堆内存大小，我强烈建议采用这种方法而不是默认值。如果你的程序很大，有很多对象需要被创建的话，你可以用-Xms and -Xmx这两个参数来改变堆内存的大小。Xms表示起始的堆内存大小，Xmx表示最大的堆内存的大小。另外有一个参数 -Xmn，它表示new generation（后面会提到）的大小。有一件事你需要注意，你不能任意改变堆内存的大小，你只能在启动JVM时设定它。

**堆和垃圾回收**

我们知道对象创建在堆内存中，垃圾回收这样一个进程，它将已死对象清除出堆空间，并将这些内存再还给堆。为了给垃圾回收器使用，堆主要分成三个区域，分别叫作New Generation，Old Generation或叫Tenured Generation，以及Perm space。New Generation是用来存放新建的对象的空间，在对象新建的时候被使用。如果长时间还使用的话，它们会被垃圾回收器移动到Old Generation(或叫Tenured Generation)。Perm space是JVM存放Meta数据的地方，例如类，方法，字符串池和类级别的详细信息。你可以查看“Java中垃圾回收的工作原理”来获得更多关于堆和垃圾回收的信息。

**Java堆中的OutOfMemoryError错误**

当JVM启动时，使用了-Xms参数设置的对内存。当程序继续进行，创建更多对象，JVM开始扩大堆内存以容纳更多对象。JVM也会使用垃圾回收器来回收内存。当快达到-Xmx设置的最大堆内存时，如果没有更多的内存可被分配给新对象的话，JVM就会抛出java.lang.outofmemoryerror，你的程序就会当掉。在抛出OutOfMemoryError之前，JVM会尝试着用垃圾回收器来释放足够的空间，但是发现仍旧没有足够的空间时，就会抛出这个错误。为了解决这个问题，你需要清楚你的程序对象的信息，例如，你创建了哪些对象，哪些对象占用了多少空间等等。你可以使用profiler或者堆分析器来处理OutOfMemoryError错误。”java.lang.OutOfMemoryError: Java heap space”表示堆没有足够的空间了，不能继续扩大了。”java.lang.OutOfMemoryError: PermGen space”表示permanent generation已经装满了，你的程序不能再装在类或者再分配一个字符串了。

**Java Heap dump**

Heap dump是在某一时间对Java堆内存的快照。它对于分析堆内存或处理内存泄露和Java.lang.outofmemoryerror错误是非常有用的。在JDK中有一些工具可以帮你获取heap dump，也有一些堆分析工具来帮你分析heap dump。你可以用“jmap”来获取heap dump，它帮你创建heap dump文件，然后，你可以用“jhat”（堆分析工具）来分析这些heap dump。

**Java堆内存(heap memory)的十个要点**

​1. Java堆内存是操作系统分配给JVM的内存的一部分。

​2. 当我们创建对象时，它们存储在Java堆内存中。

​3. 为了便于垃圾回收，Java堆空间分成三个区域，分别叫作New Generation, Old Generation或叫作Tenured Generation，还有Perm Space。

​4. 你可以通过用JVM的命令行选项 -Xms, -Xmx, -Xmn来调整Java堆空间的大小。不要忘了在大小后面加上”M”或者”G”来表示单位。举个例子，你可以用-Xmx256m来设置堆内存最大的大小为256MB。

​5. 你可以用JConsole或者 Runtime.maxMemory(), Runtime.totalMemory(), Runtime.freeMemory()来查看Java中堆内存的大小。

​6. 你可以使用命令“jmap”来获得heap dump，用“jhat”来分析heap dump。

​7. Java堆空间不同于栈空间，栈空间是用来储存调用栈和局部变量的。

​8.Java垃圾回收器是用来将死掉的对象(不再使用的对象)所占用的内存回收回来，再释放到Java堆空间中。

​9.当你遇到java.lang.outOfMemoryError时，不要紧张，有时候仅仅增加堆空间就可以了，但如果经常出现的话，就要看看Java程序中是不是存在内存泄露了。

​10. 请使用Profiler和Heap dump分析工具来查看Java堆空间，可以查看给每个对象分配了多少内存。


原文链接：[java revisited](http://javarevisited.blogspot.com/2011/05/java-heap-space-memory-size-jvm.html)
