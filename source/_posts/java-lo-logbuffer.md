---
layout: post
category: default
date: 2013-08-08
title: "Java 日志缓存机制的实现"
description: "Java 日志缓存机制的实现"
tags: [java]
redirecturl: http://www.ibm.com/developerworks/cn/java/j-lo-logbuffer/index.html
---


概述
----

日志技术为产品的质量和服务提供了重要的支撑。JDK 在 1.4 版本以后加入了日志机制，为 Java 开发人员提供了便利。但这种日志机制是基于静态日志级别的，也就是在程序运行前就需设定下来要打印的日志级别，这样就会带来一些不便。

在 JDK 提供的日志功能中，日志级别被细化为 9 级，用以区分不同日志的用途，用来记录一个错误，或者记录正常运行的信息，又或是记录详细的调试信息。由于日志级别是静态的，如果日志级别设定过高，低级别的日志难以打印出来，从而导致在错误发生时候，难以去追踪错误的发生原因，目前常常采用的方式是在错误发生的时候，不得不先调整日志级别到相对低的程度，然后再去触发错误，使得问题根源得到显现。但是这种发生问题需要改动产品配置，然后重新触发问题进行调试的方式使得产品用户体验变差，而且有些问题会因为偶发性，环境很复杂等原因很难重新触发。

相反，如果起初就把日志级别调整到比较低，那么日志中间会有大量无用信息，而且当产品比较复杂的时候，会导致产生的日志文件很大，刷新很快，无法及时的记录有效的信息，甚至成为性能瓶颈，从而降低了日志功能对产品的帮助。

本文借助 Java Logging 中的 MemoryHandler 类将所有级别日志缓存起来，在适当时刻输出，来解决这个问题。主要围绕 MemoryHandler 的定义和logging.properties 文件的处理而展开。

实例依附的场景如下，设想用户需要在产品发生严重错误时，查看先前发生的包含 Exception 的错误信息，以此作为诊断问题缘由的依据。使用 Java 缓冲机制作出的一个解决方案是，将所有产品运行过程中产生的包含 Exception 的日志条目保存在一个可设定大小的循环缓冲队列中，当严重错误（SEVERE）发生时，将缓冲队列中的日志输出到指定平台，供用户查阅。

Java 日志机制的介绍
-------------------

Java 日志机制在很多文章中都有介绍，为了便于后面文章部分的理解，在这里再简单介绍一下本文用到的一些关键字。

Level：JDK 中定义了 Off、Severe、Warning、Info、Config、Fine、Finer、Finest、All 九个日志级别，定义 Off 为日志最高等级，All 为最低等级。每条日志必须对应一个级别。级别的定义主要用来对日志的严重程度进行分类，同时可以用于控制日志是否输出。

LogRecord：每一条日志会被记录为一条 LogRecord, 其中存储了类名、方法名、线程 ID、打印的消息等等一些信息。

Logger：日志结构的基本单元。Logger 是以树形结构存储在内存中的，根节点为 root。com.test（如果存在）一定是 com.test.demo（如果存在）的父节点，即前缀匹配的已存在的 logger 一定是这个 logger 的父节点。这种父子关系的定义，可以为用户提供更为自由的控制粒度。因为子节点中如果没有定义处理规则，如级别 handler、formatter 等，那么默认就会使用父节点中的这些处理规则。

Handler：用来处理 LogRecord，默认 Handler 是可以连接成一个链状，依次对 LogRecord 进行处理。

Filter：日志过滤器。在 JDK 中，没有实现。

Formatter：它主要用于定义一个 LogRecord 的输出格式。

**图 1. Java 日志处理流程**

![javarz01](/post-images/2013-08/javarz01.png "javarz01")

图 1 展示了一个 LogRecord 的处理流程。一条日志进入处理流程首先是 Logger，其中定义了可通过的 Level，如果 LogRecord 的 Level 高于Logger 的等级，则进入 Filter（如果有）过滤。如果没有定义 Level，则使用父 Logger 的 Level。Handler 中过程类似，其中 Handler 也定义了可通过 Level，然后进行 Filter 过滤，通过如果后面还有其他 Handler，则直接交由后面的 Handler 进行处理，否则会直接绑定到 formatter 上面输出到指定位置。

在实现日志缓存之前，先对 Filter 和 Formatter 两个辅助类进行介绍。

### Filter

Filter 是一个接口，主要是对 LogRecord 进行过滤，控制是否对 LogRecord 进行进一步处理，其可以绑定在 Logger 下或 Handler 下。

