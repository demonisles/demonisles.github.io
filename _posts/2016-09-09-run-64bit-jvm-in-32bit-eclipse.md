---
layout: post
title: "在32位eclipse里运行64位jvm"
description: ""
category: exception
tags: []
---

mybatis里mapper太多时启动报错

		Cannot load JDBC driver class 'com.ibm.db2.jcc.DB2Driver'
		java.lang.StackOverflowError
		    at java.lang.ClassLoader.defineClass1(Native Method)
		    at java.lang.ClassLoader.defineClassCond(ClassLoader.java:631)
		    at java.lang.ClassLoader.defineClass(ClassLoader.java:615)
		    at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:141)
		    at org.apache.catalina.loader.WebappClassLoader.findClassInternal(WebappClassLoader.java:1850)
		    at org.apache.catalina.loader.WebappClassLoader.findClass(WebappClassLoader.java:890)
		    at org.apache.catalina.loader.WebappClassLoader.loadClass(WebappClassLoader.java:1354)
		    at org.apache.catalina.loader.WebappClassLoader.loadClass(WebappClassLoader.java:1233)
		    at java.lang.ClassLoader.defineClass1(Native Method)
		    at java.lang.ClassLoader.defineClassCond(ClassLoader.java:631)
		    at java.lang.ClassLoader.defineClass(ClassLoader.java:615)
		    at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:141)

网上也有人遇到，但是报的是stackoverflow

[Mybatis中Mapper文件过多，引起java.lang.StackOverflowError](http://www.dewen.net.cn/q/5558/Mybatis%E4%B8%ADMapper%E6%96%87%E4%BB%B6%E8%BF%87%E5%A4%9A%EF%BC%8C%E5%BC%95%E8%B5%B7java.lang.StackOverflowError)

上次做了两个处理

1.	修改jvm启动参数

	<img src="/images/2016-09-09/eclipse.jpg"  width="800"/>

2.	把commons-dbcp-1.2.2.jar更换为commons-dbcp-1.4.jar

但是今天启动又报错了。

考虑换成64位jvm，但是开发环境同一都是用的32位的eclipse，安装64位jvm后eclipse无法启动。于是想到在eclipse里运行64位jvm，外面环境变量还是用32的安装在C盘的jvm。
只是在eclipse做下面的修改,指向64位jvm

<img src="/images/2016-09-09/eclipse2.jpg"  width="800"/>

问题解决。

以上。