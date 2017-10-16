---
layout: post
category: default
date: 2013-06-22
title: "使用面向对象技术创建高级Web应用程序"
description: "使用面向对象技术创建高级Web应用程序"
tags: [javascript]
redirecturl: http://www.oschina.net/translate/create-advanced-web-applications-with-object-oriented-techniques
---


*英文原文：[Create Advanced Web Applications With Object-Oriented Techniques](http://msdn.microsoft.com/en-us/magazine/cc163419.aspx)*

最近，我面试了一位具有5年Web应用开发经验的软件开发人员。她有4年半的JavaScript编程经验，自认为自己具有非常优秀的JavaScript技能，可是，随后我很快发现，实际上她对JavaScript却知之甚少。然而，我并不是要为此而责怪她。JavaScript就是这么不可思议。有很多人（也包括我自己，这种情况直到最近才有所改观）都自以为是，觉得因为他们懂C/C++/C#或者具有编程经验，便以为他们非常擅长JavaScript这门语言。

从某个角度讲，这种自以为是也并非毫无道理。用JavaScript做一些简单的事情是非常容易的。其入门的门槛非常低；这个语言待人宽厚，并不苛求你必须懂它很多才能开始用它编写代码。甚至对于非程序员来说，也可以仅花个把小时就能够上手用它为他的网站编写几段或多或少都有些用的脚本。

实际上直到最近，无论懂的JavaScript有多么少，仅仅在MSDN® DHTML参考资料以及我在C++/C#方面编程经验的帮助下，我都能够凑合过下去。直到我在工作中真正开始编写AJAX应用时，我才发现我对JavaScript的了解有多么欠缺。这种新一代的Web应用复杂的交互特性要求使用一种完全不同的方式来编写JavaScript代码。这些都是非常严肃的JavaScript应用！我们以往那种漫不经心编写脚本的方法不灵了。

面向对象的编程(OOP)这种方法广泛用于多种JavaScript库，采用这种方法可使代码库更加易于管理和维护。JavaScript支持OOP，但它的支持方式同流行的Microsoft®.NET框架下的C++、C\#、Visual Basic®等语言完全不同，所以，大量使用这些语言的开发者起初可能会发现，JavaScript中的OOP比较怪异，同直觉不符。我写这篇文章就是要对JavaScript到底是如何支持面向对象编程的以及如何高效利用这种支进行面向对象的JavaScript开发进行深入讨论。接下来让我们开始谈谈对象（除了对象还能有别的吗？）吧。

JavaScript对象是字典
--------------------

在C++或C#中，当谈及对象时，我们指的是类或者结构的实例。对象根据实例化出它的模版（也即，类）的不同而具有不同的属性和方法。JavaScript对象不是这样的。在JavaScript中，对象仅仅是name/value对的集合，我们可以把JavaScript对象看作字典，字典中的键为字符串。我们可以用我们熟悉的"."(点)操作符或者一般用于字典的"[]"操作符，来获取或者设置对象的属性。下面的代码片段

    var userObject = new Object();
    userObject.lastLoginTime = new Date();
    alert(userObject.lastLoginTime);


同这段代码所做的完全是同样的事情：

    var userObject = {}; // equivalent to new Object()
    userObject["lastLoginTime"] = new Date();
    alert(userObject["lastLoginTime"]);

我们还可以用这样的方式，直接在userObject的定义中定义lastLoginTime属性：

    var userObject = { "lastLoginTime": new Date() };
    alert(userObject.lastLoginTime);

请注意这同C\#3.0的对象初始化表达式是多么的相似。另外，熟悉Python的读者会发现，在第二段和第三段代码中，我们实例化userObject的方式就是Python中指定字典的方式。这里唯一的区别的就是，JavaScript中的对象/字典只接受字符串作为键，而Python中字典则无此限制。

这些例子也表明，同C++或者C\#对象相比，JavaScript对象是多么地更加具有可塑性。属性lastLoginTime不必事先声明，如果在使用这个属性的时候userObject还不具有以此为名的属性，就会在userObject中把这个属性添加进来。如果记住了JavaScript对象就是字典的话，你就不会对此大惊小怪了 —— 毕竟我们随时都可以把新键（及其对应的值）添加到字典中去。

JavaScript对象的属性就是这个样子的。那么，JavaScript对象的方法呢？和属性一样，JavaScript仍然和C++/C\#不同。为了理解对象的方法，就需要首先仔细看看JavaScript函数。

JavaScript中的函数具有首要地位
------------------------------

在许多编程语言中，函数和对象一般都认为是两种不同的东西。可在JavaScript中，它们之间的区别就没有那么明显了 —— JavaScript中的函数实际上就是对象，只不过这个对象具有同其相关联的一段可执行代码。请看下面这段再普通不过的代码：

    function func(x) {
        alert(x);
    }
    func("blah");

这是JavaScript中定义函数最常用的方式了。但是，你还可以先创建一个匿名函数对象再将该对象赋值给变量func，也即，象下面那样，定义出完全相同的函数

    var func = function(x) {
        alert(x);
    };
    func("blah2");

或者甚至通过使用Function构造器，向下面这样来定义它：

    var func = new Function("x", "alert(x);");
    func("blah3");

这表明，函数实际上就是一个支持函数调用操作的对象。最后这种使用Function构造器来定义函数的方式并不常用，但却为我们带来很多很有趣的可能，其原因可能你也已经发现了，在这种函数定义的方式中，函数体只是Function构造器的一个字符串型的参数。这就意味着，你可以在JavaScript运行的时候构造出任意的函数。

要进一步证明函数是对象，你可以就象为任何其它JavaScript对象一样，为函数设置或添加属性：

    function sayHi(x) {
        alert("Hi, " + x + "!");
    }

    sayHi.text = "Hello World!";
    sayHi["text2"] = "Hello World... again.";

    alert(sayHi["text"]); // displays "Hello World!"
    alert(sayHi.text2); // displays "Hello World... again."

作为对象，函数还可以赋值给变量、作为参数传递给其它函数、作为其它函数的返回值、保存为对象的属性或数组中的一员等等。**图1**所示为其中一例。

图1 **函数在JavaScript具有首要地位**

    // assign an anonymous function to a variable
    var greet = function(x) {
        alert("Hello, " + x);
    };

    greet("MSDN readers");

    // passing a function as an argument to another
    function square(x) {
        return x * x;
    }

    function operateOn(num, func) {
        return func(num);
    }

    // displays 256
    alert(operateOn(16, square));

    // functions as return values
    function makeIncrementer() {
        return function(x) { return x + 1; };
    }

    var inc = makeIncrementer();
    // displays 8
    alert(inc(7));

    // functions stored as array elements
    var arr = [];
    arr[0] = function(x) { return x * x; };
    arr[1] = arr[0](2);
    arr[2] = arr[0](arr[1]);
    arr[3] = arr[0](arr[2]);

    // displays 256
    alert(arr[3]);

    // functions as object properties
    var obj = { "toString" : function() { return "This is an object."; } };

    // calls obj.toString()
    alert(obj);

记住这一点后，为对象添加方法就简单了，只要选择一个函数名并把一个函数赋值为这个函数名即可。接下来我通过将三个匿名函数分别赋值给各自相应的方法名，为一个对象定义了三个方法：

    var myDog = {
        "name" : "Spot",
        "bark" : function() { alert("Woof!"); },
        "displayFullName" : function() {
            alert(this.name + " The Alpha Dog");
        },

        "chaseMrPostman" : function() { 
            // implementation beyond the scope of this article 
        }    
    };

    myDog.displayFullName(); 
    myDog.bark(); // Woof!

函数displayFullName中"this"关键字的用法对C++/C\#开发者来说并不陌生 —— 该方法是通过哪个对象调用的，它指的就是哪个对象（使用Visual Basic的开发者也应该熟悉这种用法 —— 只不过"this"在Visual Basic称作"Me"）。因此在上面的例子中，displayFullName中"this"的值指的就是myDog对象。但是，"this"的值不是静态的。如果通过别的对象对函数进行调用，"this"的值也会随之指向这个别的对象，如**图2**所示。

图2 **“this”随着对象的改变而改变**

    function displayQuote() {
        // the value of "this" will change; depends on 
        // which object it is called through
        alert(this.memorableQuote);    
    }

    var williamShakespeare = {
        "memorableQuote": "It is a wise father that knows his own child.", 
        "sayIt" : displayQuote
    };

    var markTwain = {
        "memorableQuote": "Golf is a good walk spoiled.", 
        "sayIt" : displayQuote
    };

    var oscarWilde = {
        "memorableQuote": "True friends stab you in the front." 
        // we can call the function displayQuote
        // as a method of oscarWilde without assigning it 
        // as oscarWilde’s method. 
        //"sayIt" : displayQuote
    };

    williamShakespeare.sayIt(); // true, true
    markTwain.sayIt(); // he didn’t know where to play golf

    // watch this, each function has a method call()
    // that allows the function to be called as a 
    // method of the object passed to call() as an
    // argument. 
    // this line below is equivalent to assigning
    // displayQuote to sayIt, and calling oscarWilde.sayIt().

    displayQuote.call(oscarWilde); // ouch!

**图2**最后一行的代码是将函数作为一个对象的方法进行调用的另外一种方式。别忘了，JavaScript中的函数是对象。每个函数对象都有一个叫做call的方法，这个方法会将函数作为该方法的第一个参数的方法进行调用。也就是说，无论将哪个对象作为第一个参数传递给call方法，它都会成为此次函数调用中"this"的值。后面我们就会看到，这个技术在调用基类构造器时会非常有用。

有一点要记住，那就是永远不要调用不属于任意对象却包含有"this"的函数。如果调用了的话，就会搅乱全局命名空间。这是因为在这种调用中，"this"将指向Global对象，此举将严重损害你的应用。例如，下面的脚本将会改变JavaScript的全局函数isNaN的行为。我们不推荐这么干。

    alert("NaN is NaN: " + isNaN(NaN));

    function x() {
        this.isNaN = function() { 
            return "not anymore!";
        };
    }

    // alert!!! trampling the Global object!!!
    x();

    alert("NaN is NaN: " + isNaN(NaN));

到此我们已经看过了创建对象并为其添加熟悉和方法的几种方式。但是，如果你仔细看了以上所举的所以代码片段就会发现，所有的熟悉和方法都是在对象的定义之中通过硬性编码定义的。要是你需要对对象的创建进行更加严格的控制，那该怎么办？例如，你可能会需要根据某些参数对对象属性中的值进行计算，或者你可能需要将对象的属性初始化为只有到代码运行时才会得到的值，你还有可能需要创建一个对象的多个实例，这些要求也是非常常见的。

在C\#中，我们使用类类实例化出对象实例。但是JavaScript不一样，它并没有类的概念。相反， 在下一小节你将看到，你可以利用这一点：将函数同"new"操作符一起使用就可以把函数当着构造器来用。

有构造函数但没有类
------------------

JavaScript中的OOP最奇怪的事，如前所述，就是JavaScript没有C\#和C++中所具有的类。在C\#中，通过如下这样的代码

    Dog spot = new Dog();

能够得到一个对象，这个对象就是Dog类的一个实例。但在JavaScript中根本就没有类。要想得到同类最近似的效果，可以象下面这样定义一个构造器函数：

    function DogConstructor(name) {
        this.name = name;
        this.respondTo = function(name) {
            if(this.name == name) {
                alert("Woof");        
            }
        };
    }

    var spot = new DogConstructor("Spot");
    spot.respondTo("Rover"); // nope
    spot.respondTo("Spot"); // yeah!

好吧，这里都发生了什么？先请不要管DogConstructor函数的定义，仔细看看这行代码：

    var spot = new DogConstructor("Spot");

"new"操作符所做的事情很简单。首先，它会创建出一个新的空对象。然后，紧跟其后的函数调用就会得到执行，并且会将那个新建的空对象设置为该函数中"this"的值。换句话说，这行带有"new"操作符的代码可以看作等价于下面这两行代码：

    // create an empty object
    var spot = {}; 
    // call the function as a method of the empty object
    DogConstructor.call(spot, "Spot");

在DogConstructor的函数体中可以看出，调用该函数就会对调用中关键字"this"所指的对象进行初始化。采用这种方式，你就可以为对象创建模版了！无论何时当你需要创建类似的对象时，你就可以用"new"来调用该构造器函数，然后你就能够得到一个完全初始化好的对象。这和类看上去非常相似，不是吗？实际上，JavaScript中构造器函数的名字往往就是你想模拟的类的名字，所以上面例子中的构造函数你就可以直接命名为Dog：

    // Think of this as class Dog
    function Dog(name) {
        // instance variable 
        this.name = name;

        // instance method? Hmmm...
        this.respondTo = function(name) {
            if(this.name == name) {
                alert("Woof");        
            }
        };
    }

    var spot = new Dog("Spot");

上面在Dog的定义中，我定义了一个叫做name的实例变量。将Dog作为构造器函数使用而创建的每个对象都有自己的一份叫做name的实例变量（如前所述，name就是该对象的字典入口）。这符合我们的期望；毕竟每个对象都需属于自己的一份实例变量，只有这样才能保存它自己的状态。但是如果你再看接下来的那行代码，就会发现Dog的每个实例都有自己的一份respondTo方法，这可是个浪费；respondTo的实例你只需要一个，只有将这一个实例在所有的Dog实例间共享即可！你可以把respondTo的定义从Dog中拿出来，这样就可以克服此问题了，就向下面这样：

    function respondTo() {
        // respondTo definition
    }

    function Dog(name) {
        this.name = name;
        // attached this function as a method of the object
        this.respondTo = respondTo;
    }

这样一来，Dog的所有实例（也即，用构造器函数Dog创建的所有实例）都可以共享respondTo方法的同一个实例了。但是，随着方法数量的增加，这种方式维护起来会越来越困难。最后你的代码库中会堆积大量的全局函数，而且，随着“类”的数量不断增加，特别是这些类的方法具有类似的方法名时，情况会变得更加糟糕。这里还有一个更好的办法，就是使用原型对象，这就是下一个小节要讨论的内容。

原型（Prototype）
---------------

原型对象是JavaScript面向对象编程中的一个核心概念。原型这个名称来自于这样一个概念：在JavaScript中，所有对象都是通过对已有的样本（也即，原型）对象进行拷贝而创建的。该原型对象的所有属性和方法都会成为通过使用该原型的构造函数生成的对象的属性和方法。你可以认为，这些对象从它们的原型中继承了相应的属性和方法。当你象这样来创建一个新的Dog对象时

    var buddy = new Dog("Buddy");

buddy所引用的对象将从它的原型中继承到相应的属性和方法，虽然仅从上面这一行代码可能会很难看出来其原型来自哪里。buddy对象的原型来自来自构造器函数（在此例中指的就是函数Dog）的一个属性。

在JavaScript中，每个函数都有一个叫做“prototype”的属性，该属性指向一个原型对象。发过来，该原型对象据有一个叫做"constructor"的属性，该属性又指回了这个函数本身。这是一种循环引用；**图3** 更好地揭示出了这种环形关系。

![](/post-images/2013-06/20151040_gQ9A.gif) 图3** 每个函数的原型都具有一个叫做Constructor的属性 **

好了，当一个函数（比如上例中的Dog）和"new"操作符一起使用，创建出一个对象时，该对象将从Dog.prototype中继承所有的属性。在图**3**中，你可以看出，Dog.prototype对象具有一个指会Dog函数的construtor属性，每个Dog对象（它们继承自Dog.prototype)将同样也具有一个指会Dog函数的constructor属性。**图4**中的代码证明了这一点。构造器函数、原型对象以及用它们创建出来的对象这三者之间的关系如**图5**所示。

