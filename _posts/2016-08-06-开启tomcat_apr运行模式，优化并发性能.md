---
layout: post
title: 开启Tomcat APR运行模式，优化并发性能
date:  2016-08-06 13:50:39
categories: java

---

#### 开启Tomcat APR运行模式，优化并发性能

Tomcat支持三种接收请求的处理方式：***BIO***、***NIO***、***APR*** 

1>、BIO模式：阻塞式I/O操作，表示Tomcat使用的是传统Java I/O操作(即java.io包及其子包)。Tomcat7以下版本默认情况下是以bio模式运行的，由于每个请求都要创建一个线程来处理，线程开销较大，不能处理高并发的场景，在三种模式中性能也最低。启动tomcat看到如下日志，表示使用的是BIO模式
![](http://img.blog.csdn.net/20160525232533550)

2>、NIO模式：是Java SE 1.4及后续版本提供的一种新的I/O操作方式(即java.nio包及其子包)。是一个基于缓冲区、并能提供非阻塞I/O操作的Java API，它拥有比传统I/O操作(bio)更好的并发运行性能。要让Tomcat以nio模式来运行比较简单，只需要在Tomcat安装目录/conf/server.xml文件中将如下配置：

```
<Connector port="8080" protocol="HTTP/1.1"
connectionTimeout="20000"
redirectPort="8443" />
```

修改成：

```
<Connector port="8080" protocol="org.apache.coyote.http11.Http11NioProtocol"
connectionTimeout="20000"
redirectPort="8443" />
```
注意：Tomcat8以上版本，默认使用的就是NIO模式，不需要额外修改 
![](http://img.blog.csdn.net/20160525232557988)
3>、apr模式：简单理解，就是从操作系统级别解决异步IO问题，大幅度的提高服务器的处理和响应性能， 也是Tomcat运行高并发应用的首选模式。 
启用这种模式稍微麻烦一些，需要安装一些依赖库，下面以在CentOS7 mini版环境下Tomcat-8.0.35为例，介绍安装步聚：

```
APR 1.2+ development headers (libapr1-dev package)
OpenSSL 0.9.7+ development headers (libssl-dev package)
JNI headers from Java compatible JDK 1.4+
GNU development environment (gcc, make)
```

###### 启用apr模式步聚

1、安装依赖库
因为apr模式本质是使用JNI技术调用操作系统IO接口，需要用到相关API的头文件

```
yum install apr-devel
yum install openssl-devel
yum install gcc
yum install make
```
注意：openssl库要求在0.9.7以上版本，APR要求在1.2以上版本，用rpm -qa | grep openssl检查本机安装的依赖库版本是否大于或等于apr要求的版本。

2、安装apr动态库

进入tomcat的bin目录，解压tomcat-native.tar.gz文件，并进入tomcat-native-1.2.7-src/native目录，执行./configure && make && make install 命令，动态库默认安装在/usr/local/apr/lib目录下，如下图所示： 

![](http://img.blog.csdn.net/20160525232037850)

3、配置APR本地库到系统共享库搜索路径中

方式1：

设置LD_LIBRARY_PATH和LD_RUN_PATH环境变量，指向/usr/local/apr/lib目录，可配置到$HOME/.profile文件中

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/apr/lib
export LD_RUN_PATH=$LD_RUN_PATH:/usr/local/apr/lib
```

方式2：

拷贝/usr/local/apr/lib目录下所有动态库到/usr/lib或/lib系统共享库搜索目录下即可。

```
copy /usr/local/apr/lib/libtcnative* /usr/lib/
```
方式3：（推荐）

编辑$TOMCAT_HOME/bin/catalina.sh文件，在虚拟机启动参数JAVA_OPTS中添加java.library.path参数，指定apr库的路径

```
JAVA_OPTS="$JAVA_OPTS -Djava.library.path=/usr/local/apr/lib"
```
Tomcat8以下版本，需要指定运行模式，将protocol从HTTP/1.1改成org.apache.coyote.http11.Http11AprProtocol

```
<Connector port="8080" protocol="org.apache.coyote.http11.Http11AprProtocol"
connectionTimeout="20000"
redirectPort="8443" />
```
运行Tomcat后，看到如下日志表示开启apr模式成功：

![](http://img.blog.csdn.net/20160525232913789)

如果没有配置SSL相关参数，并且开启了SSL，启动时会发生org.apache.tomcat.jni.Error: 70023: This function has not been implemented on this platform异常： 

![](http://img.blog.csdn.net/20160525233144543)

如果不想启用SSL，将server.xml中apr模式下ssl关闭即可：

```
<Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
```
将SSLEngine的值从on改成off即可：

```
<Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="off" />
```

```
Tomcat 6.x版本从6.0.32开始就默认支持apr。
Tomcat 7.x版本从7.0.30开始就默认支持apr。
因此，如果读者使用的Tomcat版本比较陈旧的话，强烈建议升级到最新的稳定版本。
```
