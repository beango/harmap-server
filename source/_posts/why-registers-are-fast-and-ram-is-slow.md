---
layout: post
category: default
date: 2013-10-11
title: "为什么寄存器比内存快？"
description: "为什么寄存器比内存快？"
tags: [寄存器]
redirecturl: http://www.ruanyifeng.com/blog/2013/10/register.html
---


原文出处： [Mike Ash](http://www.mikeash.com/pyblog/friday-qa-2013-10-11-why-registers-are-fast-and-ram-is-slow.html)   译文出处：[阮一峰](http://www.ruanyifeng.com/blog/2013/10/register.html)


计算机的[存储层次](http://zh.wikipedia.org/wiki/%E5%AD%98%E5%82%A8%E5%B1%82%E6%AC%A1)（memory hierarchy）之中，[寄存器](http://zh.wikipedia.org/wiki/%E5%AF%84%E5%AD%98%E5%99%A8)（register）最快，内存其次，最慢的是硬盘。

[![20131015105220](/post-images/2013-10/20131015105220.jpg)](/post-images/2013-10/20131015105220.jpg "为什么寄存器比内存快？")

同样都是晶体管存储设备，为什么寄存器比内存快呢？

[![2013101402](/post-images/2013-10/2013101402.jpg)](/post-images/2013-10/2013101402.jpg "为什么寄存器比内存快？")

[Mike Ash](http://www.mikeash.com/pyblog/friday-qa-2013-10-11-why-registers-are-fast-and-ram-is-slow.html)写了一篇很好的解释，非常通俗地回答了这个问题，有助于加深对硬件的理解。下面就是我的简单翻译。

**原因一：距离不同**

距离不是主要因素，但是最好懂，所以放在最前面说。内存离CPU比较远，所以要耗费更长的时间读取。

以3GHz的CPU为例，电流每秒钟可以振荡30亿次，每次耗时大约为0.33[纳秒](http://en.wikipedia.org/wiki/Nanosecond)。光在1纳秒的时间内，可以前进30厘米。也就是说，在CPU的一个[时钟周期](http://zh.wikipedia.org/wiki/%E6%97%B6%E9%92%9F%E9%A2%91%E7%8E%87)内，光可以前进10厘米。因此，如果内存距离CPU超过5厘米，就不可能在一个时钟周期内完成数据的读取，这还没有考虑硬件的限制和电流实际上达不到光速。相比之下，寄存器在CPU内部，当然读起来会快一点。

距离对于桌面电脑影响很大，对于手机影响就要小得多。手机CPU的时钟频率比较慢（iPhone 5s为1.3GHz），而且手机的内存紧挨着CPU。

**原因二：硬件设计不同**

苹果公司新推出的iPhone 5s，CPU是[A7](http://en.wikipedia.org/wiki/Apple_A7)，寄存器有6000多位（31个64位寄存器，加上32个128位寄存器）。而iPhone 5s的内存是1GB，约为80亿位（bit）。这意味着，高性能、高成本、高耗电的设计可以用在寄存器上，反正只有6000多位，而不能用在内存上。因为每个位的成本和能耗只要增加一点点，就会被放大80亿倍。

[![2013101403](/post-images/2013-10/20131014031.jpg)](/post-images/2013-10/20131014031.jpg "为什么寄存器比内存快？")

事实上确实如此，内存的设计相对简单，每个位就是一个电容和一个晶体管，而寄存器的[设计](http://en.wikipedia.org/wiki/Register_file#Array)则完全不同，多出好几个电子元件。并且通电以后，寄存器的晶体管一直有电，而内存的晶体管只有用到的才有电，没用到的就没电，这样有利于省电。这些设计上的因素，决定了寄存器比内存读取速度更快。

**原因三：工作方式不同**

寄存器的工作方式很简单，只有两步：（1）找到相关的位，（2）读取这些位。

内存的工作方式就要复杂得多：

> （1）找到数据的指针。（指针可能存放在寄存器内，所以这一步就已经包括寄存器的全部工作了。）
>
> （2）将指针送往[内存管理单元](http://zh.wikipedia.org/wiki/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%8D%95%E5%85%83)（MMU），由MMU将虚拟的内存地址翻译成实际的物理地址。
>
> （3）将物理地址送往内存控制器（[memory
> controller](http://en.wikipedia.org/wiki/Memory_controller)），由内存控制器找出该地址在哪一根内存插槽（bank）上。
>
> （4）确定数据在哪一个内存块（chunk）上，从该块读取数据。
>
> （5）数据先送回内存控制器，再送回CPU，然后开始使用。

内存的工作流程比寄存器多出许多步。每一步都会产生延迟，累积起来就使得内存比寄存器慢得多。

为了缓解寄存器与内存之间的巨大速度差异，硬件设计师做出了许多努力，包括在CPU内部设置[缓存](http://zh.wikipedia.org/wiki/CPU%E7%BC%93%E5%AD%98)、优化CPU工作方式，尽量一次性从内存读取指令所要用到的全部数据等等。