图4 **对象同样也具有它们原型的属性**

    var spot = new Dog("Spot");

    // Dog.prototype is the prototype of spot
    alert(Dog.prototype.isPrototypeOf(spot));

    // spot inherits the constructor property
    // from Dog.prototype
    alert(spot.constructor == Dog.prototype.constructor);
    alert(spot.constructor == Dog);

    // But constructor property doesn’t belong
    // to spot. The line below displays "false"
    alert(spot.hasOwnProperty("constructor"));

    // The constructor property belongs to Dog.prototype
    // The line below displays "true"
    alert(Dog.prototype.hasOwnProperty("constructor"));

![](/post-images/2013-06/20151041_UD0W.gif) 图5** 继承自它们的原型的实例**

有些读者可能已经注意到了**图4**中对hasOwnProperty方法和isPrototypeOf方法的调用。这些方法又来自哪里呢？它们并不是来自Dog.prototype。实际上，JavaScript中还有其它一些类似于toString、toLocaleString和valueOf等等我们可以直接对Dog.prototype以及Dog的实例进行调用的方法，但它们统统都不是来自于Dog.prototype的。其实就象.NET框架具有System.Object一样，JavaScript中也有Object.prototype，它是所有类的最顶级的基类。（Object.prototype的原型为null。）

在这个例子中，请记住Dog.prototype也是一个对象。它也是通过对Object的构造函数进行调用后生成的，虽然这一点在代码中并不直接出现：

    Dog.prototype = new Object();

