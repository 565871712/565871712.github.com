---
layout: post
title: Bean的生命周期
date:  2016-08-05 13:50:39
categories: java

---

### Spring Bean的生命周期（非常详细）

Spring作为当前Java最流行、最强大的轻量级框架，受到了程序员的热烈欢迎。准确的了解Spring Bean的生命周期是非常必要的。我们通常使用ApplicationContext作为Spring容器。这里，我们讲的也是 ApplicationContext中Bean的生命周期。而实际上BeanFactory也是差不多的，只不过处理器需要手动注册。

 转载请注明地址 http://www.cnblogs.com/zrtqsk/p/3735273.html，谢谢。
 
###  一、生命周期流程图：
![fas](http://images.cnitblog.com/i/580631/201405/181453414212066.png) 

![fas](http://images.cnitblog.com/i/580631/201405/181454040628981.png) 

若容器注册了以上各种接口，程序那么将会按照以上的流程进行。下面将仔细讲解各接口作用。

### 二、各种接口方法分类

Bean的完整生命周期经历了各种方法调用，这些方法可以划分为以下几类：

1. Bean自身的方法：这个包括了Bean本身调用的方法和通过配置文件中<bean>的init-method和destroy-method指定的方法
2. Bean级生命周期接口方法：这个包括了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这些接口的方法
3. 容器级生命周期接口方法：这个包括了InstantiationAwareBeanPostProcessor 和 BeanPostProcessor 这两个接口实现，一般称它们的实现类为“后处理器”。
4. 工厂后处理器接口方法：这个包括了AspectJWeavingEnabler, ConfigurationClassPostProcessor, CustomAutowireConfigurer等等非常有用的工厂后处理器　　接口的方法。工厂后处理器也是容器级的。在应用上下文装配配置文件之后立即调用。

### 三、演示
我们用一个简单的Spring Bean来演示一下Spring Bean的生命周期。

1. 首先是一个简单的Spring Bean，调用Bean自身的方法和Bean级生命周期接口方法，为了方便演示，它实现了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这4个接口，同时有2个方法，对应配置文件中<bean>的init-method和destroy-method。如下：

```
 1 package springBeanTest;
 2 
 3 import org.springframework.beans.BeansException;
 4 import org.springframework.beans.factory.BeanFactory;
 5 import org.springframework.beans.factory.BeanFactoryAware;
 6 import org.springframework.beans.factory.BeanNameAware;
 7 import org.springframework.beans.factory.DisposableBean;
 8 import org.springframework.beans.factory.InitializingBean;
 9 
10 /**
11  * @author qsk
12  */
13 public class Person implements BeanFactoryAware, BeanNameAware,
14         InitializingBean, DisposableBean {
15 
16     private String name;
17     private String address;
18     private int phone;
19 
20     private BeanFactory beanFactory;
21     private String beanName;
22 
23     public Person() {
24         System.out.println("【构造器】调用Person的构造器实例化");
25     }
26 
27     public String getName() {
28         return name;
29     }
30 
31     public void setName(String name) {
32         System.out.println("【注入属性】注入属性name");
33         this.name = name;
34     }
35 
36     public String getAddress() {
37         return address;
38     }
39 
40     public void setAddress(String address) {
41         System.out.println("【注入属性】注入属性address");
42         this.address = address;
43     }
44 
45     public int getPhone() {
46         return phone;
47     }
48 
49     public void setPhone(int phone) {
50         System.out.println("【注入属性】注入属性phone");
51         this.phone = phone;
52     }
53 
54     @Override
55     public String toString() {
56         return "Person [address=" + address + ", name=" + name + ", phone="
57                 + phone + "]";
58     }
59 
60     // 这是BeanFactoryAware接口方法
61     @Override
62     public void setBeanFactory(BeanFactory arg0) throws BeansException {
63         System.out
64                 .println("【BeanFactoryAware接口】调用BeanFactoryAware.setBeanFactory()");
65         this.beanFactory = arg0;
66     }
67 
68     // 这是BeanNameAware接口方法
69     @Override
70     public void setBeanName(String arg0) {
71         System.out.println("【BeanNameAware接口】调用BeanNameAware.setBeanName()");
72         this.beanName = arg0;
73     }
74 
75     // 这是InitializingBean接口方法
76     @Override
77     public void afterPropertiesSet() throws Exception {
78         System.out
79                 .println("【InitializingBean接口】调用InitializingBean.afterPropertiesSet()");
80     }
81 
82     // 这是DiposibleBean接口方法
83     @Override
84     public void destroy() throws Exception {
85         System.out.println("【DiposibleBean接口】调用DiposibleBean.destory()");
86     }
87 
88     // 通过<bean>的init-method属性指定的初始化方法
89     public void myInit() {
90         System.out.println("【init-method】调用<bean>的init-method属性指定的初始化方法");
91     }
92 
93     // 通过<bean>的destroy-method属性指定的初始化方法
94     public void myDestory() {
95         System.out.println("【destroy-method】调用<bean>的destroy-method属性指定的初始化方法");
96     }
97 }

```

2. 接下来是演示BeanPostProcessor接口的方法，如下：


```
 1 package springBeanTest;
 2 
 3 import org.springframework.beans.BeansException;
 4 import org.springframework.beans.factory.config.BeanPostProcessor;
 5 
 6 public class MyBeanPostProcessor implements BeanPostProcessor {
 7 
 8     public MyBeanPostProcessor() {
 9         super();
10         System.out.println("这是BeanPostProcessor实现类构造器！！");
11         // TODO Auto-generated constructor stub
12     }
13 
14     @Override
15     public Object postProcessAfterInitialization(Object arg0, String arg1)
16             throws BeansException {
17         System.out
18         .println("BeanPostProcessor接口方法postProcessAfterInitialization对属性进行更改！");
19         return arg0;
20     }
21 
22     @Override
23     public Object postProcessBeforeInitialization(Object arg0, String arg1)
24             throws BeansException {
25         System.out
26         .println("BeanPostProcessor接口方法postProcessBeforeInitialization对属性进行更改！");
27         return arg0;
28     }
29 }

```

 