只要在 boolean isLoggable（LogRecord）方法中加上过滤逻辑就可以实现对 logrecord 进行控制，如果只想对发生了 Exception 的那些 log 记录进行记录，那么可以通过清单 1 来实现，当然首先需要将该 Filter 通过调用 setFilter（Filter）方法或者配置文件方式绑定到对应的 Logger 或 Handler。

**清单 1. 一个 Filter 实例的实现**

    @Override
     public boolean isLoggable(LogRecord record){ 
         if(record.getThrown()!=null){ 
                return true; 
         }else{ 
                 return false;  
         } 
     }

### Formatter

Formatter 主要是对 Handler 在输出 log 记录的格式进行控制，比如输出日期的格式，输出为 HTML 还是 XML 格式，文本参数替换等。Formatter 可以绑定到 Handler 上，Handler 会自动调用 Formatter 的 String format（LogRecord r） 方法对日志记录进行格式化，该方法具有默认的实现，如果想实现自定义格式可以继承 Formater 类并重写该方法，默认情况下例如清单 2 在经过 Formatter 格式化后，会将 {0} 和 {1} 替换成对应的参数。

**清单 2. 记录一条 log**

    logger.log(Level.WARNING,"this log is for test1: {0} and test2:{1}", 
        new Object[]{newTest1(), 
        new Test2()});

MemoryHandler
-------------

MemoryHandler 是 Java Logging 中两大类 Handler 之一，另一类是 StreamHandler，二者直接继承于 Handler，代表了两种不同的设计思路。Java Logging Handler 是一个抽象类，需要根据使用场景创建具体 Handler，实现各自的 publish、flush 以及 close 等方法。

MemoryHandler 使用了典型的“注册 – 通知”的观察者模式。MemoryHandler 先注册到对自己感兴趣的 Logger 中（logger.addHandler(handler)），在这些 Logger 调用发布日志的 API：log()、logp()、logrb() 等，遍历这些 Logger 下绑定的所有 Handlers 时，通知触发自身 publish（LogRecord）方法的调用，将日志写入 buffer，当转储到下一个日志发布平台的条件成立，转储日志并清空 buffer。

这里的 buffer 是 MemoryHandler 自身维护一个可自定义大小的循环缓冲队列，来保存所有运行时触发的 Exception 日志条目。同时在构造函数中要求指定一个 Target Handler，用于承接输出；在满足特定 flush buffer 的条件下，如日志条目等级高于 MemoryHandler 设定的 push level 等级（实例中定义为SEVERE）等，将日志移交至下一步输出平台。从而形成如下日志转储输出链：

**图 2. Log 转储链**

![javarz02](/post-images/2013-08/javarz02.jpg "javarz02")

在实例中，通过对 MemoryHandler 配置项 .push 的 Level 进行判断，决定是否将日志推向下一个 Handler，通常在 publish() 方法内实现。代码清单如下：

**清单 3**

    // 只纪录有异常并且高于 pushLevel 的 logRecord 
    final Level level = record.getLevel();        
    final Throwable thrown = record.getThrown(); 
    If(level >= pushLevel){ 
       push(); 
    }

### MemoryHandler.push 方法的触发条件

Push 方法会导致 MemoryHandler 转储日志到下一 handler，清空buffer。触发条件可以是但不局限于以下几种，实例中使用的是默认的第一种：

-   日志条目的 Level 大于或等于当前 MemoryHandler 中默认定义或用户配置的 pushLevel；
-   外部程序调用 MemoryHandler 的 push 方法；
-   MemoryHandler 子类可以重载 log 方法或自定义触发方法，在方法中逐一扫描日志条目，满足自定义规则则触发转储日志和清空 buffer 的操作。MemoryHanadler 的可配置属性

**表 1.MemoryHandler 可配置属性**