所以，就如同Dog的实例继承自Dog.prototype一样，Dog.prototype继承自Object.prototype。这就使得Dog的所有实例也都会继承Object.prototype的方法和实例。

每个JavaScript对象都会继承一个原型链，该链的最末端都是Object.prototype。请注意，到此为止你在这里所见到的继承都是活生生的对象间的继承。这同你通常所认识的类在定义时形成的继承的概念不同。因此，JavaScript中的继承要来得更加的动态化。继承的算法非常简单，就是这样的：当你要访问一个对象的属性/方法时，JavaScript会首先对该属性/方法是否定义于该对象之中。如果不是，接下来就要对该对象的原型进行检查。如果还没有发现相应的定义，然后就会对该对象的原型的原型进行检查，并以此类推，直到碰到Object.prototype。**图6**所示即为这个解析过程。

JavaScript这种动态解析属性访问和方法调用的方式将对JavaScript带来一些影响。对原型对象的修改会马上在继承它的对象中得以体现，即使这种修改是在对象创建后才进行的也无关紧要。如果你在对象中定义了一个叫做X的属性/方法，那么该对象原型中同名的属性/方法就会无法访问到。例如，你可以通过在Dog.prototype中定义一个toString方法来对Object.prototype中的toString方法进行重载。所有修改指挥在一个方向上产生作用，即慈宁宫原型到继承它的对象这个方向，相反则不然。

