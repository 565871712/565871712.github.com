---
layout: post
title: Java 多线程：synchronized 关键字用法
date: 2016-08-08 21:57:11
categories: java

---


#### Java 多线程：synchronized 关键字用法
修饰类，方法，静态方法，代码块

##### 前言
在 多线程生成的原因（Java内存模型与i++操作解析） 中，介绍了Java的内存模型，从而可能导致的多线程问题。synchronized就是避免这个问题的解决方法之一。除了 synchronized 的方式，还有 lock，condition，volatile，threadlocal，atomicInteger，cas等方式。

##### synchronized 用法
它的修饰对象有几种：

1. 修饰一个类，其作用的范围是synchronized后面括号括起来的部分，作用的对象是这个类的所有对象。
2. 修饰一个方法，被修饰的方法称为同步方法，其作用的范围是整个方法，作用的对象是调用这个方法的对象；
3. 修改一个静态的方法，其作用的范围是整个静态方法，作用的对象是这个类的所有对象；
4. 修饰一个代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码，作用的对象是调用这个代码块的对象；

##### 修饰一个类

其作用的范围是synchronized后面括号括起来的部分，作用的对象是这个类的所有对象，如下代码：

```
class ClassName {
   public void method() {
      synchronized(ClassName.class) {
         // todo
      }
   }
}
```

##### 修饰一个方法
Synchronized修饰一个方法很简单，就是在方法的前面加synchronized，例如：

```
public synchronized void method()
{
   // todo
}
```

另外，有几点需要注意：

1. 在定义接口方法时不能使用synchronized关键字。
2. 构造方法不能使用synchronized关键字，但可以使用synchronized代码块来进行同步。 
3. synchronized 关键字不能被继承。如果子类覆盖了父类的 被 synchronized 关键字修饰的方法，那么子类的该方法只要没有 synchronized 关键字，那么就默认没有同步，也就是说，不能继承父类的 synchronized。

##### 修饰静态方法
我们知道静态方法是属于类的而不属于对象的。同样的，synchronized修饰的静态方法锁定的是这个类的所有对象。如下：

```
public synchronized static void method() {
   // todo
}
```

##### 修饰代码块

* 当两个并发线程访问同一个对象object中的这个synchronized(this)同步代码块时，一个时间内只能有一个线程得到执行。另一个线程必须等待当前线程执行完这个代码块以后才能执行该代码块。
* 当一个线程访问object的一个synchronized(this)同步代码块时，另一个线程仍然可以访问该object中的非synchronized(this)同步代码块。
* 尤其关键的是，当一个线程访问object的一个synchronized(this)同步代码块时，其他线程对object中所有其它synchronized(this)同步代码块的访问将被阻塞。
* 第三个例子同样适用其它同步代码块。也就是说，当一个线程访问object的一个synchronized(this)同步代码块时，它就获得了这个object的对象锁。结果，其它线程对该object对象所有同步代码部分的访问都被暂时阻塞。
* 以上规则对其它对象锁同样适用.

##### JDK中对 synchronized 的使用举例

对于 synchronized ，个人觉得是一种比较粗糙的加锁，尤其是对整个对象，或者整个类进行加锁的时候。例如，HashTable，它相当于 HashMap的线程安全版本。实现如下：

```
public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
     
```

可以看到，很多方法都是使用了 synchronized 进行了修饰，那么就意味如果还有别的同步方法x，这个x方法和get方法即使在没有冲突的情况下也需要等待执行。这样效率上 必然会有一定的影响，所以会有 ConcurrentHashMap 进行分段加锁。

另外，在JDK中，Collections有一个方法可以把不是线程安全的集合转化为线性安全的集合，它是这样实现的：

```
public static <T> Collection<T> synchronizedCollection(Collection<T> c) {
        return new SynchronizedCollection<>(c);
    }
```

```
static class SynchronizedCollection<E> implements Collection<E>, Serializable {
        private static final long serialVersionUID = 3053995032091335093L;

        final Collection<E> c;  // Backing Collection
        final Object mutex;     // Object on which to synchronize

        SynchronizedCollection(Collection<E> c) {
            this.c = Objects.requireNonNull(c);
            mutex = this;
        }

        SynchronizedCollection(Collection<E> c, Object mutex) {
            this.c = Objects.requireNonNull(c);
            this.mutex = Objects.requireNonNull(mutex);
        }

        public int size() {
            synchronized (mutex) {return c.size();}
        }

        ......
        
```
可以看到 在构造函数中 有 mutex = this, 然后在具体的方法使用了 synchronized(mutex)，这样就会对调用该方法的对象进行加锁。还是会出现HashTable 出现的那种问题，也就是效率不高。

以上就是 JDK 对于 synchronized 用法的简单举例。            