<table summary="" border="1" cellspacing="0" cellpadding="0">
<tbody>
<tr>
<th></th>
<th>属性名</th>
<th>描述</th>
<th>缺省值</th>
</tr>
<tr>
<td rowspan="4">继承属性</td>
<td>MemoryHandler.level</td>
<td>MemoryHandler 接受的输入到 buffer 的日志等级</td>
<td>Level.INFO</td>
</tr>
<tr>
<td>MemoryHandler.filter</td>
<td>在输入到 buffer 之前，可在 filter 中自定义除日志等级外的其他过滤条件</td>
<td>(Undefined)</td>
</tr>
<tr>
<td>MemoryHandler.formatter</td>
<td>指定输入至 buffer 的日志格式</td>
<td>(Undefined)</td>
</tr>
<tr>
<td>MemoryHandler.encoding</td>
<td>指定输入至 buffer 的日志编码，在 MemoryHandler 中应用甚少</td>
<td>(Undefined)</td>
</tr>
<tr>
<td rowspan="3">私有属性</td>
<td>MemoryHandler.size</td>
<td>以日志条目为单位定义循环 buffer 的大小</td>
<td>1,000</td>
</tr>
<tr>
<td>MemoryHandler.push</td>
<td>定义将 buffer 中的日志条目发送至下一个 Handler 的最低 Level（包含）</td>
<td>Level.SEVERE</td>
</tr>
<tr>
<td>MemoryHandler.target</td>
<td>在构造函数中指定下一步承接日志的 Handler</td>
<td>(Undefined)</td>
</tr>
</tbody>
</table>

**使用方式：**

以上是记录产品 Exception 错误日志，以及如何转储的 MemoryHandler 处理的内部细节；接下来给出 MemoryHandler 的一些使用方式。

**1. 直接使用 java.util.logging 中的 MemoryHandler**

**清单4**

    // 在 buffer 中维护 5 条日志信息
    // 仅记录 Level 大于等于 Warning 的日志条目并
    // 刷新 buffer 中的日志条目到 fileHandler 中处理
    int bufferSize = 5; 
    f = new FileHandler("testMemoryHandler.log"); 
    m = new MemoryHandler(f, bufferSize, Level.WARNING); 
    …
    myLogger = Logger.getLogger("com.ibm.test"); 
    myLogger.addHandler(m); 
    myLogger.log(Level.WARNING, “this is a WARNING log”);

**2. 自定义**

**1）反射**

思考自定义 MyHandler 继承自 MemoryHandler 的场景，由于无法直接使用作为父类私有属性的 size、buffer 及 buffer 中的 cursor，如果在 MyHandler 中有获取和改变这些属性的需求，一个途径是使用反射。清单 5 展示了使用反射读取用户配置并设置私有属性。

**清单5**

    int m_size; 
    String sizeString = manager.getProperty(loggerName + ".size"); 
    if (null != sizeString) { 
         try { 
          m_size = Integer.parseInt(sizeString); 
          if (m_size <= 0) { 
             m_size = BUFFER_SIZE; // default 1000 
          } 
          // 通过 java 反射机制获取私有属性
          Field f; 
          f = getClass().getSuperclass().getDeclaredField("size"); 
          f.setAccessible(true); 
          f.setInt(this, m_size); 
          f = getClass().getSuperclass().getDeclaredField("buffer"); 
          f.setAccessible(true); 
          f.set(this, new LogRecord[m_size]); 
         } catch (Exception e) { 
         } 
    }

**2）重写**

直接使用反射方便快捷，适用于对父类私有属性无频繁访问的场景。思考这样一种场景，默认环形队列无法满足我们存储需求，此时不妨令自定义的 MyMemoryHandler 直接继承 Handler，直接对存储结构进行操作，可以通过清单 6 实现。

**清单 6**

    public class MyMemoryHandler extends Handler{ 
      // 默认存储 LogRecord 的缓冲区容量
      private static final int DEFAULT_SIZE = 1000; 
      // 设置缓冲区大小
      private int size = DEFAULT_SIZE; 
      // 设置缓冲区
      private LogRecord[] buffer; 
      // 参考 java.util.logging.MemoryHandler 实现其它部分
      ... 
    }

使用 MemoryHandler 时需关注的几个问题
-------------------------------------

了解了使用 MemoryHandler 实现的 Java 日志缓冲机制的内部细节和外部应用之后，来着眼于两处具体实现过程中遇到的问题：Logger/Handler/LogRecord Level 的传递影响，以及如何在开发 MemoryHandler 过程中处理错误日志。

**1. Level 的传递影响**

Java.util.logging 中有三种类型的 Level，分别是 Logger 的 Level，Handler 的 Level 和 LogRecord 的 Level. 前两者可以通过配置文件设置。之后将日志的 Level 分别与 Logger 和 Handler 的 Level 进行比较，过滤无须记录的日志。在使用 Java Log 时需关注 Level 之间相互影响的问题，尤其在遍历 Logger 绑定了多个 Handlers 时。如图 3 所示：

