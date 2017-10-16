---
layout: post
category: default
date: 2012-12-25
title: ".Net垃圾回收机制原理（二）"
description: ".Net垃圾回收机制原理（二）"
tags: [.net]
redirecturl: http://www.cnblogs.com/yukaizhao/archive/2011/11/25/dot_net_GC_2.html
---


英文原文：[Jeffrey Richter](http://msdn.microsoft.com/zh-cn/magazine/bb985011\(en-us\).aspx)，编译：[赵玉开](http://www.cnblogs.com/yukaizhao/archive/2011/11/25/dot_net_GC_2.html)

[上一篇文章](/archives/2012/12/13/dot-net-gc-1.html)介绍了.Net垃圾回收的基本原理和垃圾回收执行Finalize方法的内部机制；这一篇我们看下弱引用对象，代，多线程垃圾回收，大对象处理以及和垃圾回收相关的性能计数器。

让我们从弱引用对象说起，弱引用对象可以减轻大对象带来的内存压力。

**弱引用（Weak References）**

当程序的根对象指向一个对象时，这个对象是可达的，垃圾回收器不能回收它，这称为对对象的强引用。和强引用相对的是弱引用，当一个对象上存在弱引用时，垃圾回收器可以回收此对象，但是也允许程序访问这个对象。这是怎么回事儿呢？请往下看。

如果一个对象上仅存在弱引用，并且垃圾回收器在运行，这个对象就会被回收，之后如果程序中要访问这个对象，访问就会失败。另一方面，要使用弱引用的对象，程序必须先对这个对象进行强引用，如果程序在垃圾回收器回收这个对象之前对对象进行了强引用，这样（有了强引用之后）垃圾回收器就不能回收此对象了。这有点绕，让我们用一段代码来说明一下：

    void Method() {
        //创建对象的强引用
        Object o = new Object(); 
        // 用一个短弱引用对象弱引用o.
        WeakReference wr = new WeakReference(o);
     
        o = null; // 移除对象的强引用
     
        o = wr.Target; //尝试从弱引用对象中获得对象的强引用
        if (o == null) {
            // 如果对象为空说明对象已经被垃圾回收器回收掉了
        } else {
            // 如果垃圾回收器还没有回收此对象就可以继续使用对象了
        }
    }

为什么需要弱对象呢？因为，有一些数据创建起来很容易，但是却需要很多内存。例如：你有一个程序，这个程序需要访问用户硬盘上的所有文件夹和文件名；你可以在程序第一次需要这个数据时访问用户磁盘生成一次数据，数据生成之后你就可以访问内存中的数据来得到用户文件数据，而不是每次都去读磁盘获得数据，这样做可以提升程序的性能。

问题是这个数据可能相当大，需要相当大的内存。如果用户去操作程序的另外一部分功能了，这块相当大的内存就没有占用的必要了。你可以通过代码删除这些数据，但是如果用户马上切换到需要这块数据的功能上，你就必须重新从用户的磁盘上构建这个数据。弱引用为这种场景提供了一种简单有效的方案。

当用户切换到其他功能时，你可以为这个数据创建一个弱引用对象，并把对这个数据的强引用解除掉。这样如果程序占用的内存很低，垃圾回收操作就不会触发，弱引用对象就不会被回收掉；这样当程序需要使用这块数据时就可以通过一个强引用来获得数据，如果成功得到了对象引用，程序就没有必要再次读取用户的磁盘了。

WeakReference类型提供了两个构造函数：

    WeakReference(object target);
    WeakReference(object target, bool trackResurrection);

target参数显然就是弱引用要跟踪的对象了。trackResurrection参数表示当对象的Finalize方法执行之后是否还要跟踪这个对象。默认这个参数是false。有关对象的复活请参考[这里](http://blog.jobbole.com/31443/)。

方便起见，不跟踪复活对象的弱引用称为“短弱引用”；而要跟踪复活对象的的弱引用称为“长弱引用”。如果对象没有实现Finalize方法，那么长弱引用和短弱引用是完全一样的。强烈建议你尽量避免使用长弱引用。长弱引用允许你使用复活的对象，而复活对象的行为可能是不可以预知的。

一旦你使用WeakReference引用了一个对象，建议你将这个对象的所有强用都设置为null；如果强引用存在的话，垃圾回收器是永远都不可能回收弱引用指向的对象的。

当你要使用弱引用目标对象时，你必须为目标对象创建一个强引用，这很简单，只要用object a = weekRefer.Target;就可以了，然后你必须判断a是否为空，弱不为空才可以继续使用，弱为空就表示对象已经被垃圾回收器回收了，得通过其他方法重新获得此对象。

**弱引用的内部实现**

从前文中的描述中我们可以推断出弱引用对象肯定和一般对象的处理是不一样的。一般情况下如果一个对象引用了另一个对象就是强引用，垃圾回收器就不能回收被引用的对象，而WeakReference对象却不是这样子，它引用的对象是有可能被回收的。

要完全理解弱对象是如何工作的，我们还需要看一下托管堆。托管堆上有两个内部数据结构他们的唯一作用是管理弱引用：我们可以把它们称作长弱引用表和短弱引用表；这两个表存放托管堆上的弱引用目标对象指针。

程序运行之初，这两个表都是空的。当你创建一个WeakReference对象时，这个对象并不是分配到托管堆上的，而是在弱对象表中创建一个空槽（Empty Slot）。短弱引用对象被放在短弱对象表中，长弱引用对象被放在长弱引用表中。

一旦发现空槽，空槽的值会被设置成弱引用目标对象的地址；显然长短弱对象表中的对象是不会当作应用程序的根对象的。垃圾回收器不会回收长短弱对象表中的数据。

让我们来看下垃圾回收执行时发生了什么：

​1.垃圾回收器构建一个可达对象图，构建步骤请[参考上文](http://blog.jobbole.com/31443/)

​2.垃圾回收器扫描短弱对象表，如果弱对象表中指向的对像没有在可达对象图中，那么这个对像就被标识为垃圾对象，然后短对象表中的对象指针被设置为空

​3.垃圾回收器扫描终结队列（[参考上文](http://blog.jobbole.com/31443/)），如果队列中的对象不在可达对象图中，这个对象从终结队列中移动到Freachable队列中，这时候，这个对象又被标识为可达对象，不再是垃圾了

​4.垃圾回收器扫描长弱引用表。如果表中的对象不在可达对象图中（可达对象图中包括在Freachable队列中对象），将长引用对象表中对应的对象指针设置为null

​5. 垃圾回收器移动可达对象

一旦你理解了垃圾回收器的工作过程，就很容易理解弱引用是如何起作用了。访问WeakReference的Target属性导致系统返回弱对象表中的目标对象指针，如果是null，表示对象已经被回收了。

短弱引用不跟踪复活，这意味着垃圾回收器可以在扫描终结队列之前检查弱引用表中指向的对象是否是垃圾对象。

而长弱引用跟踪复活对象，这意味着垃圾回收器必须在确认对象回收之后才可以将弱引用表中的指针设置为null。

**代：**

提起.Net的垃圾回收，c++或者c[程序员](http://blog.jobbole.com/821/ "程序员的本质")可能就会想，这么管理内存会不会出现性能问题呢。GC的开发人员一直在调整垃圾回收器提升它的性能。代就是一种为了降低垃圾回收对性能影响的机制。垃圾回收器在工作时会假定如下说法是成立的：

​1. 一个对象越新，那么这个对象的生命周期就越短

​2. 一个对象越老，那么这个对象的生命周期就越长

​3. 新对象之间通常更可能和新对象之间存在引用关系

​4. 压缩堆的一部分要比压缩整个堆要快

当然大量研究证明以上几个假设在很多程序上是成立的。那就让我们来谈谈这几个假设是如何影响垃圾回收器工作的吧。

在程序初始化时，托管堆上没有对象。这时候新添到托管堆上的对象是的代是0.如下图所示，0代对象是最年轻的对象，他们从来没有经过垃圾回收器的检查。

[![.Net垃圾回收机制原理（二）](/post-images/2011-11/NET-GC-07.gif ".Net 垃圾回收机制原理（二）")](/post-images/2011-11/NET-GC-07.gif ".Net 垃圾回收机制原理（二）")

图1 托管堆上的0代对象

现在如果堆上添加了更多的对象，堆填满时就会触发垃圾回收。当垃圾回收器分析托管堆时，会构建一个垃圾对象（图2中浅紫色块）和非垃圾对象的图。所有没有被回收的对象会被移动压缩到堆的最底端。这些没有被回收掉的对象就成为了1代对象，如图2所示

[![.Net垃圾回收机制原理（二）](/post-images/2011-11/NET-GC-08.gif ".Net 垃圾回收机制原理（二）")](/post-images/2011-11/NET-GC-08.gif ".Net 垃圾回收机制原理（二）")

图2 托管堆上的0代1代对象

当堆上分配了更多的对象时，新对象被放在了0代区。如果0代堆填满了，就会触发一次垃圾回收。这时候活下来的对象成为1代对象被移动到堆的底部；再此发生垃圾回收后1代对象中存活下来的对象会提升为2代对象并被移动压缩。如图3所示：

[![.Net
垃圾回收机制原理（二）](/post-images/2011-11/NET-GC-09.gif ".Net 垃圾回收机制原理（二）")](/post-images/2011-11/NET-GC-09.gif ".Net 垃圾回收机制原理（二）")

图3 托管堆上的0、1、2代对象

2代对象是目前垃圾回收器的最高代，当再次垃圾回收时，没有回收的对象的代数依然保持2.

**垃圾回收分代为什么可以优化性能**

如前所述，分代回收可以提高性能。当堆填满之后会触发垃圾回收，垃圾回收器可以只选择0代上的对象进行回收，而忽略更高代堆上的对象。然而，由于越年轻的对象生命周期越短，因此，回收0代堆可以回收相当多的内存，而且回收所耗的性能也比回收所有代对象要少得多。

这是分代垃圾回收的最简单优化。分代回收不需要便利整个托管堆，如果一个根对象引用了一个高代对象，那么垃圾回收器可以忽略高代对象和其引用对象的遍历，这会大大减少构建可达对象图的时间。

如果回收0代对象没有释放出足够的内存，垃圾回收器会尝试回收1代和0代堆；如果仍然没有获得足够的内存，那么垃圾回收器会尝试回收2，1，0代堆。具体会回收那一代对象的算法不是确定的，微软会持续做算法优化。

多数堆（像c-runtime堆）只要找到足够的空闲内存就分配给对象。因此，如果我连续分配多个对象时，这些对象的地址空间可能会相差几M。然而在托管堆上，连续分配的对象的内存地址是连续的。

前面的假设中还提到，新对象之间更可能存在相互引用关系。因此新对象分配到连续的内存上，你可以获得就近引用的性能优化（you gain performance from locality of reference）。这样的话很可能你的对象都在CPU的缓存中，这样CPU的很多操作就不需要去存取内存了。

微软的性能测试显示托管堆的分配速度比标准的win32 HeapAlloc方法还要快。这些测试也显示了200MHz的Pentium的CPU做一次0代回收时间可以小于1毫秒。微软的优化目的是让垃圾回收耗用的时间小于一次普通的页面错误。

**使用System.GC类控制垃圾回收**

类型System.GC运行开发人员直接控制垃圾回收器。你可以通过GC.MaxGeneration属性获得GC的最高代数，目前最高代是定值2.

你可以调用GC.Collect()方法强制垃圾回收器做垃圾回收，Collect方法有两个重载：

    void GC.Collect(Int32 generation)
     
    void GC.Collect()

第一个方法允许你指定要回收那一代。你可以传0到GC.MaxGeneration的数字做参数，传0只做0代堆的回收，传1会回收1代和0代堆，而传2会回收整个托管堆。而无参数的方法调用GC.Collect(GC.MaxGeneration)相当于整个回收。

在通常情况下，不应该去调用GC.Collect方法；最好让垃圾回收器按照自己的算法判断什么时候该调用Collect方法。尽管如此，如果你确信比运行时更了解什么时候该做垃圾回收，你就可以调用Collect方法去做回收。比如说程序可以在保存数据文件之后做一次垃圾回收。比如你的程序刚刚用完一个长度为10000的大数组，你不再需要他了，就可以把它设置为null然后执行垃圾回收，缓解内存的压力。

GC还提供了WaitForPendingFinalizers方法。这个方法简单的挂起执行线程，知道Freachable队列中的清空之后，执行完所有队列中的Finalize方法之后才继续执行。

GC还提供了两个方法用来返回某个对象是几代对象，他们是

Int32 GC.GetGeneration(object o);

Int32 GC.GetGeneration(WeakReference wr)

第一个方法返回普通对象是几代，第二个方法返回弱引用对象的代数。

下面的代码可以帮助你理解代的意义：

    private static void GenerationDemo() {
      // Let's see how many generations the GCH supports (we know it's 2)
      Display("Maximum GC generations: " + GC.MaxGeneration);
     
      // Create a new BaseObj in the heap
      GenObj obj = new GenObj("Generation");
     
      // Since this object is newly created, it should be in generation 0
      obj.DisplayGeneration(); // Displays 0
     
      // Performing a garbage collection promotes the object's generation
      GC.Collect();
      obj.DisplayGeneration(); // Displays 1
     
      GC.Collect();
      obj.DisplayGeneration(); // Displays 2
     
      GC.Collect();
      obj.DisplayGeneration(); // Displays 2 (max generation)
     
      obj = null; // Destroy the strong reference to this object
     
      GC.Collect(0); // Collect objects in generation 0
      GC.WaitForPendingFinalizers(); // We should see nothing
     
      GC.Collect(1); // Collect objects in generation 1
      GC.WaitForPendingFinalizers(); // We should see nothing
     
      GC.Collect(2); // Same as Collect()
      GC.WaitForPendingFinalizers(); // Now, we should see the Finalize 
      // method run
     
      Display(-1, "Demo stop: Understanding Generations.", 0);
    }
    class GenObj{
      public void DisplayGeneration(){
        Console.WriteLine(“my generation is ” + GC.GetGeneration(this));
      }
     
      ~GenObj(){
        Console.WriteLine(“My Finalize method called”);
      }
    }

**垃圾回收机制的多线程性能优化**

在前面的部分，我解释了GC的算法和优化，然后讨论的前提都是在单线程情况下的。而在真实的程序中，很可能是多个线程一起工作，多个线程一起操纵托管堆上的对象。当一个线程触发了垃圾回收，其他所有的线程都应该暂停访问任何引用对象（包括他们自己栈上引用的对象），因为垃圾回收器有可能要移动对象，修改对象的内存地址。

因此当垃圾回收器开始回收时，所有执行托管代码的线程必须挂起。运行时有几种不同的机制可以安全的挂起线程来执行垃圾回收。这一块的内部机制我不打算详细说明。但是微软会持续修改垃圾回收的机制来降低垃圾回收带来的性能损耗。

下面几段描述了垃圾回收器在多线程情况下是如何工作的：

完全中断代码执行：当垃圾回收开始执行时，挂起所有应用程序线程。垃圾回收器随后将线程挂起的位置记录到一个just-in-time(JIT)编译器生成的表中，垃圾回收器负责将线程挂起的位置记录在表中，记录当前正在访问的对象，以及对象存放的位置（变量中，CPU寄存器中，等等）

劫持：垃圾回收器可以修改线程的栈让返回地址指向一个特殊的方法，当当前执行的方法返回时，这个特殊的方法将会执行，挂起线程，这种改变线程执行路径的方式称为劫持线程。当垃圾回收完成之后，线程会重新返回到之前执行的方法上。

安全点：
当JIT编译器编译一个方法时，可以在某个点插入一段代码判断GC是否挂起，如果是，线程就挂起等待垃圾回收完成，然后线程重新开始执行。JIT编译器插入检查GC代码的位置被称作“安全点”

请注意，线程劫持允许正在执行非托管代码的线程在垃圾回收过程中执行。如果非托管代码不访问托管堆上的对象时这是没有问题的。如果这个线程当前执行非托管代码然后返回执行托管代码，这个线程将会被劫持，直到垃圾回收完成之后再继续执行。

除了我刚提到的集中机制之外，垃圾回收器还有其他改进来增强多线程程序中的对象内存分配和回收。

同步释放分配（Synchronization-free Allocations）：在一个多线程系统中，0代堆被分成几个区域，一个线程使用一个区域。这允许多线程同时分配对象，并不需要一个线程独占堆。

可伸缩回收(Scalable Collections)：在多线程系统中运行执行引擎的服务器版本（MXSorSvr.dll）.托管堆会被分成几个不同的区域，一个CPU一个区域。当回收初始化时，每个CPU执行一个回收线程，各个线程回收各自的区域。而工作站版本的执行引擎（MXCorWks.dll）不支持这个功能。

大对象回收

这一块就不翻译了，有一篇[专门的文章](http://blog.jobbole.com/31459/)谈这件事儿

**监视垃圾回收**

如果你安装了.Net framework你的性能计数器（开始菜单—管理工具—性能
进入）中就会有.Net CLR
Memory一项，你可以从实例列表中选择某个程序进行观察，如下图所示。

[![.Net 垃圾回收机制原理（二）](/post-images/2011-11/NET-GC-10.gif ".Net 垃圾回收机制原理（二）")](/post-images/2011-11/NET-GC-10.gif ".Net 垃圾回收机制原理（二）")

这些性能指标的具体含义如下：

<table border="1" cellspacing="0" cellpadding="0">
<tbody>
<tr>
<td valign="top">
<p align="left">性能计数器</p>
</td>
<td valign="top">
<p align="left">说明</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># Bytes in all Heaps</strong><strong>（所有堆中的字节数）</strong></p>
</td>
<td valign="top">
<p align="left">显示以下计数器值的总和：“第 0 级堆大小”计数器、“第 1 级堆大小”计数器、“第 2 级堆大小”计数器和“大对象堆大小”计数器。此计数器指示在垃圾回收堆上分配的当前内存（以字节为单位）。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># GC Handles</strong><strong>（</strong><strong>GC&nbsp;</strong><strong>处理数目）</strong></p>
</td>
<td valign="top">
<p align="left">显示正在使用的垃圾回收处理的当前数目。垃圾回收处理是对公共语言运行库和托管环境外部的资源的处理。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># Gen 0 Collections</strong><strong>（第</strong><strong>&nbsp;2&nbsp;</strong><strong>级回收次数）</strong></p>
</td>
<td valign="top">
<p align="left">显示自应用程序启动后第 0 级对象（即最年轻、最近分配的对象）被垃圾回收的次数。</p>
<p align="left">当第 0 级中的可用内存不足以满足分配请求时发生第 0 级垃圾回收。此计数器在第 0 级垃圾回收结束时递增。较高级的垃圾回收包括所有较低级的垃圾回收。当较高级（第 1 级或第 2 级）垃圾回收发生时此计数器被显式递增。</p>
<p align="left">此计数器显示最近的观察所得值。<strong>_Global_</strong>&nbsp;计数器值不准确，应该忽略。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># Gen 1 Collections</strong><strong>（第</strong><strong>&nbsp;2&nbsp;</strong><strong>级回收次数）</strong></p>
</td>
<td valign="top">
<p align="left">显示自应用程序启动后对第 1 级对象进行垃圾回收的次数。</p>
<p align="left">此计数器在第 1 级垃圾回收结束时递增。较高级的垃圾回收包括所有较低级的垃圾回收。当较高级（第 2 级）垃圾回收发生时此计数器被显式递增。</p>
<p align="left">此计数器显示最近的观察所得值。<strong>_Global_</strong>&nbsp;计数器值不准确，应该忽略。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># Gen 2 Collections</strong><strong>（第</strong><strong>&nbsp;2&nbsp;</strong><strong>级回收次数）</strong></p>
</td>
<td valign="top">
<p align="left">显示自应用程序启动后对第 2 级对象进行垃圾回收的次数。此计数器在第 2 级垃圾回收（也称作完整垃圾回收）结束时递增。</p>
<p align="left">此计数器显示最近的观察所得值。<strong>_Global_</strong>&nbsp;计数器值不准确，应该忽略。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># Induced GC</strong><strong>（引发的</strong><strong>&nbsp;GC&nbsp;</strong><strong>的数目）</strong></p>
</td>
<td valign="top">
<p align="left">显示由于对&nbsp;<a href="http://msdn.microsoft.com/zh-cn/library/system.gc.collect(v=vs.80).aspx" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://msdn.microsoft.com']);">GC.Collect</a>&nbsp;的显式调用而执行的垃圾回收的峰值次数。让垃圾回收器对其回收的频率进行微调是切实可行的。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># of Pinned Objects</strong><strong>（钉住的对象的数目）</strong></p>
</td>
<td valign="top">
<p align="left">显示上次垃圾回收中遇到的钉住的对象的数目。钉住的对象是垃圾回收器不能移入内存的对象。此计数器只跟踪被进行垃圾回收的堆中的钉住的对象。例如，第 0 级垃圾回收导致仅枚举第 0 级堆中钉住的对象。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># of Sink Blocks in use</strong><strong>（正在使用的接收块的数目）</strong></p>
</td>
<td valign="top">
<p align="left">显示正在使用的同步块的当前数目。同步块是为存储同步信息分配的基于对象的数据结构。同步块保留对托管对象的弱引用并且必须由垃圾回收器扫描。同步块不局限于只存储同步信息；它们还可以存储 COM interop 元数据。该计数器指示与同步基元的过度使用有关的性能问题。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># Total committed Bytes</strong><strong>（提交字节的总数）</strong></p>
</td>
<td valign="top">
<p align="left">显示垃圾回收器当前提交的虚拟内存量（以字节为单位）。提交的内存是在磁盘页面文件中保留的空间的物理内存。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong># Total reserved Bytes</strong><strong>（保留字节的总数）</strong></p>
</td>
<td valign="top">
<p align="left">显示垃圾回收器当前保留的虚拟内存量（以字节为单位）。保留内存是为应用程序保留（但尚未使用任何磁盘或主内存页）的虚拟内存空间。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>% Time in GC</strong><strong>（</strong><strong>GC&nbsp;</strong><strong>中时间的百分比）</strong></p>
</td>
<td valign="top">
<p align="left">显示自上次垃圾回收周期后执行垃圾回收所用运行时间的百分比。此计数器通常指示垃圾回收器代表该应用程序为收集和压缩内存而执行的工作。只在每次垃圾回收结束时更新此计数器。此计数器不是一个平均值；它的值反映了最近观察所得值。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Allocated Bytes/second</strong><strong>（每秒分配的字节数）</strong></p>
</td>
<td valign="top">
<p align="left">显示每秒在垃圾回收堆上分配的字节数。此计数器在每次垃圾回收结束时（而不是在每次分配时）进行更新。此计数器不是一段时间内的平均值；它显示最近两个样本中观测的值的差除以取样间隔时间所得的结果。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Finalization Survivors</strong><strong>（完成时存留对象数目）</strong></p>
</td>
<td valign="top">
<p align="left">显示因正等待完成而从回收后保留下来的进行垃圾回收的对象的数目。如果这些对象保留对其他对象的引用，则那些对象也保留下来，但此计数器不对它们计数。“从第 0 级提升的完成内存”和“从第 1 级提升的完成内存”计数器表示因完成而保留下来的所有内存。</p>
<p align="left">此计数器不是累积计数器；它在每次垃圾回收结束时由仅在该特定回收期间存留对象的计数更新。此计数器指示由于完成应用程序可能导致系统开销过高。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Gen 0 heap size</strong><strong>（第</strong><strong>&nbsp;2&nbsp;</strong><strong>级堆大小）</strong></p>
</td>
<td valign="top">
<p align="left">显示在第 0 级中可以分配的最大字节数；它不指示在第 0 级中当前分配的字节数。</p>
<p align="left">当自最近回收后的分配超出此大小时发生第 0 级垃圾回收。第 0 级大小由垃圾回收器进行微调并且可在应用程序执行期间更改。在第 0 级回收结束时，第 0 级堆的大小是 0 字节。此计数器显示调用下一个第 0 级垃圾回收的分配的大小（以字节为单位）。</p>
<p align="left">此计数器在垃圾回收结束时（而不是在每次分配时）进行更新。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Gen 0 Promoted Bytes/Sec</strong><strong>（从第</strong><strong>&nbsp;1&nbsp;</strong><strong>级提升的字节数</strong><strong>/</strong><strong>秒）</strong></p>
</td>
<td valign="top">
<p align="left">显示每秒从第 0 级提升到第 1 级的字节数。内存在从垃圾回收保留下来后被提升。此计数器是每秒创建的在相当长时间保留下来的对象的指示符。</p>
<p align="left">此计数器显示在最后两个样本（以取样间隔持续时间来划分）中观察到的值之间的差异。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Gen 1 heap size</strong><strong>（第</strong><strong>&nbsp;2&nbsp;</strong><strong>级堆大小）</strong></p>
</td>
<td valign="top">
<p align="left">显示第 1 级中的当前字节数；此计数器不显示第 1 级的最大大小。不直接在此代中分配对象；这些对象是从前面的第 0 级垃圾回收提升的。此计数器在垃圾回收结束时（而不是在每次分配时）进行更新。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Gen 1 Promoted Bytes/Sec</strong><strong>（从第</strong><strong>&nbsp;1&nbsp;</strong><strong>级提升的字节数</strong><strong>/</strong><strong>秒）</strong></p>
</td>
<td valign="top">
<p align="left">显示每秒从第 1 级提升到第 2 级的字节数。在此计数器中不包括只因正等待完成而被提升的对象。</p>
<p align="left">内存在从垃圾回收保留下来后被提升。不会从第 2 级进行任何提升，因为它是最旧的一级。此计数器是每秒创建的非常长时间保留下来的对象的指示符。</p>
<p align="left">此计数器显示在最后两个样本（以取样间隔持续时间来划分）中观察到的值之间的差异。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Gen 2 heap size</strong><strong>（第</strong><strong>&nbsp;2&nbsp;</strong><strong>级堆大小）</strong></p>
</td>
<td valign="top">
<p align="left">显示第 2 级中当前字节数。不直接在此代中分配对象；这些对象是在以前的第 1 级垃圾回收期间从第 1 级提升的。此计数器在垃圾回收结束时（而不是在每次分配时）进行更新。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Large Object Heap size</strong><strong>（大对象堆大小）</strong></p>
</td>
<td valign="top">
<p align="left">显示大对象堆的当前大小（以字节为单位）。垃圾回收器将大于 20 KB 的对象视作大对象并且直接在特殊堆中分配大对象；它们不是通过这些级别提升的。此计数器在垃圾回收结束时（而不是在每次分配时）进行更新。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Promoted Finalization-Memory from Gen 0</strong><strong>（从第</strong><strong>&nbsp;1&nbsp;</strong><strong>级提升的完成内存）</strong></p>
</td>
<td valign="top">
<p align="left">显示只因等待完成而从第 0 级提升到第 1 级的内存的字节数。此计数器不是累积计数器；它显示在最后一次垃圾回收结束时观察到的值。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Promoted Finalization-Memory from Gen 1</strong><strong>（从第</strong><strong>&nbsp;1&nbsp;</strong><strong>级提升的完成内存）</strong></p>
</td>
<td valign="top">
<p align="left">显示只因等待完成而从第 1 级提升到第 2 级的内存的字节数。此计数器不是累积计数器；它显示在最后一次垃圾回收结束时观察到的值。如果最后一次垃圾回收就是第 0 级回收，此计数器则重置为 0。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Promoted Memory from Gen 0</strong><strong>（从第</strong><strong>&nbsp;1&nbsp;</strong><strong>级提升的内存）</strong></p>
</td>
<td valign="top">
<p align="left">显示在垃圾回收后保留下来并且从第 0 级提升到第 1 级的内存的字节数。此计数器中不包括那些只因等待完成而提升的对象。此计数器不是累积计数器；它显示在最后一次垃圾回收结束时观察到的值。</p>
</td>
</tr>
<tr>
<td valign="top">
<p align="left"><strong>Promoted Memory from Gen 1</strong><strong>（从第</strong><strong>&nbsp;1&nbsp;</strong><strong>级提升的内存）</strong></p>
</td>
<td valign="top">
<p align="left">显示在垃圾回收后保留下来并且从第 1 级提升到第 2 级的内存的字节数。此计数器中不包括那些只因等待完成而提升的对象。此计数器不是累积计数器；它显示在最后一次垃圾回收结束时观察到的值。如果最后一次垃圾回收就是第 0 级回收，此计数器则重置为 0。</p>
</td>
</tr>
</tbody>
</table>

这个表来自[MSDN](http://msdn.microsoft.com/zh-cn/library/x2tyfybc(v=vs.80).aspx)

 
