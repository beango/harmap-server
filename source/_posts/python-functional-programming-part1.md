---
layout: post
category: default
date: 2013-03-04
title: "Python中函数式编程，第一部分"
description: "Python中函数式编程，第一部分"
tags: [python]
redirecturl: http://www.oschina.net/translate/python-functional-programming-part1
---


英文原文：[Charming Python: Functional programming in Python, Part 1](http://www.ibm.com/developerworks/linux/library/l-prog/index.html)，翻译：[开源中国](http://www.oschina.net/translate/python-functional-programming-part1)

**摘要**：虽然人们总把Python当作过程化的，面向对象的语言，但是他实际上包含了函数化编程中，你需要的任何东西。这篇文章主要讨论函数化编程的一般概念，并说明用Python来函数化编程的技术。

我们最好从艰难的问题开始出发：“到底什么是函数化编程呢？”其中一个答案可能是这样的，函数化编程就是你在使用Lisp这样的语言时所做的（还有Scheme，Haskell，ML，OCAML，Mercury，Erlang和其他一些语言）。这是一个保险的回答，但是它解释得并不清晰。不幸的是对于什么是函数化编程，很难能有一个协调一致的定义，即使是从函数化变成本身出发，也很难说明。这点倒很像盲人摸象。不过，把它拿来和命令式编程（imperative programming）做比较也不错（命令式编程就像你在用C，Pascal，C++，Java，Perl，Awk，TCL和很多其他类似语言时所做的，至少大部分一样 ）。

让我们回想一下功能模块的绑定类。使用该类的特性，我们可以确认在一个给定的范围块内，一个特定的名字仅仅代表了一个唯一的事物。

我个人粗略总结了一下，认为函数式编程至少应该具有下列几点中的多个特点。在谓之为函数式的语言中，要做到这些就比较容易，但要做到其它一些事情不是很难就是完全不可能：

-   函数具有首要地位(对象)。也就是说，能对“数据”做什么事，就要能对函数本身做到那些事（比如将函数作为参数传递给另外一个函数）。
-   将递归作为主要的控制结构。在有些函数式语言中，都不存在其它的“循环”结构。
-   列表处理作为一个重点（例如，Lisp语言的名字）。列表往往是通过对子列表进行递归取代了循环。
-   “纯”函数式语言会完全避免副作用。这么做就完全弃绝了命令式语言中几乎无处不在的这种做法：将第一个值赋给一个变量之后为了跟踪程序的运行状态，接着又将另外一个值赋给同一个变量。
-   函数式编程不是不鼓励就是完全禁止使用*语句*，而是通过对表达式(换句话说，就是函数加上参数）求值（evaluation of expressions）完成任务.
    在最纯粹的情形下，一个程序就是一个表达式（再加上辅助性的定义）
-   函数式编程中最关心的是要对*什么*进行计算，而不是要*怎么来*进行计算。
-   在很多函数式[编程语言](http://blog.jobbole.com/tag/%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80/ "如何选择语言和编程语言排名相关文章")中都会用到“高阶”（higher order）函数
    (换句话说，高阶函数就是对对函数进行运算的函数进行运算的函数）。

函数式编程的倡导者们认为，所有这些特性都有助于更快地编写出更多更简洁并且更不容易出Bug的代码。而且，计算机科学、逻辑学和数学这三个领域中的高级理论家发现，函数式编程语言和程序的形式化特性在证明起来比命令式编程语言和程序要简单很多。

Python内在的函数式功能
----------------------

自Python 1.0起，Python就已具有了以上所列中的绝大多数特点。但是就象Python所具有的大多数特性一样，这些特点出现在了一种混合了各种特性的语言中。 和Python的OOP（面向对象编程）特性非常象，你想用多少就用多少，剩下的都可以不管（直到你随后需要用到它们为止）。在Python 2.0中，加入了*列表解析*（*list comprehensions*）这个非常好用的”语法糖“。尽管列表解析没有添加什么新功能，但它让很多旧功能看起来好了*不少*。

Python中函数式编程的基本要素包括functionsmap()、reduce()、filter()和lambda算子（operator）。
在Python 1.x中，apply()函数也可以非常方便地拿来将一个函数的列表返回值直接用于另外一个函数。Python 2.0为此提供了一个改进后的语法。可能有点让人惊奇，使用如此之少的函数（以及基本的算子）几乎就足以写出任何Python程序了；更加特别的是，几乎用不着什么执行流程控制语句。

所有(if,elif,else,assert,try,except,finally,for,break,continue,while,def)这些都都能通过仅仅使用函数式编程中的函数和算子就能以函数式编程的风格处理好。尽管真正地在程序中完全排除使用所有流程控制命令可能只在想参加”Python混乱编程“大赛（可将Python代码写得跟Lisp代码非常象）时才有意义，但这对理解函数式编程如何通过函数和递归表达流程控制很有价值。

剔除流程控制语句
----------------

剔除练习首先要考虑的第一件事是，实际上，Python会对布尔表达式求值进行“短路”处理。这就为我们提供了一个if/elif/else分支语句的表达式版（假设每个分支只调用一个函数，不是这种情况时也很容易组织成重新安排成这种情况）。
这里给出怎么做：

对Python中的条件调用进行短路处理

    # Normal statement-based flow control
    if <cond1>:   func1()
    elif <cond2>: func2()
    else:         func3()
     
    # Equivalent "short circuit" expression
    (<cond1> and func1()) or (<cond2> and func2()) or (func3())
     
    # Example "short circuit" expression
    >>> x = 3
    >>> def pr(s): return s
    >>> (x==1 and pr('one')) or (x==2 and pr('two')) or (pr('other'))
    'other'
    >>> x = 2
    >>> (x==1 and pr('one')) or (x==2 and pr('two')) or (pr('other'))
    'two'

我们的表达式版本的条件调用看上去可能不算什么，更象是个小把戏；然而，如果我们注意到lambda算子必须返回一个表达式，这就更值得关注了。既然如我们所示，表达式能够通过短路包含一个条件判断，那么，lambda表达式就是个完全通用的表达条件判断返回值的手段了。我们来一个例子：

Python中短路的Lambda

    >>> pr = lambda s:s
    >>> namenum = lambda x: (x==1 and pr("one")) \
    ....                  or (x==2 and pr("two")) \
    ....                  or (pr("other"))
    >>> namenum(1)
    'one'
    >>> namenum(2)
    'two'
    >>> namenum(3)
    'other'

**将函数作为具有首要地位的对象**

前面的例子已经表明了Python中函数具有首要地位，但有点委婉。当我们用lambda操作创建一个*函数对象*时，我们所得到的东西是完全通用的。就其本质而言，我们可以将我们的对象同名字”pr”和”namenum”绑定到一起,以完全相同的方式，我们也也完全可以将数字23或者字符串”spam”同这些名字绑定到一起。但是，就象我们可以无需将其绑定到任何名字之上就能直接使用数字23（也就是说，它可以用作函数的参数）一样，我们也可以直接使用我们使用lambda创建的函数对象，而无需将其绑定到任何名字之上。在Python中，函数就是另外一种我们能够就像某种处理的值。

我们对具有首要地位的对象做的比较多的事情就是，将它们作为参数传递给函数式编程固有的函数map()、reduce()和filter()。这三个函数接受的第一个参数都是一个函数对象。

-   map()针对指定给它的一个或多个列表中每一项对应的内容，执行一次作为参数传递给它的那个函数，最后返回一个结果列表。
-   reduce()针对每个后继项以及最后结果的累积结果，执行一次作为参数传递给它的那个函数；例如，reduce(lambdan,m:n\*m,range(1,10))是求”10的阶乘”的意思（换言之，将每一项和前面所得的乘积进行相乘）
-   filter()使用那个作为参数传递给它的函数，对一个列表中的所有项进行”求值“，返回一个由所有能够通过那个函数测试的项组成的经过遴选后的列表。

我们经常也会把函数对象传递给我们自己定义的函数，不过一般情况下这些自定义的函数就是前文提及的内建函数的某种形式的组合。

通过组合使用这三种函数式编程内建的函数，能够实现范围惊人的“执行流程”操作(全都不用语句，仅仅使用表达式实现)。

 Python中的函数式循环
---------------------

替换循环语言和条件状态语言块同样简单。for可以直接翻译成map()函数。正如我们的条件执行，我们会需要简化语句块成简单的函数调用（我们正在接近通常能做的）：

替换循环

    for e in lst:  func(e)      # statement-based loop
    map(func,lst)           # map()-based loop

通过这种方法，对有序程序流将有一个相似的函数式方式。那就是，命令式编程几乎是由大量“做这，然后做那，之后做其它的”语句组成。map()让我们只要做这样：

Map-based 动作序列

    # let's create an execution utility function
    do_it = lambda f: f()
     
    # let f1, f2, f3 (etc) be functions that perform actions
     
    map(do_it, [f1,f2,f3])   # map()-based action sequence

通常，我们的整个主要的程序都可以使用一个map表达式加上一些函数列表的执行来完成这个程序。最高级别的函数的另一个方便的特性是你可以把它们放在一个列表里。

翻译while会稍稍复杂一些，但仍然可以直接地完成：

Python中的函数式”while”循环

    # statement-based while loop
    while <cond>:
        <pre-suite>
        if <break_condition>:
            break
        else:
            <suite>
     
    # FP-style recursive while loop
    def while_block():
        <pre-suite>
        if <break_condition>:
            return 1
        else:
            <suite>
        return 0
     
    while_FP = lambda: (<cond> and while_block()) or while_FP()
    while_FP()

在翻译while循环时，我们仍然需要使用while\_block()函数，这个函数本身里面可以包含语句而不是仅仅包含表达式。但我们可能还能够对这个函数再进行更进一步的剔除过程（就像前面模版中的对if/else进行短路处理一样）。还有，\<cond\>很难对普通的测试有什么用，比如while myvar==7，既然循环体（在设计上）不能对任何变量的值进行修改（当然，在while\_block()中可以修改全局变量）。有一种方法可以用来为while\_block()添加更有用的条件判断，让while\_block()返回一个有意义的值，然后将这个返回值同循环结束条件进行比较。现在应该来看一个剔除其中语句的具体例子了：

Python中’echo’循环

    # imperative version of "echo()"
    def echo_IMP():
        while 1:
            x = raw_input("IMP -- ")
            if x == 'quit':
                break
            else
                print x
    echo_IMP()
     
    # utility function for "identity with side-effect"
    def monadic_print(x):
        print x
        return x
     
    # FP version of "echo()"
    echo_FP = lambda: monadic_print(raw_input("FP -- "))=='quit' or echo_FP()
    echo_FP()

在上面的例子中我们所做的，就是想办法将一个涉及I/O、循环和条件判断的小程序，表达为一个递归方式的纯粹的表达式 （确切地说，表达为一个可以在需要的情况下传递到别的地方的函数对象）。我们*的确*仍然使用了实用函数monadic\_print()，但这个函数是完全通用的，而且可以用于以后我们可能会创建的每个函数式程序的表达式中（它的代价是一次性的）。请注意，任何包含monadic\_print(x)的表达式的*值*都是一样的，好像它只是包含了一个x而已。函数式编程中（特别是在Haskell中）的函数有一种叫做”monad”（一元）的概念，这种一元函数“实际什么都不做，只是在执行过程中产生一个副作用”。

 避免副作用
-----------

在做完这些没有非常明智的理由陈述，并把晦涩的嵌套表达式代替他们之后，一个很自然的问题是“为什么要这样做？！”　我描述的函数式编程在Python中都实现了。但是最重要的特性和一个有具体用处——就是避免副作用（或至少它们阻止如monads的特殊区域）。程序错误的大部分——并且这些问题驱使程序员去debug——出现是因为在程序的运行中变量获取了非期望的值。函数式编程简单地通过从不给变量赋值而绕过了这个问题。

现在让我们看一段非常普通的命令式代码。这段代码的目的是打印出乘积大于25的一对一对数字所组成的一个列表。组成每对数字的每一个数字都是取自另外的两个列表。这种事情和很多程序员在他们的编程中经常做的一些事情比较相似。命令式的解决方式有可能就象下面这样：

命令式的”打印大乘积”的Python代码

    # Nested loop procedural style for finding big products
    xs = (1,2,3,4)
    ys = (10,15,3,22)
    bigmuls = []
    # ...more stuff...
    for x in xs:
        for y in ys:
            # ...more stuff...
            if x*y > 25:
                bigmuls.append((x,y))
                # ...more stuff...
    # ...more stuff...
    print bigmuls

这个项目足够小了，好像没有地方会出什么差错。但有可能在这段代码中我们会嵌入一些同时完成其它任务的代码。用”more stuff”（其它代码）注释掉的部分，就是有可能存在导致出现bug的副作用的地方。在那三部分的任何一点上，变量sxs、ys、bigmuls、x、y都有可能在这段按照理想情况简化后的代码中取得一个出人意料的值。还有，这段代码执行完后，后继代码有可能需要也有可能不需要对所有这些变量中的值有所预期。显而易见，将这段代码封装到函数/实例中，小心处理变量的作用范围，就能够避免这种类型的错误。你也可以总是将使用完毕的变量del掉。但在实践中，这里指出的这种类型的错误很常见。

以一种函数式的途径一举消除这些副作用所产生的错误，这样就达到了我们的目的。一种可能的代码如下：

以函数式途径达到我们的目的

    bigmuls = lambda xs,ys: filter(lambda (x,y):x*y > 25, combine(xs,ys))
    combine = lambda xs,ys: map(None, xs*len(ys), dupelms(ys,len(xs)))
    dupelms = lambda lst,n: reduce(lambda s,t:s+t, map(lambda l,n=n: [l]*n, lst))
    print bigmuls((1,2,3,4),(10,15,3,22))

在例子中我们绑定我们的匿名（lambda）函数对象到变量名，但严格意义上讲这并不是必须的。我们可以用简单的嵌套定义来代替之。这不仅是为了代码的可读性，我们才这样做的；而且是因为combine()函数在任何地方都是一个非常好的功能函数（函数从两个输入的列表读入数据生成一个相应的pair列表）。函数dupelms()只是用来辅助函数combine()的。即使这个函数式的例子跟命令式的例子显得要累赘些，不过一旦你考虑到功能函数的重用，则新的bigmuls()中代码就会比命令式的那个要稍少些。

这个函数式例子的真正优点在于：在函数中绝对没有改变变量的值。这样就***不可能***在之后的代码（或者从之前的代码）中产生不可预期的副作用。显然，在函数中没有副作用，并不能保证代码的正确性，但它仍然是一个优点。无论如何请注意，Python（不像很多其它的函数式语言）不会阻止名字bigmuls，combine和dupelms的再次绑定。如果combine()运行在之后的程序中意味着有所不同时，所有的预测都会失效。你可能会需要新建一个单例类来包含这个不变的绑定（也就是说，s.bigmuls之类的）；但是这一例并没有空间来做这些。

一个明显值得注意的是，我们特定的目标是定制Python 2的一些特性。而不是命令式的或函数式编程的例子，最好的（也是函数式的）方法是：

    print [(x,y) for x in (1,2,3,4) for y in (10,15,3,22) if x*y > 25]

 结束语
-------

我已经列出了把每一个Python控制流替换成一个相等的函数式代码的方法（在程序中减少副作用）。高效翻译一个特定的程序需要一些额外的思考，但我们已经看出内置的函数式功能是全面且完善的。在接下来的文章里，我们会看到更多函数式编程的高级技巧；并且希望我们接下来能够摸索到函数式编程风格的更多优点和缺点。

[Python中的函数式编程，第一部分](/archives/2013/03/04/python-functional-programming-part1.html "Python中函数式编程，第一部分")

[Python中的函数式编程，第二部分](/archives/2013/04/11/python-functional-programming-part2.html "Python中函数式编程，第二部分")

[Python中的函数式编程，第三部分](/archives/2013/04/25/python-functional-programming-part3.html "Python中的函数式编程，第三部分")

 