**图 3. Java Log 中 Level 的传递影响**

![javarz03](/post-images/2013-08/javarz03.jpg "javarz03")

Java.util.logging.Logger 提供的 setUseParentHandlers 方法，也可能会影响到最终输出终端的日志显示。这个方法允许用户将自身的日志条目打印一份到 Parent Logger 的输出终端中。缺省会打印到 Parent Logger 终端。此时，如果 Parent Logger Level 相关的设置与自身 Logger 不同，则打印到 Parent Logger 和自身中的日志条目也会有所不同。如图 4 所示：

**图 4. 子类日志需打印到父类输出终端**

!["javarz04"](/post-images/2013-08/javarz04.jpg "javarz04")

**2. 开发 log 接口过程中处理错误日志**

在开发 log 相关接口中调用自身接口打印 log，可能会陷入无限循环。Java.util.logging 中考虑到这类问题，提供了一个 ErrorManager 接口，供 Handler 在记录日志期间报告任何错误，而非直接抛出异常或调用自身的 log 相关接口记录错误或异常。Handler 需实现 setErrorManager() 方法，该方法为此应用程序构造 java.util.logging.ErrorManager 对象，并在错误发生时，通过 reportError 方法调用 ErrorManager 的 error 方法，缺省将错误输出到标准错误流，或依据 Handler 中自定义的实现处理错误流。关闭错误流时，使用 Logger.removeHandler 移除此 Handler 实例。

两种经典使用场景，一种是自定义 MyErrorManager，实现父类相关接口，在记录日志的程序中调用 MyHandler.setErrorManager(new MyEroorManager()); 另一种是在 Handler 中自定义 ErrorManager 相关方法，示例如清单 7：

**清单 7**

    public class MyHandler extends Handler{ 
    // 在构造方法中实现 setErrorManager 方法
    public MyHandler(){ 
       ......
        setErrorManager (new ErrorManager() { 
            public void  error (String msg, Exception ex, int code) { 
                System.err.println("Error reported by MyHandler "
                                 + msg + ex.getMessage()); 
            } 
        }); 
    } 
    public void publish(LogRecord record){ 
        if (!isLoggable(record)) return; 
        try { 
            // 一些可能会抛出异常的操作
        } catch(Exception e) { 
            reportError ("Error occurs in publish ", e, ErrorManager.WRITE_FAILURE); 
        } 
    } 
    ......
    }

logging.properties
------------------

logging.properties 文件是 Java 日志的配置文件，每一行以“key=value”的形式描述，可以配置日志的全局信息和特定日志配置信息，清单 8 是我们为测试代码配置的 logging.properties。

**清单 8. logging.properties 文件示例**

    #Level 等级 OFF > SEVERE > WARNING > INFO > CONFIG > FINE > FINER > FINEST > ALL 
    # 为 FileHandler 指定日志级别
    java.util.logging.FileHandler.level=WARNING 
    # 为 FileHandler 指定 formatter 
    java.util.logging.FileHandler.formatter=java.util.logging.SimpleFormatter 
    # 为自定义的 TestMemoryHandler 指定日志级别
    com.ibm.test.MemoryHandler.level=INFO 
    # 设置 TestMemoryHandler 最多记录日志条数
    com.ibm.test.TestMemoryHandler.size=1000
    # 设置 TestMemoryHandler 的自定义域 useParentLevel 
    com.ibm.test.TestMemoryHandler.useParentLevel=WARNING 
    # 设置特定 log 的 handler 为 TestMemoryHandler 
    com.ibm.test.handlers=com.ibm.test.TestMemoryHandler 
    # 指定全局的 Handler 为 FileHandler 
    handlers=java.util.logging.FileHandler

从 清单 8 中可以看出 logging.properties 文件主要是用来给 logger 指定等级（level），配置 handler 和 formatter 信息。

### 如何监听 logging.properties

如果一个系统对安全性要求比较高，例如系统需要对更改 logging.properties 文件进行日志记录，记录何时何人更改了哪些记录，那么应该怎么做呢？

这里可以利用 JDK 提供的 PropertyChangeListener 来监听 logging.properties 文件属性的改变。

例如创建一个 LogPropertyListener 类，其实现了 java.benas.PropertyChangeListener 接口，PropertyChangeListener 接口中只包含一个 propertyChange（PropertyChangeEvent）方法，该方法的实现如清 9 所示。

