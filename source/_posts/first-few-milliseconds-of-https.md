---
layout: post
category: default
date: 2013-09-21
title: "HTTPS连接的前几毫秒发生了什么"
description: "HTTPS连接的前几毫秒发生了什么"
tags: [HTTPS]
redirecturl: http://blog.jobbole.com/48369/
---


花了数小时阅读了如潮的好评，Bob最终迫不及待为他购买的托斯卡纳全脂牛奶点击了“进行结算”，然后……

哇！刚刚发生了什么?

![](/post-images/2013-09/63918611gw1e8uafpk13uj207002cglj.jpg)

在点击按钮过后的220毫秒时间内，发生了一系列有趣的事情，火狐浏览器（Firefox）不仅改变了地址栏颜色，而且在浏览器的右下角出现了一个小锁头的标志。在我最喜欢的互联网工具Wireshark的帮助下，我们可以通过一个经过略微调整的用于debug的火狐浏览器来探究这一过程。

根据RFC 2818标准（译者注：RFC 2818为HTTP Over TLS-网络协议），火狐浏览器自动通过连接Amazon.com的443端口来响应HTTPS请求。

![](/post-images/2013-09/63918611gw1e8uafqg1ynj20b403taaj.jpg)

很多人会把HTTPS和网景公司（Netscape）于上世纪九十年代中期创建的SSL（安全套接层）联系起来。事实上，随着时间的推移，这两者之间的关系也慢慢淡化。随着网景公司渐渐的失去市场份额，SSL的维护工作移交给了Internet工程任务组（IETF）。由网景公司发布的第一个版本被重新命名为TLS1.0（安全传输层协议1.0），并于1999年1月正式发布。考虑到TLS已经发布了将近10年，如今已经很难再见到真正的SSL通信了。

客户端问候（Client Hello）
------------------------

TLS将全部的通信以不同方式包裹为“记录”（Records）。我们可以看到，从浏览器发出的第一个字节为0×16（十进制的22），它表示了这是一个“握手”记录。

![](/post-images/2013-09/63918611gw1e8uafrxaddj20b403baac.jpg)

接下来的两个字节是0×0301，它表示了这是一条版本为3.1的记录，同时也向我们表明了TLS1.0实际上是基于SSL3.1构建而来的。

整个握手记录被拆分为数条信息，其中第一条就是我们的客户端问候（ClientHello），即0×01。在客户端问候中，有几个需要着重注意的地方：

-    随机数：

![](/post-images/2013-09/63918611gw1e8uafsvt2cj20b4022mxd.jpg)

在客户端问候中，有四个字节以Unix时间格式记录了客户端的协调世界时间（UTC）。协调世界时间是从1970年1月1日开始到当前时刻所经历的秒数。在这个例子中，0x4a2f07ca就是协调世界时间。在他后面有28字节的随机数，在后面的过程中我们会用到这个随机数。

-   SID（Session ID）：

![](/post-images/2013-09/63918611gw1e8uafu3wcij20b400tt8m.jpg)

在这里，SID是一个空值（Null）。如果我们在几秒钟之前就登陆过了Amazon.com，我们有可能会恢复之前的会话，从而避免一个完整的握手过程。

-   密文族（Cipher Suites）：

![](/post-images/2013-09/63918611gw1e8uafv0zu5j20b403aq3h.jpg)

密文族是浏览器所支持的加密算法的清单。整个密文族是由推荐的加密算法“TLS\_ECDHE\_ECDSA\_WITH\_AES\_256\_CBC\_SHA”和33种其他加密算法所组成。别担心其他的加密算法会出现问题，我们一会儿就会发现Amazon也没有使用推荐的加密算法。

-    Server\_name扩展：

![](/post-images/2013-09/63918611gw1e8uafvtru3j20b401caa4.jpg)