**图7**所示即为这种影响。**图7**还演示了如何解决前文碰到的避免不必要的方法实例问题。不用让每个对象都具有一个单独的方法对象的实例，你可以通过将方法放到其原型之中来让所有对象共享同一个方法。此例中，getBreed方法由rover和spot共享
——
至少直到在spot中重载了getBreed（译者注：原文为toString，应为笔误）方法之前。spot在重载之后就具有自己版本的getBreed方法，但是rover对象以及随后使用new和GreatDane创建的对象仍将继承的是定义于GreatDane.prototype对象的getBreed方法。

图7  **从原型中进行继承**

    function GreatDane() { }

    var rover = new GreatDane();
    var spot = new GreatDane();

    GreatDane.prototype.getBreed = function() {
        return "Great Dane";
    };

    // Works, even though at this point
    // rover and spot are already created.
    alert(rover.getBreed());

    // this hides getBreed() in GreatDane.prototype
    spot.getBreed = function() {
        return "Little Great Dane";
    };

    alert(spot.getBreed()); 

    // but of course, the change to getBreed 
    // doesn’t propagate back to GreatDane.prototype
    // and other objects inheriting from it,
    // it only happens in the spot object
    alert(rover.getBreed());

静态属性和方法
--------------

有些时候你会需要同类而不是实例捆绑到一起的属性或方法 ——
也即，静态属性和静态方法。在JavaScript中这很容易就能做到，因为函数就是对象，所以可以随心所欲为其设置属性和方法。既然构造器函数在JavaScript代表了类这个概念，所以你可以通过在构造器函数中设置属性和昂奋来为一个类添加静态方法和属性，就象这样：

    function DateTime() { }

    // set static method now()
    DateTime.now = function() {
        return new Date();
    };

    alert(DateTime.now());