**清单 9. propertyChange 方法的实现**

    @Override 
    public void propertyChange(PropertyChangeEvent event) { 
       if (event.getSource() instanceof LogManager){ 
           LogManager manager=(LogManager)event.getSource(); 
           update(manager); 
           execute(); 
           reset(); 
       } 
    }

propertyChange（PropertyChangeEvent）方法中首先调用 update（LogManager）方法来找出 logging.properties 文件中更改的，增加的以及删除的项，这部分代码如清单 10 所示；然后调用 execute() 方法来执行具体逻辑，参见 清单 11；最后调用 reset() 方法对相关属性保存以及清空，如 清单 12 所示。

**清单 10. 监听改变的条目**

    public void update(LogManager manager){ 
     Properties logProps = null ; 
      // 使用 Java 反射机制获取私有属性
       try { 
         Field f = manager.getClass().getDeclaredField("props"); 
         f.setAccessible(true ); 
         logProps=(Properties)f.get(manager); 
        }catch (Exception e){ 
           logger.log(Level.SEVERE,"Get private field error.", e); 
            return ; 
       } 
       Set<String> logPropsName=logProps.stringPropertyNames(); 
        for (String logPropName:logPropsName){ 
            String newVal=logProps.getProperty(logPropName).trim(); 
           // 记录当前的属性
           newProps.put(logPropName, newVal);   
           // 如果给属性上次已经记录过
           if (oldProps.containsKey(logPropName)){ 
                String oldVal = oldProps.get(logPropName); 
                if (newVal== null ?oldVal== null :newVal.equals(oldVal)){ 
               // 属性值没有改变，不做任何操作
            }else { 
                changedProps.put(logPropName, newVal); 
           } 
           oldProps.remove(logPropName); 
       }else {// 如果上次没有记录过该属性，则其应为新加的属性，记录之
            changedProps.put(logPropName, newVal);               
           } 
        } 
    }

代码中 oldProps、newProps 以及 changedProps 都是 HashMap类型，oldProps 存储修改前 logging.properties 文件内容，newProps 存储修改后 logging.properties 内容，changedProps 主要用来存储增加的或者是修改的部分。

方法首先通过 Java 的反射机制获得 LogManager 中的私有属性 props（存储了 logging.properties 文件中的属性信息），然后通过与 oldProps 比较可以得到增加的以及修改的属性信息，最后 oldProps 中剩下的就是删除的信息了。

**清单 11. 具体处理逻辑方法**

    private void execute(){ 
     // 处理删除的属性
     for (String prop:oldProps.keySet()){ 
       // 这里可以加入其它处理步骤
       logger.info("'"+prop+"="+oldProps.get(prop)+"'has been removed");           
     } 
     // 处理改变或者新加的属性
     for (String prop:changedProps.keySet()){ 
         // 这里可以加入其它处理步骤
         logger.info("'"+prop+"="+oldProps.get(prop)+"'has been changed or added"); 
     } 
    }

该方法是主要的处理逻辑，对修改或者删除的属性进行相应的处理，比如记录属性更改日志等。这里也可以获取当前系统的登录者，和当前时间，这样便可以详细记录何人何时更改过哪个日志条目。

**清单 12. 重置所有数据结构**

    private void reset(){ 
        oldProps = newProps; 
        newProps= new HashMap< String,String>(); 
        changedProps.clear(); 
    }

reset() 方法主要是用来重置各个属性，以便下一次使用。

当然如果只写一个 PropertyChangeListener 还不能发挥应有的功能，还需要将这个 PropertyChangeListener 实例注册到 LogManager 中，可以通过清单 13 实现。

**清单 13. 注册 PropertyChangeListener**

    // 为'logging.properties'文件注册监听器
    LogPropertyListener listener= new LogPropertyListener(); 
    LogManager.getLogManager().addPropertyChangeListener(listener);

### 如何实现自定义标签

在 清单 8中有一些自定义的条目，比如 com.ibm.test.TestMemoryHandler。

useParentLever=WARNING”，表示如果日志等级超过 useParentLever 所定义的等级 WARNING 时，该条日志在 TestMemoryHandler 处理后需要传递到对应 Log 的父 Log 的 Handler 进行处理（例如将发生了 WARNING 及以上等级的日志上下文缓存信息打印到文件中），否则不传递到父 Log 的 Handler 进行处理，这种情况下如果不做任何处理，Java 原有的 Log 机制是不支持这种定义的。那么如何使得 Java Log 支持这种自定义标签呢？这里可以使用 PropertyListener 对自定义标签进行处理来使得 Java Log 支持这种自定义标签，例如对“useParentLever”进行处理可以通过清单 14 实现。

