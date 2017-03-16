---
layout: post
title:  Java 多线程：volatile关键字
date:  2016-08-08 13:50:39
categories: java

---

#### Java 多线程：volatile关键字

##### 概念
volatile 也是 多线程的解决方案之一。***volatile 能够保证 可见性，但是不能保证原子性。它只能作用于变量，不能作用于方法。***

如何理解 volatile 能够保证可见性：

首先，需要了解一个概念：缓存行 - 缓存中可以分配的最小存储单位。当我们对 volatile 变量进行写操作的时候，把 Java 代码翻译到汇编代码的时候，会多出一个 Lock 前缀的指令,它在多核处理下会引发两个事情：

1. 将当前处理器缓存行的数据会写回到系统内存。
2. 这个写回内存的操作会引起在其他CPU里缓存了该内存地址的数据无效。

再细节一点说：

每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里。

具体可以参考：聊聊并发（一）[深入分析Volatile的实现原理](http://ifeve.com/volatile/)

##### volatile 的使用

使用volatile 有两点需要注意的地方：
1. 运算结果并不依赖于当前值，或者能确保只有单一的线程能够修改变量的值。
2. 变量不需要和其他的状态变量共同参与不变约束

##### 对于第一点的理解：

```
{% highlight ruby %}
public class Test {
    public static volatile int i = 0;
    public static void main(String args[]){

        new Thread(new Runnable(){
            public void run(){
                for(int j = 0; j < 10000; j++)
                    i++;
                System.out.println("Thread1 end...");
            }
        }).start();

        new Thread(new Runnable(){
            public void run(){
                for(int j = 0; j < 10000; j++)
                    i++;
                System.out.println("Thread2 end...");
            }
        }).start(); 

        i++;

        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        System.out.println("i = " + i);
    }
}
```
##### 对于第二点的理解：

```
{% highlight ruby %}
private Date start;      
private Date end;      

public void setInterval(Date newStart, Date newEnd) {      
    // 检查start<end是否成立, 在给start赋值之前不变式是有效的      
    start = newStart;      

    // 但是如果另外的线程在给start赋值之后给end赋值之前时检查start<end, 该不变式是无效的      

    end = newEnd;      
    // 给end赋值之后start<end不变式重新变为有效      
}  
```
最后，关于什么时候使用 volatile，一般是用来当做标记来使用。比如说，当shutdown() 方法被调用的时候，所有的 doWork() 方法都会停下来。


##### 备注
任何 对该变量的读写都会绕过 高速缓存，直接读取主内存的变量的值。
这句话应该有问题。

http://ifeve.com/volatile/
有volatile变量修饰的共享变量进行写操作的时候在多核处理器下会引发了两件事情：

1. 将当前处理器缓存行的数据会写回到系统内存。
2. 这个写回内存的操作会引起在其他CPU里缓存了该内存地址的数据无效。