通过这种方式，我们能够告诉Amazon.com：浏览器正在试图访问[https://www.amazon.com](https://www.amazon.com/)。这确实方便了很多，因为我们的TLS握手时间发生在HTTP通信之前，而HTTP请求会包含一个“Host头”，从而使那些为了节约成本而将数百个网站域名解析到一个IP地址上的网络托管商能够分辨出一个网络请求对应的是哪个网站。传统意义上的SSL同样要求一个网站请求对应一个IP地址，但是Server\_name扩展则允许服务器对浏览器的请求授予相对应的证书。如果没有其他的请求，Server\_name扩展应该允许浏览器访问这个IPV4地址一周左右的时间。

 

服务器问候（Server Hello）
------------------------

Amazon.com回复的握手记录由两个比较大的包组成（2551字节）。记录中包含了0×0301的版本信息，意味着Amazon同意我们使用TLS1.0访问的请求。这条记录包含了三条有趣的子信息：

**1.服务器问候信息（Server Hello）(2):**

![](/post-images/2013-09/63918611gw1e8uafwsvx9j20b4057dgj.jpg)

1.  我们得到了服务器的以Unix时间格式记录的UTC和28字节的随机数。
2.  32字节的SID，在我们想要重新连接到Amazon.com的时候可以避免一整套握手过程。
3.  在我们所提供的34个加密族中，Amazon挑选了“TLS\_RSA\_WITH\_RC4\_128\_MD5”（0×0004）。这就意味着Amazon会使用RSA公钥加密算法来区分证书签名和交换密钥，通过RC4加密算法来加密数据，利用Md5来校验信息。我们之后会深入的研究这一部分内容。我个人认为，Amazon选择这一密码组是有其自身的原因的。在我们所提供的密码族中，这一加密组的加密方式是CPU占用最低的，这就允许Amazon的每台服务器接受更多的连接。当然了，也许还有一个原因是，Amazon是在向这三种加密算法的发明者Ron Rivest（罗恩·李·维斯特）致敬。

**2.证书信息（11）：**

![](/post-images/2013-09/63918611gw1e8uafxsvrjj20b405vjs7.jpg)

这段巨大的信息共有2464字节，其证书允许客户端在Amazon服务器上进行认证。这个证书其实并没有什么奇特之处，你能通过浏览器浏览它的大部分内容。

![](/post-images/2013-09/63918611gw1e8uafz3g7cj20b40a1dgl.jpg)

**3.服务器问候结束信息（14）：**

![](/post-images/2013-09/63918611gw1e8uafzsnfjj20b401o3yi.jpg)

这是一个零字节信息，用于告诉客户端整个“问候”过程已经结束，并且表明服务器不会再向客户端询问证书。

校验证书
--------

此时，浏览器已经知道是否应该信任Amazon.com。在这个例子中，浏览器通过证书确认网站是否受信，它会检查Amazon.com的证书，并且确认当前的时间是在“最早时间”2008年8月26日之后，在“最晚时间”2009年8月27日之前。浏览器还会确认证书所携带的公共密钥已被授权用于交换密钥。

为什么我们要信任这个证书？

证书中所包含的签名是一串非常长的大端格式的数字：

![](/post-images/2013-09/63918611gw1e8uag3onewj20b401mq31.jpg)

任何人都可以向我们发送这些字节，但我们为什么要信任这个签名？为了解释这个问题，我们首先要回顾一些重要的数学知识：

RSA加密算法的基础介绍
---------------------

人人常常会问，编程和数学之间有什么联系？证书就为数学在编程领域的应用提供了一个实际的例子。Amazon的服务器告诉我们需要使用RSA算法来校验证书签名。什么又是RSA算法呢？RSA算法是由麻省理工（MIT）的Ron Rivest、Adi Shamirh和Len Adleman（RSA命名各取了三人名字中的首字母）三人于上世纪70年代创建的。三位天才的学者结合了2000多年数学史上的精华，发明了这种简洁高效的算法：

选取两个较大的初值p和q，相乘得n；**n = p\*q **接下来选取一个较小的数作为加密指数e，d作为解密指数是e的倒数。在加密的过程中，n和e是公开信息，解密密钥d则是最高机密。至于p和q，你可以将他们公开，也可以作为机密保管。但是一定要记住，e和d是互为倒数的两个数。

假设你现在有一段信息M（转换成数字），将其加密只需要进行运算：C ≡ M^e (mod n)

这个公式表示M的e次幂，mod n表示除以n取余数。当这段密文的接受者知道解密指数d的时候就可以将密文进行还原：C^d ≡ ( M^e )^d ≡ M^e*d ≡ M^1 ≡ M(mod n)

有趣的是，解密指数d的持有者还可以将信息M进行用解密指数d进行加密：M^d ≡ S(mod n)

加密者将S、M、e、n公开之后，任何人都可以获得这段信息的原文：S^e ≡ ( M^d )^e ≡ M^d\*e ≡ M^e\*d ≡ M^1 ≡ M(mod n)

如同RSA的公共密钥加密算法经常被称之为非对称算法，因为加密密钥（在我们的例子中为e）和解密密钥（在我们的例子中是d）并不对称。取余运算的过程也不像我们平常接触的运算（诸如对数运算）那样简单。RSA加密算法的神奇之处在于你可以非常快速的进行数据的加密运算，即，但是如果没有解密密码d，你将很难破解出密码，即运算将不可能实现。正如我们所看到的，通过对n进行因式分解而得到p和q，再推断出解密密钥d的过程难于上青天。

签名验证
--------

在使用RSA加密算法的时候，最重要的一条就是要确保任何涉及到的数字都要足够复杂才能保证不被现有的计算方法所破解。这些数字要多复杂呢？Amazon.com的服务器是利用“VeriSign Class 3 Secure Server CA”来对证书进行签名的。从证书中，我们可以看到这个VeriSign（电子签名校验器，也称威瑞信公司）的系数n有2048位二进制数构成，换算成十进制足足有617位数字：

1890572922 9464742433 9498401781 6528521078 8629616064 3051642608
4317020197 7241822595 6075980039 8371048211 4887504542 4200635317
0422636532 2091550579 0341204005 1169453804 7325464426 0479594122
4167270607 6731441028 3698615569 9947933786 3789783838 5829991518
1037601365 0218058341 7944190228 0926880299 3425241541 4300090021
1055372661 2125414429 9349272172 5333752665 6605550620 5558450610
3253786958 8361121949 2417723618 5199653627 5260212221 0847786057
9342235500 9443918198 9038906234 1550747726 8041766919 1500918876
1961879460 3091993360 6376719337 6644159792 1249204891 7079005527
7689341573 9395596650 5484628101 0469658502 1566385762 0175231997
6268718746 7514321

（如果你想要对这一大串数字进行分解因式获得p和q，那就祝你好运！顺便一提，如果你真的计算出了p和q，那你就破解了Amazon.com数字签名证书了！）

这个VeriSign的加密密钥e是。当然，他们将解密密钥d保管得十分严密，通常是在拥有视网膜扫描和荷枪实弹的警卫守护的机房当中。在签名之前，VeriSign会根据相关约定的技术文档，对Amazon.com证书上所提供的信息进行校验。一旦证书信息符合相关要求，VeriSign会利用SHA-1哈希算法获取证书的哈希值（hash），并对其进行声明。在Wireshark中，完整的证书信息会显示在“signedCertificate”（已签名证书）中：

![](/post-images/2013-09/63918611gw1e8uag4tlrhj20b406kjs7.jpg)

这里应该是软件的用词不当，因为这一段实际上是指那些即将被签名的信息，而不是指那些已经包含了签名的信息。

![](/post-images/2013-09/63918611gw1e8uag5yeqij20b404cjs3.jpg)

实际上经过签名的信息S，在Wireshark中被称之为“encrypted”（密文）。我们将S的e次幂除以n取余数（即公式：）就能计算出被加密的原文，其十六进制如下：

0001FFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF
FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF
FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF
FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF
FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF
FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF
FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFF00302130
0906052B0E03021A05000414C19F8786 871775C60EFE0542 E4C2167C830539DB

根据PKCS\#1 v1.5标准（译者注：The Public-Key Cryptography Standards (PKCS)是由美国RSA数据安全公司及其合作伙伴制定的一组公钥密码学标准）规定：“第一个字节是00，这样就可以保证加密块在被转换为整数的时候比其加密参数要小。”第二个字节为01，表示了这是一个私有密钥操作（数字签名就是私有密钥操作的一种）。后面紧接着的一连串的FF字节是为了填充数据，使得这一串数字变得足够大（加大黑客恶意破解的难度）。填充数字以一个00字节结束。紧接着的30 21 30 09 06 05 2B 0E 03 02 1A 05 00 04 14这些字节是PKCS\#1 v2.1标准中用于说明这段哈希值是通过SHA-1算法计算而出的。最后的20字节是SHA-1算法所计算出来的哈希值，即对未加密信息的摘要描述。（译者注：原文中这里使用了带引号的signedCertificate，根据作者前文描述，这应该是Wireshark软件的bug，实际上应指的是未被加密的信息。）

因为这段信息的格式正确，且最后的哈希值与我们独立计算出来的校验一致，所以我们可以断定，这一定是知道“VeriSign Class 3 Secure Server CA”的解密密钥d的人对它进行了签名。而世界上只有VeriSign公司才知道这串密钥。

当然了，我们也可以重复验证这个“VeriSign Class 3 Secure Server CA”的证书的确是通过VeriSign公司的“第三类公私证书认证（Class 3 Public Primary Certification Authority）”进行签名的。

但是，即便是这样，我们为什么要信任VeriSign公司？整个的信任链条就此断掉了。

![](/post-images/2013-09/63918611gw1e8uag6ww8rj20b40353ym.jpg)

由图可以看到，“VeriSign Class 3 Secure Server CA”对Amazon.com进行了签名，而“VeriSign Class 3 Public Primary Certification Authority”对“VeriSign Class 3 Secure Server CA”进行了签名，但是最顶部的“VeriSign Class 3 Public Primary Certification Authority”则对自己进行了签名。这是因为，这个证书自从NSS（网络安全服务）库中的certdata.txt升级到1.4版之后就作为“受信任的根证书颁发机构”（译者注：参照微软官方翻译）被编译到了Mozilla产品中（火狐浏览器）。这段信息是由网景公司的Robert Relyea于2000年9月6日提交的，并随附以下注释：

“由仅存的NSS编译了框架。包含一个在线的certdata.txt文档，其中包含了我们受信的跟证书颁发机构（一旦我们获得了其他受信机构的许可会陆续将他们添加进去）”。

这个举动有着相当长远的影响，因为这些证书的有效日期是从1996年1月28日到2028年1月1日。

肯·汤普逊（Ken Thompson）在他的《对深信不疑的信任》（译者注：Reflections on Trusting Trust是肯汤普逊1983年获得图灵奖时的演说）的演说中解释的很好：你最终还是要绝对信任某一人，在这个问题上没有第二条路可走。在本文的例子中，我们就毫无保留的信任Robert Relyea做了一个正确的决定。我们同样希望Mozilla在自己软件中加入“受信任根证书颁发机构”这种行为也是合理的吧。

这里需要注意的是：这一系列的证书和签名只是用来形成一个信任链。在公共互联网上，VeriSign的根证书被火狐浏览器完全信任的时间远早于你接触互联网。在一个公司中，你可以创建自己的受信任的根证书颁发机构并把它安装到任何人的计算机中。

相对的，你也可以购买VeriSign公司的业务，降低整个证书信任链的信任风险。通过第三方的认证机构（在这个例子里是VeriSign公司）我们能利用证书建立起信任关系。如果你有类似于“悄悄话”的安全途径来传递一个秘密的key，那你也可以使用一个预共享密钥（PSK）来建立起信任关系。诸如TLS-PSK、或者带有安全远程密码（SRP）的TLS扩展包都能让我们使用预共享密钥。不行的是，这些扩展包在应用和支持方面远远比不上TLS，所以他们有的时候并不实用。另外，这些替代选项需要额外德尔安全途径进行保密信息的传输，这一部分的开销远比我们现在正在应用的TLS庞大。换句话说，这也就是我们为什么不应用那些其他途径构建信任关系的原因。

言归正传，我们所需要的最后确认的信息就是在证书上的主机名跟我们预想的是一样的。Nelson Bolyard在SSL\_AuthCertificate 函数中的注释为我们解释其中的原因：

“SSL连接的客户端确认证书正确，并检查证书中所对应的主机名是否正确，因为这是我们应对中间人攻击的唯一方式！”（译者注：中间人攻击是一种“间接”的入侵攻击，这种攻击模式是通过各种技术手段将受入侵者控制的一台计算机虚拟放置在网络连接中的两台通信计算机之间，这台计算机就称为“中间人”。）

    /* cert is OK. This is the client side of an SSL connection.
     * Now check the name field in the cert against the desired hostname.
     * NB: This is our only defense against Man-In-The-Middle (MITM) attacks! */

这样的检查是为了防止中间人攻击：因为我们对整个信任链条上的人都采取了完全信任的态度，认为他们并不会进行黑客行为，就像我们的证书中所声称它是来自Amazon.com，但是假如他的真实来源并非Amazon.com，那我们可能就有被攻击的危险。如果攻击者使用域名污染（DNS cache poisoning）等技术对你的DNS服务器进行篡改，那么你也许会把黑客的网站误认为是一个安全的受信网站（诸如Amazon.com），因为地址栏显示的信息一切正常。这最后一步对证书颁发机构的检查就是为了防止这样的事情发生。

随机密码串（Pre-Master Secret）
-----------------------------

现在我们已经了解了Amazon.com的各项要求，并且知道了公共解密密钥e和参数n。在通信过程中的任何一方也都知道了这些信息（佐证就是我们通过Wireshark获得了这些信息）。现在我们所需要做的事情就是生成一串窃密者/攻击者都不能知道的随机密码。这并不像听上去的那么简单。早在1996年，研究人员就发现了网景浏览器1.1的伪随机数发生器仅仅利用了三个参数：当天的时间，进程ID和父进程ID。正如研究人员所指出的问题：这些用于生成随机数的参数并不具有随机性，而且他们相对来说比较容易被破解。

因为一切都是来源于这三个随机数参数，所以在1996，利用当时的机器仅需要25秒钟的时间就可以破解一个SSL通信。找到一种生成真正随机数的方法是非常困难的，如果你不相信这一点，那就去问问Debian
OpenSSL的维护工程师吧。如果随机数的生成方式遭到破解，那么建立在这之上的一系列安全措施都是毫无意义的。

在Windows操作系统中，用于加密目的随机数都是利用一个叫做CryptGenRandom的函数生成的。这个函数的哈希表位对超过125个来源的数据进行抽样！火狐浏览器利用CryptGenRandom函数和它自身的函数来构成它自己的伪随机数发生器。（译者注：之所以称之为伪随机数是因为真正意义上的随机数算法并不存在，这些函数还是利用大量的时变、量变参数来通过复杂的运算生成相对意义上的随机数，但是这些数之间还是存在统计学规律的，只是想要找到生成随机数的过程并不那么容易）。

我们并不会直接利用生成的这48字节的随机密码串，但是由于很多重要的信息都是由他计算而来的，所以对随机密码串的保密就显得格外重要。正如我之前所预料到的，火狐浏览器对随机密码串的保密十分严格，所以我不得不编译了一个用于debug的版本。为了观察随机密码串，我还特地设置了SSLDEBUGFILE和SSLTRACE两个环境变量。

其中，SSLDEBUGFILE显示的就是随机密码串的值：

    4456: SSL[131491792]: Pre-Master Secret [Len: 48]
    03 01 bb 7b 08 98 a7 49 de e8 e9 b8 91 52 ec 81 ...{...I.....R..
    4c c2 39 7b f6 ba 1c 0a b1 95 50 29 be 02 ad e6 L.9{......P)....
    ad 6e 11 3f 20 c4 66 f0 64 22 57 7e e1 06 7a 3b .n.? .f.d"W~..z;

需要注意的是，这串数字从各种意义上来说都不是真正的随机数，就拿它的前两位来说：这就是根据TLS协议约定的TLS版本号（0301）。

****密码交换（Trading Secret）
----------------------------

我们现在需要做的就是计算出Amazon.com所要求的密码。因为Amazon.com希望使用“TLS\_RSA\_WITH\_RC4\_128\_MD5”加密组，所以我们使用RSA加密算法进行这一过程。你可以将这48字节的随机密码串作为初始参数，但是根据公共密钥密码标准（PKCS）\#1 v1.5中的注释，我们需要用随机数据将随机密码串填充到实际要求的参数大小（1024位二进制/128字节）。这样的话攻击者想要破解我们的随机密码串就难上加难了。这也是我们保障自己安全的最后一道防线，以防我们在前面的步骤中犯了诸如重复使用密码这样的低级错误。如果我们重复使用了随机密码串，由于使用了随机数填充，窃密者在网络中拦截的也会是两个不同的值。

同样的，我们很难直接观察到火狐浏览器中的这一过程，所以我不得不在填充随机数的函数中增加了debug的语句，使我们能够观察这一过程：

    wrapperHandle = fopen("plaintextpadding.txt", "a");
    fprintf(wrapperHandle, "PLAINTEXT = ");
    for(i = 0; i < modulusLen; i++)
    {
        fprintf(wrapperHandle, "%02X ", block[i]);
    }
    fprintf(wrapperHandle, "\r\n");
    fclose(wrapperHandle);

在这个例子中，完整的填充后的随机密码串为：

00 02 12 A3 EA B1 65 D6 81 6C 13 14 13 62 10 53 23 B3 96 85 FF 24 FA CC
46 11 21 24 A4 81 EA 30 63 95 D4 DC BF 9C CC D0 2E DD 5A A6 41 6A 4E 82
65 7D 70 7D 50 09 17 CD 10 55 97 B9 C1 A1 84 F2 A9 AB EA 7D F4 CC 54 E4
64 6E 3A E5 91 A0 06 00 03 01 BB 7B 08 98 A7 49 DE E8 E9 B8 91 52 EC 81
4C C2 39 7B F6 BA 1C0A B1 95 50 29 BE 02 AD E6 AD 6E 11 3F20 C4 66 F0 64
22 57 7E E1 06 7A 3B

火狐浏览器使用这个值计算出 ，我们可以看到它显示在“客户端交换密钥”（Client
Key Exchange）的记录中：

![](/post-images/2013-09/63918611gw1e8uag7w4chj20b403sgm7.jpg)

在这个过程的最后，火狐浏览器会发送一个不加密的信息：一条“Change Cipher
Spec”记录：

![](/post-images/2013-09/63918611gw1e8uag8ow99j20b4023aa6.jpg)

通过这种方式：火狐浏览器要求Amazon.com在后面的通信过程中使用约定的加密方式传输信息。

获得主密钥（Master Secret）
-------------------------

如果我们正确完成了之前的过程，并且各方都获得了48字节（256二进制位）的随机密码串。从Amazon.com的角度来看，这里还有一些信任问题：随机密码串是由客户端生成的，并没有将任何服务器信息或者之前约定的信息加入其中。这一点，我们会通过生成主密钥的方式加以完善。根据协议规范约定，这个的计算过程为：

    master_secret = PRF(pre_master_secret, "master secret", ClientHello.random + ServerHello.random)

pre\_master\_secret就是我们之前传送的随机密码串，”master secret”是一串ASCII码（例如：6d 61 73 74 65 72……），再连接上在客户端问候和服务器问候（来自Amazon的）的信息。

PRF是在规范中约定的伪随机函数，它将密钥、ASCII码标签、哈希值整合在一起。各有一半的参数分别使用MD5和SHA-1获取哈希值。这是一种十分明智的做法，即使是想要单单破解相对简单MD5和SHA-1也不是那么容易的事情。而且这个函数会将返回值传给自身直至迭代到我们需要的位数。

利用这个函数，我们生成了48字节的主密钥：

4C AF 20 30 8F4C AA C5 66 4A 02 90 F2 AC 10 00 39 DB 1D E0 1F CB E0 E0
9D D7 E6 BE 62 A4 6C 18 06 AD 79 21 DB 82 1D 53 84 DB 35 A7 1F C1 01 19

 

生成各种密钥
------------

现在，各方面已经有了主密钥，根据协议约定，我们需要利用PRF生成这个会话中所需要的各种密钥，称之为“密钥块”（key
block）：

    key_block = PRF(SecurityParameters.master_secret, "key expansion", SecurityParameters.server_random + SecurityParameters.client_random);

密钥块用于构成以下密钥：

    client_write_MAC_secret[SecurityParameters.hash_size]
    server_write_MAC_secret[SecurityParameters.hash_size]
    client_write_key[SecurityParameters.key_material_length]
    server_write_key[SecurityParameters.key_material_length]
    client_write_IV[SecurityParameters.IV_size]
    server_write_IV[SecurityParameters.IV_size]

因为我们使用了类似于高级加密标准（AES）的密码流代替了分组密码我们就不需要初始化向量（IVs）了。因此我们只需要双方的两个16字节（128二进制位）的消息认证码（Message Authentication Code，MAC），因为MD5的哈希值就是16字节的。此外，双方也需要16字节（128二进制位）的RC4码。所以我们总共需要从密码块获得2\*16 + 2\*16 = 64字节的数据。

运行PRF，我们能得到以下值：

    client_write_MAC_secret = 80 B8 F6 09 51 74 EA DB 29 28 EF 6F 9A B8 81 B0
    server_write_MAC_secret = 67 7C 96 7B 70 C5 BC 62 9D 1D 1F 4A A6 79 81 61
    client_write_key = 32 13 2C DD 1B 39 36 40 84 4A DE E5 6C 52 46 72
    server_write_key = 58 36 C4 0D 8C 7C 74 DA 6D B7 34 0A 91 B6 8F A7

准备加密！
---------

客户端最后一次送出的握手信息是“结束信息”。这条信息保证了没有人篡改握手信息，并且我们已经知晓所必须的密钥。客户端将整个握手过程的全部信息都放入一个名为“handshake\_messages”的缓冲区。我们能通过伪随机函数利用主密钥、“client finished”标签、MD5和SHA-1的哈希值生成12字节的“区别数据”（verify\_data）：

    verify_data = PRF(master_secret, "client finished", MD5(handshake_messages) + SHA-1(handshake_messages)) [12]

我们在这个结果前面加上0×14（用于表示结束信息）和00 00 0c（用于表示verify\_data有12字节）。就像以后所有的加密过程一样，我们要在加密之前确保原始数据没有被篡改。因为我们使用的是“TLS\_RSA\_WITH\_RC4\_128\_MD5”密码组，这就意味着我们需要使用MD5哈希函数。

有些人一听到MD5函数就会嗤之以鼻，因为其自身的确存在一些缺陷。我自己当然也不会推荐这种算法。但是TLS的聪明之处就在于他并不直接使用MD5函数，只是利用哈希值的版本来校验数据。只就意味着我们并未直接应用到MD5(m)：

HMAC\_MD5(Key, m) = MD5((Key ⊕ opad) ++ MD5((Key ⊕ ipad) ++ m)

（其中，⊕表示的是异或运算）

在实际中：

HMAC\_MD5(client\_write\_MAC\_secret, seq\_num + TLSCompressed.type +
TLSCompressed.version + TLSCompressed.length + TLSCompressed.fragment));

正如你所见，我们在函数中使用了一个根据明文（在这里明文叫做“TLSCompressed”）编号的序号（seq\_num）。这个序号的作用就是为了阻止攻击者在数据流中间插入之前被其截获的信息。如果发生了这样的攻击，序号就能清楚的警告我们数据中的异常。同样的，这个序号也能帮助我们发现攻击者从数据流中剔除的数据。

剩下的工作只剩下啊加密这些数据了！

RC4加密
-------

我们之前协商过的密码组是“TLS\_RSA\_WITH\_RC4\_128\_MD5”。这就意味着我们需要使用RC4（Ron\`s code 4）加密规则进行数据流的加密。罗纳德李威斯特（Ron Rivest）开发了这种基于一个256字节的Key产生随机加密效果的算法。这个算法简单到你几分钟就能记住。

首先，RC4生成一个256字节的数组S，并用0-255填充。接下来的工作就是将需要将KEY混合插入进数组中并反复迭代。你可以编写一个状态机，利用它来阐释随机字节。为了产生随机字节，我们需要将S数组打乱，如图所示：

![](/post-images/2013-09/63918611gw1e8uag9iu9nj208w0480st.jpg)

为了加密一个字节，我们将其与随机字节进行异或运算。记住：一位二进制数与1作异或运算会翻转（译者注：即1\^1=0；1\^0=1）。因为我们利用的是随机数，所以从统计学的角度来讲，有一半的数被翻转。这种随机翻转的现象就是我们加密数据的有效方法。正如你所见，这并不复杂，而且计算速度十分快，我认为这也是Amazon.com选择它的原因之一。

记得我们有“client\_write\_key”和“server\_write\_key”吗？这就意味我们需要两个RC4实例：一个用来加密浏览器向服务器传送的数据，一个用来解密服务器向浏览器传送的数据。

“client\_write\_key”最初的几个字节是7E 20 7A 4D FE FB 78 A7 33 …如果我们对这些字节和未加密的数据头以及版本信息（“14 00 00 0C98 F0 AE CB C4 …”）进行异或运算，我们就能得到在Wireshark中看到的加密信息了：

![](/post-images/2013-09/63918611gw1e8uagaluufj20b402l3yr.jpg)

服务器端做的几乎是相同的事情。它们发送了一个密钥协议的说明和一个包含全部握手过程的结束信息，其中有结束信息的解密版本。因此，这种机制就保证了客户端和服务器能成功的解密信息。

欢迎来到应用层！
---------------

现在，从我们点击了按钮之后已经过去了220毫秒，我们终于为应用层做好了准备！现在，我们发送的普通的HTTP数据流会通过TLS层的加密实例进行加密，在服务器的解密实例进行解密。而且TLS会对数据进行哈希校验，以保证数据内容的准确性。

在这个时候，整个的握手过程就结束了。我们的TLS记录内容现在有了23条（0×17）。加密数据以“17 03 01”开头，表示了记录类型和TLS版本，后面紧跟着加密数据的大小和哈希校验值。

加密的数据的明文如下：

    GET /gp/cart/view.html/ref=pd_luc_mri HTTP/1.1
    Host: www.amazon.com
    User-Agent: Mozilla/5.0 (Windows; U; Windows NT 6.0; en-US; rv:1.9.0.10) Gecko/2009060911 Minefield/3.0.10 (.NET CLR 3.5.30729)
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-us,en;q=0.5
    Accept-Encoding: gzip,deflate
    Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
    Keep-Alive: 300
    Connection: keep-alive
    ...

在Wireshark中显示如下：

![](/post-images/2013-09/63918611gw1e8uagbgauoj20b4017dfw.jpg)

唯一有趣的地方是序号是按照记录来增长，这条记录是1，下一条就是2。

服务器端利用“server\_write\_key”做着同样的事情。我们能看到服务器的相应结果，包括程序开头的指示位：

![](/post-images/2013-09/63918611gw1e8uagcffmyj20b4042dgh.jpg)

解密后的信息如下：

    HTTP/1.1 200 OK
    Date: Wed, 10 Jun 2009 01:09:30 GMT
    Server: Server
    ...
    Cneonction: close
    Transfer-Encoding: chunked

这就是一个来自Amazon负载平衡服务器的普通HTTP回应：包含了非描述性的服务器信息“Server: Server”和一个拼错了的“Cneonction: close”。

TLS层在应用层的下面，所以软件和服务器能够像正常的HTTP传输那样进行工作，唯一的区别就是传输的数据会被TLS层进行加密。

OpenSSL是一个应用很广的TLS开源库。

整个连接会一直保持，除非有一方提出了“关闭警告（closure alert）”并且关闭了连接。如果我们在连接断开后的短时间内再次提出连接请求，我们可以使用之前使用过的key来进行连接，从而避免一次新的握手过程。（这个要取决于服务器端key的有效时间。）

需要注意的是：应用程序可以发送任何数据，但是HTTPS的特殊之处在于WEB应用的广泛普及。要知道还有非常多的基于TCP/IP并且使用TLS进行数据加密的协议（如FTPS，sSMTP）。使用TLS要比你自己发明一种是数据加密方案便捷的多。况且，你所使用的安全协议一定要足够安全。

…完工！
------

TLS RFC的文档包含了更多的信息，有需要的朋友们可以自己查阅，我们在这里只是简单的介绍了其中的过程和原理，观察了这220毫秒内发生在火狐浏览器和Amazon服务器之间发生的故事：由Amazon.com基于速度和安全的综合考虑选择的“TLS\_RSA\_WITH\_RC4\_128\_MD5”密码组在HTTPS连接建立过程中的全部流程。

正如我们所看到的那样，如果有人能对Amazon服务器的参数n进行因式分解得到p和q的话，那他就能破解全部的基于亚马逊证书的安全通信。所以Amazon为这个参数设置了有效期以防止这种事情的发生：

![](/post-images/2013-09/63918611gw1e8uagf9f96j205n01nt8j.jpg)

在我们提供的密码族中，有一组密码组“TLS\_DHE\_RSA\_WITH\_AES\_256\_CBC\_SHA”使用了Diffie-Hellman密钥交换，并因此能提供良好的前向安全特性。这就意味着如果有人破解了交换密钥的数学运算方式，他们也不能利用这个来破解其他的会话。但是他的一个劣势在于其运算需求更大的数字和更高的运算能力。AES算法在很多密码组中都出现了，它与RC4的不同之处在于它每次使用的是16字节的“块”而RC4使用的是单字节。因为其key最高能到256位二进制位，所以一般认为它比RC4的安全性更高。

在短短的220毫秒的时间里，两个节点通过互联网连接起来，并且利用一系列手段建立起了互信机制，构建了加密算法，进行加密数据的传输。

正是因为如此，我们故事的主人公才能在Amazon上买到他想要的牛奶！

（译者注：作者所有相关的程序已经提交到Github上，地址：[https://github.com/moserware/TLS-1.0-Analyzer/tree/master](https://github.com/moserware/TLS-1.0-Analyzer/tree/master)）

**本文由 [伯乐在线](http://blog.jobbole.com) - [水果泡腾片](http://blog.jobbole.com/author/ripenc/) 翻译自 [JEFF MOSER](http://www.moserware.com/2009/06/first-few-milliseconds-of-https.html)。**