**清单 14**

    private void execute(){ 
        // 处理删除的属性
        for (String prop:oldProps.keySet()){ 
            if (prop.endsWith(".useParentLevel")){ 
               String logName=prop.substring(0, prop.lastIndexOf(".")); 
               Logger log=Logger.getLogger(logName); 
                for (Handler handler:log.getHandlers()){ 
                    if (handler  instanceof TestMemoryHandler){ 
                       ((TestMemoryHandler)handler) 
                           .setUseParentLevel(oldProps.get(prop)); 
                        break ; 
                   } 
               } 
           } 
       } 
       // 处理改变或者新加的属性
       for (String prop:changedProps.keySet()){ 
           if (prop.endsWith(".useParentLevel")){ 
               // 在这里添加逻辑处理步骤
           } 
       } 
    }

在清单 14 处理之后，就可以在自定义的 TestMemoryHandler 中进行判断了，对 log 的等级与其域 useParentLevel 进行比较，决定是否传递到父 Log 的 Handler 进行处理。在自定义 TestMemoryHandler 中保存对应的 Log 信息可以很容易的实现将信息传递到父 Log 的 Handler，而保存对应 Log 信息又可以通过 PropertyListener 来实现，例如清单 15 更改了 清单 13中相应代码实现这一功能。

**清单 15**

    if (handler  instanceof TestMemoryHandler){ 
        ((TestMemoryHandler)handler).setUseParentLevel(oldProps.get(prop)); 
        ((TestMemoryHandler)handler).addLogger(log); 
          break ; 
    }

具体如何处理自定义标签的值那就看程序的需要了，通过这种方法就可以很容易在 logging.properties 添加自定义的标签了。

### 自定义读取配置文件

如果 logging.properties 文件更改了，需要通过调用 readConfiguration（InputStream）方法使更改生效，但是从 JDK 的源码中可以看到 readConfiguration（InputStream）方法会重置整个 Log 系统，也就是说会把所有的 log 的等级恢复为默认值，将所有 log 的 handler 置为 null 等，这样所有存储的信息就会丢失。

比如，TestMemoryHandler 缓存了 1000 条 logRecord，现在用户更改了 logging.properties 文件，并且调用了 readConfiguration（InputStream） 方法来使之生效，那么由于 JDK 本身的 Log 机制，更改后对应 log 的 TestMemoryHandler 就是新创建的，那么原来存储的 1000 条 logRecord 的 TestMemoryHandler 实例就会丢失。

那么这个问题应该如何解决呢？这里给出三种思路：

1). 由于每个 Handler 都有一个 close() 方法（任何继承于 Handler 的类都需要实现该方法），Java Log 机制在将 handler 置为 null 之前会调用对应 handler 的 close() 方法，那么就可以在 handler（例如 TestMemoryHandler）的 close() 方法中保存下相应的信息。

2). 研究 readConfiguration（InputStream）方法，写一个替代的方法，然后每次调用替代的方法。

3). 继承 LogManager 类，覆盖 readConfiguration（InputStream）方法。

这里第一种方法是保存原有的信息，然后进行恢复，但是这种方法不是很实用和高效；第二和第三种方法其实是一样的，都是写一个替代的方法，例如可以在替代的方法中对 Handler 为 TestMemoryHandler 的不置为 null，然后在读取 logging.properties 文件时发现为 TestMemoryHandler 属性时，找到对应 TestMemoryHandler 的实例，并更改相应的属性值（这个在清单 14 中有所体现），其他不属于 TestMemoryHandler 属性值的可以按照 JDK 原有的处理逻辑进行处理，比如设置 log 的 level 等。

另一方面，由于 JDK1.6 及之前版本不支持文件修改监听功能，每次修改了 logging.properties 文件后需要显式调用 readConfiguration（InputStream）才能使得修改生效，但是自 JDK1.7 开始已经支持对文件修改监听功能了，主要是在 java.nio.file.\* 包中提供了相关的 API，这里不再详述。

那么在 JDK1.7 之前，可以使用 apache 的 commons-io 库中的 FileMonitor 类，在此也不再详述。

总结
----

通过对 MemoryHandler 和 logging.properties 进行定义，可以通过 Java 日志实现自定义日志缓存，从而提高 Java 日志的可用性，为产品质量提供更强有力的支持。
