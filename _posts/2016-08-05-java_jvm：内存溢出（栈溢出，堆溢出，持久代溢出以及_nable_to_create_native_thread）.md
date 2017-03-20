---
layout: post
title: JVM 内存溢出
date:  2016-08-05
categories: java

---

### java JVM：内存溢出（栈溢出，堆溢出，持久代溢出以及 unable to create native thread） 

包括：

* 栈溢出(StackOverflowError)
* 堆溢出(OutOfMemoryError:java heap space)
* 永久代溢出(OutOfMemoryError: PermGen space)
*  OutOfMemoryError:unable to create native thread

Java虚拟机规范规定JVM的内存分为了好几块，比如`堆`，`栈`，`程序计数器`，`方法区`等，而Hotspot jvm的实现中，将堆内存分为了两部：新生代，老年代。在堆内存之外，还有永久代，其中永久代实现了规范中规定的方法区，而内存模型中不同的部分都会出现相应的OOM错误，接下来我们就分开来讨论一下。

---

### 栈溢出(StackOverflowError)
###### 表现：

栈溢出抛出StackOverflowError错误

###### 产生原因：
出现此种情况是因为方法运行的时候栈的深度超过了虚拟机容许的最大深度所致。出现这种情况，一般情况下是程序错误所致的，比如写了一个死递归，就有可能造成此种情况。 下面我们通过一段代码来模拟一下此种情况的内存溢出。

```
import java.util.*;    
import java.lang.*;    
public class OOMTest{     
    public void stackOverFlowMethod(){    
        stackOverFlowMethod();    
    }    
    public static void main(String... args){    
        OOMTest oom = new OOMTest();    
        oom.stackOverFlowMethod();    
    }    
}    

```

运行上面的代码，一段时间之后会抛出如下的异常：

**
Exception in thread "main" java.lang.StackOverflowError at OOMTest.stackOverFlowMethod(OOMTest.java:6) 
**      

对于栈内存溢出，根据[《Java 虚拟机规范》](http://download.csdn.net/detail/yao__shun__yu/5067596)中文版：如果线程请求的栈容量超过栈允许的最大容量的话，Java 虚拟机将抛出一个StackOverflow异常；如果Java虚拟机栈可以动态扩展，并且扩展的动作已经尝试过，但是无法申请到足够的内存去完成扩展，或者在新建立线程的时候没有足够的内存去创建对应的虚拟机栈，那么Java虚拟机将抛出一个OutOfMemory 异常。

###### 解决方案：

1. 增大堆栈大小值
2. 使用static对象替代nonstatic局部对象
3. 用栈把递归转换成非递归  


---  

### 堆溢出(OutOfMemoryError:java heap space) 
  
###### 表现：

堆内存溢出的时候，虚拟机会抛出java.lang.OutOfMemoryError:java heap space

###### 产生原因：

出现此种情况的时候，我们需要根据内存溢出的时候产生的dump文件来具体分析。出现此种问题的时候有可能是内存泄露，也有可能是内存溢出了。通过以下代码可以模拟堆内存溢出  

```
import java.util.*;    
import java.lang.*;    
public class OOMTest{    
    public static void main(String... args){    
        List<byte[]> buffer = new ArrayList<byte[]>();    
        buffer.add(new byte[10*1024*1024]);    
    }    
}    

```
我们通过如下的命令运行上面的代码：

``java -verbose:gc -Xmn10M -Xms20M -Xmx20M -XX:+PrintGC OOMTest***``

程序输出如下的信息：

Exception in thread "main" java.lang.OutOfMemoryError: PermGen space at java.lang.String.intern(Native Method) at OOMTest.main(OOMTest.java:8)   	
        
        
通过上面的代码，我们成功模拟了运行时常量池溢出的情况，从输出中的PermGen space可以看出确实是持久带发生了溢出，这也验证了，我们前面说的Hotspot jvm通过持久带来实现方法区的说法。

###### 解决方案：

* 如果内存泄露，我们要找出泄露的对象是怎么被GC ROOT引用起来，然后通过引用链来具体分析泄露的原因。
* 如果出现了内存溢出问题，这往往是程序本生需要的内存大于了我们给虚拟机配置的内存，这种情况下，我们可以采用增加-XX:+HeapDumpOnOutOfMemoryErrorjvm启动参数。下面我们通过如下的代码来演示一下此种情况的溢出：

---

### OutOfMemoryError:unable to create native thread
###### 表现：
最后我们在来看看java.lang.OutOfMemoryError:unable to create natvie thread这种错误。 

###### 产生原因：
出现这种情况的时候，一般是下面两种情况导致的：

1. 程序创建的线程数超过了操作系统的限制。对于Linux系统，我们可以通过ulimit -u来查看此限制。
2. 给虚拟机分配的内存过大，导致创建线程的时候需要的native内存太少。


###### 解决方案：

我们都知道操作系统对每个进程的内存是有限制的，我们启动Jvm,相当于启动了一个进程，假如我们一个进程占用了4G的内存，那么通过下面的公式计算出来的剩余内存就是建立线程栈的时候可以用的内存。线程栈总可用内存=4G-（-Xmx的值）- （-XX:MaxPermSize的值）- 程序计数器占用的内存
通过上面的公式我们可以看出，-Xmx 和 MaxPermSize的值越大，那么留给线程栈可用的空间就越小，在-Xss参数配置的栈容量不变的情况下，可以创建的线程数也就越小。因此如果是因为这种情况导致的unable to create native thread,那么要么我们增大进程所占用的总内存，或者减少-Xmx或者-Xss来达到创建更多线程的目的。

---

### 总结

1. 栈内存溢出：程序所要求的栈深度过大导致。
2. 堆内存溢出： 分清 内存泄露还是 内存容量不足。泄露则看对象如何被 GC Root 引用。不足则通过 调大 -Xms，-Xmx参数。
3. 持久带内存溢出：Class对象未被释放，Class对象占用信息过多，有过多的Class对象。
4. 无法创建本地线程：总容量不变，堆内存，非堆内存设置过大，会导致能给线程的内存不足。
5. 对于内存优化这部分可以借助orale java bin目录下的工具以及第三方软件进行调试，将会在接下来的文章中提到

该文章转载自  [Java常见内存溢出异常分析](http://www.importnew.com/14604.html) 并进行修改

### 写在最后
1. 如有不足希望大家指正。
2. 虽然现在计算机硬件技术有着飞跃的发展，很多应用程序不用太关注内存方面优化，但还是希望能再自己的项目中用到这些知识


  
      