在JavaScript调用静态方法的语法实际上和C\#完全相同。既然构造器函数就是类的名字，所以这也不应该有什么奇怪的。这样你就有了类、共有属性/方法以及静态属性/方法。你还需要什么呢？当然，还需要私有成员。但是，JavaScript并不直接支持私有成员（这方面它也不支持protected成员）。对象的所以属性和方法所有人都可以访问得到。这里有一种在类中定义出私有成员的方法，但要完成这个任务就需要首先对闭包有所了解。 

闭包
----

我学JavaScript完全是迫不得已。因为我意识到，不学习JavaScript，就无法为在工作中参加编写真正的AJAX应用做好准备。起初，我有种在程序员的级别中下降了不少等级的感觉。（我要学JavaScript了！我那些使用C++的朋友该会怎么说我啊？）但是一旦我克服了起初的抗拒心理之后，我很快发现，JavaScript实际上是一门功能强大、表达能力极强而且很小巧的语言。它甚至拥有一些其它更加流行的语言才刚刚开始支持的特性。

JavaScript中更加高级的一个特性便是它对闭包的支持，在C\# 2.0中是通过匿名方法对闭包提供支持的。闭包是一种运行时的现象，它产生于内部函数（在C\#中成为内部匿名方法）本绑定到了其外部函数的局部变量之上的时候。显然，除非内部函数可以通过某种方式在外部函数之外也可以让其可以访问得到，否则这也没有多大意义。举个例子就可以把这个现象说得更清楚了。


假如你需要基于一个简单评判标准对一个数字序列进行过滤，该标准就是大于100的数字可以留下，但要把其它的所以数字都过滤掉。你可以编写写一个如**图8**所示的函数。

图8 **基于谓词（Predicate）对元素进行过滤**

    function filter(pred, arr) {

        var len = arr.length;
        var filtered = []; // shorter version of new Array();

        // iterate through every element in the array...
        for(var i = 0; i &lt; len; i++) {
            var val = arr[i];
            // if the element satisfies the predicate let it through
            if(pred(val)) {
                filtered.push(val);
            }
        }
        return filtered;
    }

    var someRandomNumbers = [12, 32, 1, 3, 2, 2, 234, 236, 632,7, 8];
    var numbersGreaterThan100 = filter(
        function(x) { return (x &gt; 100) ? true : false; }, 
        someRandomNumbers);


    // displays 234, 236, 632
    alert(numbersGreaterThan100);

但是现在你想新建一个不同的过滤标准，比方说，这次只有大于300的数字才能留下。你可以这么做：

    var greaterThan300 = filter(
        function(x) { return (x &gt; 300) ? true : false; }, 
        someRandomNumbers);

可能还需要留下大于50、25、10、600等等的数字，然而，你是如此聪明，很快就会发现它们使用的都是“大于”这同一个谓词，所不同的只是其中的数字。所以，你可以把具体的数字拿掉，编写出这么一个函数：

    function makeGreaterThanPredicate(lowerBound) {
        return function(numberToCheck) {
            return (numberToCheck &gt; lowerBound) ? true : false;
        };
    }

