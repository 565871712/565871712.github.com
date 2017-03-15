
---
layout: post
title:  Java 多线程：ThreadLocal关键字
date:  2016-08-08 13:50:39
categories: java

---

#### Java 多线程：ThreadLocal关键字

##### 什么是ThreadLocal

ThreadLocal并非一个线程，而是一个线程局部变量。***它的作用就是为使用该变量的线程都提供一个变量值的副本，每个线程都可以独立的改变自己的副本，而不会和其他线程的副本造成冲突。***

从线程的角度看，每个线程都保持一个对其线程局部变量副本的隐式引用，只要线程是活动的并且 ThreadLocal 实例是可访问的；在线程消失之后，其线程局部实例的所有副本都会被垃圾回收（除非存在对这些副本的其他引用）。

通过ThreadLocal存取的数据，总是与当前线程相关，也就是说，JVM 为每个运行的线程，绑定了私有的本地实例存取空间，***从而为多线程环境常出现的并发访问问题提供了一种隔离机制。***

概括起来说，对于多线程资源共享的问题，同步机制采用了“以时间换空间”的方式，而ThreadLocal采用了“以空间换时间”的方式。前者仅提供一份变量，让不同的线程排队访问，而后者为每一个线程都提供了一份变量，因此可以同时访问而互不影响。

##### 实际运用中使用的 ThreadLocal

那么，在实际运用中，有这样的需求：要得到当前用户。当然，方法有很多，比如说session，cookie 或者什么的。对于本人来说，由于使用了 OAuth，每个用户都会有一个 Access Token。token 的意思就是 代表了该用户有没有权限访问 后台系统，是唯一的。那么，此时就可以根据这个 token 来得到当前用户。

具体做法： 在生成token的时候，会保存一个关联，即 用户-token 关联。那么在用户访问的时候，就可以使用拦截器 得到该用户的 token，然后查找关联表，从而得到用户，最后再放入到 ThreadLocal 变量中，以供 后续步骤可以拿到。

关键代码如下：

拦截器中得到 token，查找用户：

```
String accessToken = httpServletRequest.getHeader("Authorization");
if(!StringUtils.isEmpty(accessToken)){
    String token[] = accessToken.split(" ");

    Staff staff = staffService.findStaffByAccessToken(token[1]);
    if(staff != null){
        StaffUtils.staffHolder.set(staff);
        log.debug("CurrentStaffFilter doFilter function ...access_token = {}. and staff in the threadlocal = {}.",accessToken,StaffUtils.staffHolder.get());
    } else {
        log.debug("CurrentStaffFilter doFilter function ...access_token = {}. But staff is null",accessToken);
    }
} else {
    log.debug("CurrentStaffFilter doFilter function ...access_token is null");
}

//该 filter 不影响正常调用，所以最后放行
chain.doFilter(request, response);
```

可以得到，可以首先得到 token， 然后找出 staff，最后进行

```
StaffUtils.staffHolder.set(staff);
```
也就是把 staff 放入到 ThreadLocal中， StaffUtils 代码如下：

```
public class StaffUtils {
    public static ThreadLocal<Staff> staffHolder = new ThreadLocal<Staff>();

    public static Staff getCurrentStaff(){
        return staffHolder.get();
    }
}
```
这样，在后续的代码中，就可以直接调用 getCurrentStaff() 得到当前用户。因为是在同一个线程中

##### ThreadLocal 原理
我们从我们怎么用，一步一步分析原理。首先，我们会定义一个 ThreadLocal，如下代码：

```
public static ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>();
```

然后，我们会把我们的值放入到 ThreadLocal 中，如下代码：

```
threadLocal.set(new Integer(123));
```

这里就表示了 这个 123 的 Integer 对象就和我们的现在这个线程关联起来了，以后我们要使用这个 123 的 Integer 对象，那么我们直接使用如下代码即可返回 123 的 Integer 对象：

```
threadLocal.get()

```
##### set()
那么，在 set 的时候，发生了什么，值 和 线程是如何关联起来的：

```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
这里，首先 通过 Thread.currentThread() 拿到当前线程，然后 通过 getMap(t) 方法拿到一个 ThreadLocalMap 类型的 map 对象。

```
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

这里你可能会疑问，threadLocals 是什么，ThreadLocalMap 是什么？
由于 ***t.threadLocals*** 所以我们知道 threadLocals 是线程 Thread 类的一个属性，这个属性是 ThreadLocalMap类型的。

```
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    private Entry[] table;
    ......
```
这里我们可以知道，每个线程都有一个 ThreadLocalMap 类型的属性 threadLocals，这个 Map 中的 entry 是一个弱引用类型，key 相当于是一个 ThreadLocal 变量，value 就是我们要存储的值。

所以，当我们 调用 threadLocal.set(new Integer(123)); 的时候，首先会得到 该线程 的 threadLocals，然后就会把 该 threadlocal 和 值123 作为一个键值对，放入到该线程的 threadLocals 中。

##### get()
那么，当我们 get 的时候，发生了什么：

```
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```
首先，我们会通过 getMap() 得到 上文 set() 部分 说的 threadLocals。

```
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
之前我们已经说了，threadLocals 保存了一个 key 是 threadlocal，value 是我们的值 的这样一个键值对。 那么，我们现在首先得到 该线程的 threadLocals，然后，当我们调用 threadlocal.get() 我们就可以根据这个 threadlocal 得到 我们的值，因为 之前我们 set 的时候，key 就是 这个 threadlocal。

就这样一个 get set，过程，就完成了 线程-值的关联。

##### 总结

1. 每个线程都有一个 ThreadLocalMap 类型的 threadLocals 属性。
2. ThreadLocalMap 类相当于一个Map，key 是 ThreadLocal 本身，value 就是我们的值。
3. 当我们通过 threadLocal.set(new Integer(123)); ，我们就会在这个线程中的 threadLocals 属性中放入一个键值对，key 是 这个 threadLocal.set(new Integer(123)); 的 threadlocal，value 就是值。
4. 当我们通过 threadlocal.get() 方法的时候，首先会根据这个线程得到这个线程的 threadLocals 属性，然后由于这个属性放的是键值对，我们就可以根据键 threadlocal 拿到值。 注意，这时候这个键 threadlocal 和 我们 set 方法的时候的那个键 threadlocal 是一样的，所以我们能够拿到相同的值。
这个就是 Threadlocal 的原理。

##### 备注

Q: TheadLocal 在一定程度上让代码更优雅，不用把token在各个方法传来传去。
假设，现在有个地方要起一个job，另起线程，需要token信息，那是把token显示的传到这个job里面去吗？还是有其他更优雅的解决方法？

A:可以使用 InheritableThreadLocal 。之前并不知道有 InheritableThreadLocal 这个东西，花了蛮多时间在想如何对ThreadLocal 进行封装，实在想不出参考了美团的对ThreadLocal的封装，就发现了这个东西。

另外，看了下 InheritableThreadLocal 的实现原理，如果有空麻请烦帮看看，非常非常感谢。另外，在看InheritableThreadLocal的过程中发现自己对 ThreadLocal 的原理理解也有偏差，这篇文章也改了，如果有空，麻烦看看有没有错误。