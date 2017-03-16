
---
layout: post
title: Java 多线程：Condition关键字
date: 2016-08-08 21:57:11
categories: java

---

#### Java 多线程：Condition关键字

##### 前言
***Condition 是一种更细粒度的并发解决方案。***

就拿生产者消费者模式来说，当仓库满了的时候，又再执行到 生产者 线程的时候，会把 该 生产者 线程进行阻塞，再唤起一个线程.

***但是此时唤醒的是消费者线程还是生产者线程，是未知的。***如果再次唤醒的还是生产者线程，那么还需要把它进行阻塞，再唤起一个线程，再此循环，直到唤起的是消费者线程。这样就可能存在 时间或者资源上的浪费，

所以说 有了Condition 这个东西。***Condition 用 await() 代替了 Object 的 wait() 方法，用 signal() 方法代替 notify() 方法。***注意：Condition 是被绑定到 Lock 中，要创建一个 Lock 的 Condition 必须用 newCondition() 方法。

###### Condition 在生产者消费者模型中的代码Demo

```
class BoundedBuffer {      
    final Lock lock = new ReentrantLock();//锁对象      
    final Condition notFull  = lock.newCondition();//写线程条件       
    final Condition notEmpty = lock.newCondition();//读线程条件       

    final Object[] items = new Object[100];//缓存队列      
    int putptr/*写索引*/, takeptr/*读索引*/, count/*队列中存在的数据个数*/;      

    public void put(Object x) throws InterruptedException {      
        lock.lock();      
        try {      
            while (count == items.length)//如果队列满了       
                notFull.await();//阻塞写线程      
            items[putptr] = x;//赋值       
            if (++putptr == items.length) putptr = 0;//如果写索引写到队列的最后一个位置了，那么置为0      
            ++count;//个数++      
            notEmpty.signal();//唤醒读线程      
        } finally {      
            lock.unlock();      
        }      
    }      

    public Object take() throws InterruptedException {      
        lock.lock();      
        try {      
            while (count == 0)//如果队列为空      
                notEmpty.await();//阻塞读线程      
            Object x = items[takeptr];//取值       
            if (++takeptr == items.length) takeptr = 0;//如果读索引读到队列的最后一个位置了，那么置为0      
            --count;//个数--      
            notFull.signal();//唤醒写线程      
            return x;      
        } finally {      
            lock.unlock();      
        }      
    }       
} 
```

这是一个处于多线程工作环境下的缓存区，缓存区提供了两个方法，put和take，put是存数据，take是取数据，内部有个缓存队列，具体变量和方法说明见代码，这个缓存区类实现的功能：有多个线程往里面存数据和从里面取数据，其缓存队列(先进先出后进后出)能缓存的最大数值是100，多个线程间是互斥的，当缓存队列中存储的值达到100时，将写线程阻塞，并唤醒读线程，当缓存队列中存储的值为0时，将读线程阻塞，并唤醒写线程，下面分析一下代码的执行过程：


1. 一个写线程执行，调用put方法；
2. 判断count是否为100，显然没有100；
3. 继续执行，存入值；
4. 判断当前写入的索引位置++后，是否和100相等，相等将写入索引值变为0，并将count+1；
5. 仅唤醒读线程阻塞队列中的一个；
6. 一个读线程执行，调用take方法；
7. ……
8. 仅唤醒写线程阻塞队列中的一个。

这样由于阻塞该阻塞的，唤醒该唤醒的，阻塞该阻塞的，所以能提高效率。