有了这个函数你就可以象下面这样做了：

    var greaterThan10 = makeGreaterThanPredicate(10);
    var greaterThan100 = makeGreaterThanPredicate(100);
    alert(filter(greaterThan10, someRandomNumbers));
    alert(filter(greaterThan100, someRandomNumbers));

请注意makeGreaterThanPredicate函数所返回的内部匿名函数。该匿名内部函数使用了lowerBound，它是传递给makeGreaterThanPredicate的一个参数。根据通常的变量范围规则，当makeGreater­ThanPredicate函数退出后，lowerBound就离开了它的作用范围！但是在此种情况下，内部匿名函数仍然还携带着它，即使make­GreaterThanPredicate早就退出了也还是这样。这就是我们称之为闭包的东西 ——— 因为内部函数关闭着它的定义所在的环境（也即，外部函数的参数和局部变量）。

乍一看，闭包也许没什么大不了的。但是如果使用得当，使用它可以在将你的点子转变为代码时，为你打开很多非常有意思的新思路。在JavaScript中闭包最值得关注的用途之一就是用它来模拟出类的私有变量。

模拟私有属性
------------

好的，现在让我们来看看在闭包的帮助下怎样才能模拟出私有成员。函数中的私有变量通常在函数之外是访问不到的。在函数执行结束后，实际上局部变量就会永远消失。然而，如果内部函数捕获了局部变量的话，这样的局部变量就会继续存活下去。 这个实情就是在JavaScript中模拟出私有属性的关键所在。请看下面的Person类：

    function Person(name, age) {
        this.getName = function() { return name; };
        this.setName = function(newName) { name = newName; };
        this.getAge = function() { return age; };
        this.setAge = function(newAge) { age = newAge; };
    }

参数name和age对构造器函数Person来说就是局部变量。一旦Person函数返回之后，name和age就应该被认为永远消失了。然而，这两个参数被4个内部函数捕获，这些内部函数被赋值为Person实例的方法了，因此这样一来就使得name和age能够继续存活下去，但却被很严格地限制为只有通过这4个方法才能访问到它们。所以，你可以这样做：

    var ray = new Person("Ray", 31);
    alert(ray.getName());
    alert(ray.getAge());
    ray.setName("Younger Ray");
    // Instant rejuvenation!
    ray.setAge(22);

    alert(ray.getName() + " is now " + ray.getAge() + 
          " years old.");

不必在构造器中进行初始化的私有成员可以声明为构造器函数的局部变量，就象这样：

    function Person(name, age) {
        var occupation;
        this.getOccupation = function() { return occupation; };
        this.setOccupation = function(newOcc) { occupation = 
                             newOcc; };
        // accessors for name and age    
    }

要注意的是，这样的私有成员同我们所认为的C\#中的私有成员稍有不同。在C\#中，类的公开方法可以直接访问类的私有成员。但是在JavaScript中，私有成员只有通过在闭包中包含有这些私有成员的方法来访问（这样的方法通常称为特权方法，因为它们不同于普通的公开方法）。因此，在Person的公开方法中，你依然可以通过Person的特权方法方法来访问私有成员：

    Person.prototype.somePublicMethod = function() {
        // doesn’t work!
        // alert(this.name);
        // this one below works
        alert(this.getName());
    };

大家广泛认为，Douglas Crockford是第一个发现（或者可能说发表更合适）使用闭包来模拟私有成员的人。他的网站，[javascript.crockford.com](http://javascript.crockford.com/)，包含了JavaScript方面的大量信息 —— 对JavaScript感兴趣的开发人员都应该去他的网站看看。

类的继承
--------

好的，现在你已经看到了如何通过构造器函数和原型对象在JavaScript中模拟类。你也已经了解原型链可以确保所有的对象都能具有Object.prototype中的通用方法。你还看到了如何使用闭包来模拟出私有成员。但是，这里好像还是缺点什么东西。你还没看到在JavaScript中如何实现类的继承；这在C\#中可是司空见惯的事情。很不幸，在JavaScript进行类的继承无法象在C\#中那样键入一个冒号而实现；在JavaScript中还需要做更多的事情。但从另一方面讲，因为JavaScript非常灵活，我们有多种途径实现类的继承。

比方说，如**图9**所示，你有一个基类叫Pet，它有一个派生类叫做Dog。怎样在JavaScript中实现这个继承关系呢？Pet类就很简单了，你已经看到过怎么实现它了：

![](/post-images/2013-06/20151043_zqob.gif) 图9** 类 **

    // class Pet
    function Pet(name) {
        this.getName = function() { return name; };
        this.setName = function(newName) { name = newName; };
    }

    Pet.prototype.toString = function() {
        return "This pet’s name is: " + this.getName();
    };

    // end of class Pet
    var parrotty = new Pet("Parrotty the Parrot");
    alert(parrotty);

那该如何定义派生自Pet类的Dog类呢？从**图9**中可看出，Dog类具有一个额外的属性，breed,，并且它还重载了Pet的toString方法（请注意，avaScript中的方法和属性命名惯例采用的是驼峰式大小写方式，即camel case；而C\#推荐使用的是Pascal大小写方式）。**图10**所示即为Pet类的定义实现方法：

图10 **继承Pet类**

    // class Dog : Pet 
    // public Dog(string name, string breed)
    function Dog(name, breed) {
        // think Dog : base(name) 
        Pet.call(this, name);
        this.getBreed = function() { return breed; };
        // Breed doesn’t change, obviously! It’s read only.
        // this.setBreed = function(newBreed) { name = newName; };
    }

    // this makes Dog.prototype inherits
    // from Pet.prototype
    Dog.prototype = new Pet();

    // remember that Pet.prototype.constructor
    // points to Pet. We want our Dog instances’
    // constructor to point to Dog.
    Dog.prototype.constructor = Dog;

    // Now we override Pet.prototype.toString
    Dog.prototype.toString = function() {
        return "This dog’s name is: " + this.getName() + 
            ", and its breed is: " + this.getBreed();

    };

    // end of class Dog

    var dog = new Dog("Buddy", "Great Dane");

    // test the new toString()
    alert(dog);

    // Testing instanceof (similar to the is operator)

    // (dog is Dog)? yes
    alert(dog instanceof Dog);

    // (dog is Pet)? yes
    alert(dog instanceof Pet);

    // (dog is Object)? yes
    alert(dog instanceof Object);

通过正确设置原型链这个小把戏，就可以同在C\#中所期望的那样，使得instanceof测试在JavaScript中也能够正常进行。而且如你所愿，特权方法也能够正常得以运行。

模拟命名空间
------------

在C++和C\#中，命名空间用来将命名冲突的可能性减小到最小的程度。例如，在.NET框架中，命名空间可以帮助我们区分出Microsoft.Build.Task.Message和Sys­tem.Messaging.Message这两个类。JavaScript并没有明确的语言特性来支持命名空间，但使用对象可以非常容易的模拟出命名空间。比如说你想创建一个JavaScript代码库。不想在全局中定义函数和类，你就可以将你的函数和类封装到如下这样的命名空间之中：

    var MSDNMagNS = {};

    MSDNMagNS.Pet = function(name) { // code here };
    MSDNMagNS.Pet.prototype.toString = function() { // code };

    var pet = new MSDNMagNS.Pet("Yammer");

只有一层命名空间可能会出现不唯一的请看，所以你可以创建嵌套的命名空间：

    var MSDNMagNS = {};

    // nested namespace "Examples"
    MSDNMagNS.Examples = {}; 

    MSDNMagNS.Examples.Pet = function(name) { // code };
    MSDNMagNS.Examples.Pet.prototype.toString = function() { // code };

    var pet = new MSDNMagNS.Examples.Pet("Yammer");

不难想象，每次都键入这些很长的嵌套命名空间很快就会让人厌烦。幸运的是，你的代码库的用户可以很容易地为你的命名空间起一个比较简洁的别名：

    // MSDNMagNS.Examples and Pet definition...
    // think "using Eg = MSDNMagNS.Examples;" 
    var Eg = MSDNMagNS.Examples;
    var pet = new Eg.Pet("Yammer");

    alert(pet);

你要是看一眼Microsoft AJAX代码库的源代码的话，就会发现该库的编写者也使用了类似的技巧来实现命名空间（请看静态方法Type.registerNamespace的实现代码)。这方面更详细的信息可参见"OOP and ASP.NET AJAX"的侧边栏。

你应该用这种方式来进行JavaScript编程吗？
---------------------------------------

如你所见，JavaScript对面向对象的支持非常好。虽然设计为基于原型的语言，但是它足够灵活也足够强大，允许你拿它来进行通常是出现在其它常用语言中的基于类的编程风格。但是问题在于：你是否应该以这种方式来进行JavaScript编码吗？你是否应该采用C\#或C++的编程方式，采用比较聪明的方式模拟出本来不存在的特性来进行JavaScript编程？每种编程语言都互不相同，一种语言的最佳实践对另外一种编程语言来讲可能就不实最佳的了。

你已经了解在JavaScript中是对象继承自对象（而非类继承自类）。所以，让大量的类使用静态的继承层次结构可能不是JavaScript之道。可能就象Douglas Crockford在他的这篇文章"[Prototypal Inheritance in JavaScript](http://javascript.crockford.com/prototypal.html)"中所说的那样，JavaScript的编程之道就是创建原型对象，并使用下面这样的简单的对象函数来创建继承自原对象的新对象：

    function object(o) {
        function F() {}
        F.prototype = o;
        return new F();
    }

然后，既然JavaScript对象可塑性很强，你就可以在对象生成之后，通过为它添加必要的新字段和新方法来增强对象。

这种做法都很不错，但不可否认的是，全世界大多数开发者都更加属性基于类的编程。实际上，基于类的编程还会继续流行下去。根据即将发布的ECMA-262规范（ECMA-262是JavaScript的官方规范）的第4个版本，JavaScript 2.0将具有真正的类。所以说，JavaScript正在逼近基于类的编程语言。然而，JavaScript 2.0要得到广泛使用可能还需要几年的时间。同时还有一点也很重要，就是要全面掌握当前版本的JavaScript，只有这样才能读懂和编写出基于原型和基于类的这两种风格的JavaScript代码。

大局观
------

随着交互式、重客户端AJAX应用的普及，JavaScript很快就成为了.NET开发者工具箱中最有用的工具之一。然而，对于更加适应C++、C\#或者Visual Basic等语言的开发者来讲，JavaScript的原型本性一开始会让它们感到很不适应。我觉得我的JavaScript之旅收获颇丰，但一直以来也不乏挫折打击。如果这篇文章能够帮助你更加顺利地进步，那么我将倍感欣慰，因为这就是我写这篇文章的目的所在。

OOP 和 ASP.NET AJAX
-------------------

ASP.NET AJAX中实现的OOP同我在这篇文章里讨论的规范的实现方法稍有不同。这里面主要有两个方面的原因：ASP.NET AJAX版的实现为反射（对于象xml-scrip这样的声明式语法并且为了参数验证，反射是很有必要的手段）提供了更多的可能，而且ASP.NET AJAX旨在将.NET开发者所熟悉的其它一些语法结构，比如属性、事件、枚举以及接口等翻译为JavaScript代码。

在当前广泛可用的版本中，JavaScript缺乏.NET开发者所熟知的大量OOP方面的概念，ASP.NET AJAX模拟出了其中的大部分概念。

类可用具有基于命名规范的属性访问器（下文中有例子），还可用完全按照.NET所提供的模式进行事件多播。私有变量的命名遵从以下划线打头的成员就是私有成员这样的规范。很少有必要使用真正私有的变量，这个策略使得我们可用从调试器中直接查看这种变量。引入接口也是为了进行类型检查，而不是通常的duck-typing（一种类型方案，其基于的概念是，如果有一种东西象鸭子那样走路并且象鸭子那样嘎嘎叫，我们就认为这种东西是鸭子，或者说可用把这种东西看作鸭子）。

### 类和反射

在JavaScript中，我们无法得知函数的名字。即使有可能可以得知，多数情况下这对我们来说也没有什么帮助，因为类构造器通常就是将一个匿名函数赋值为一个命名空间变量。真正的类型名的是由该变量的全限定名组成的，但却同样无法取得，构造器函数对此名也一无所知。为了克服此局限并在JavaScript类之中具有丰富的反射机制，ASP.NET AJAX要求要将类型的名字进行注册。

ASP.NET AJAX中的反射API可用于任何类型，无论该类型是内建的类、接口、命名空间、甚至是枚举都没有问题，而且其中还包含有和.NET框架中相同的isInstanceOfType和inheritsFrom函数，这两个函数用来在程序运行时对类的层次结构进行检视。ASP.NET AJAX在调试模式还做了类型检查，其意义在于能够帮助开发者尽早地找出程序中的bug。

### 注册类的层次结构和基类的调用

要在ASP.NET AJAX中定义一个类，你需要将该类的构造器函数赋值给一个变量（要注意构造器函数是如何调用基类的方法的）：

    MyNamespace.MyClass = function() {
        MyNamespace.MyClass.initializeBase(this);
        this._myProperty = null;
    }

然后，你需要在它的原型中定义该类的成员：

    MyNamespace.MyClass.prototype = {
        get_myProperty: function() { return this._myProperty;},
        set_myProperty: function(value) { this._myProperty = value; },
        doSomething: function() {
            MyNamespace.MyClass.callBaseMethod(this, "doSomething");
            /* do something more */
        }
    }

最后，你要对这个类进行注册：

    MyNamespace.MyClass.registerClass(
        "MyNamespace.MyClass ", MyNamespace.BaseClass);

构造器和原型的继承层次结构就不需要你管了，因为registerClass函数会为你完成此项任务。

