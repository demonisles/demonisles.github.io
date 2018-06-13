---
layout: post
title: "多进程提交spark app"
author: 王一刀
description: ""
category: 
tags: [spark springboot]
---

商户数据分析系统是通过web服务来管理spark app的。我们发现在web服务里同时提交多个spark app的时候，无法提交成功。在stackoverflow上查到：单个jvm里spark不允许有多个sparksession，因为spark不支持并且以后也不会支持(It is not supported and won't be)。原文：[Multiple SparkSessions in single JVM](https://stackoverflow.com/questions/40153728/multiple-sparksessions-in-single-jvm) 。web服务启动的时候只有一个进程，那么，问题来了，怎么才能在web服务里同时提交多个spark app呢？

有两种解决方案：一种用上面的回答里提到的第三方工具livy；一种是另起进程提交。

我试了一下livy，这个的确可以同时提交多个spark app，但是，有两个问题：
1. standalone模式下，提交后返回的application id 为null，无法跟踪任务状态。[Why is Apache Livy session showing Application id NULL?
](https://stackoverflow.com/questions/45489852/why-is-apache-livy-session-showing-application-id-null)
2. 任务提交后，在spark UI界面无法看到提交的任务

如此看来，第三方工具并不好用。只好用第二种方法：自己写程序另起进程提交。并且在livy的日志里我看到一句：

```bash
18/06/01 14:59:21 INFO SparkProcessBuilder: Running 
'/home/appadm/app/spark-2.0.2-bin-hadoop2.7/bin/spark-submit' 
'--deploy-mode' 'client' 
'--name' 'EarliestTradeApp' 
'--class' 'com.allinpal.mdas.sparkapps.EarliestTradeApp' 
'--conf' 'spark.executor.memory=1G' 
'--conf' 
'spark.jars=file:///app/thfd/commjars/mysql-connector-java-5.0.2.jar' 
'--conf' 'spark.submit.deployMode=client' 
'--conf' 'spark.master=spark://10.56.201.74:7077' 
'--conf' 'spark.driver.extraClassPath=/app/thfd/commjars/mysql-connector-java-5.0.2.jar' 
'file:///app/thfd/sparkapps/earliest-trade-1.0.0.jar' '201804' 'hdfs://cluster1' 
'jdbc:mysql://10.56.201.71:3306/mdas?useUnicode=true&characterEncoding=utf8' 
'mdas' 
'mdas' 
'10.56.200.191:9000|10.56.200.193:9000'
```

可见，livy的功能也是通过SparkProcessBuilder另起进程实现的。我们用另起进程的方法可能是可行的。

于是，我们需要一个可以独立运行的提交spark app的jar。这样的功能好写，但是打成可以执行的又包含第三方依赖包的jar，有点困难...我用了maven-assembly-plugin和maven-shade-plugin，他们的配置有点复杂，而且都没有达到我想要的效果。
最后用了springboot的CommandLineRunner，通过springboot打包成可以独立运行的jar包，配置简单，操作方便。

接下来就是在web服务里启进程提交spark app ，代码类似下面这样。

```java
		String[] arg0 = new String[] { "java", "-Xmx1024M", "-Xms512M", "-XX:MaxPermSize=256M",
				"-jar",
				"/app/thfd/apps/mdas-sparkappsubmit.jar",
				"spark://10.56.201.74:7077", "client", "EarliestTradeApp",
				"com.allinpal.mdas.sparkapps.EarliestTradeApp", "1G",
				"/app/thfd/commjars/mysql-connector-java-5.0.2.jar",
				"/app/thfd/sparkapps/earliest-trade-1.0.0.jar", "201802", "hdfs://cluster1",
				"jdbc:mysql://10.56.201.71:3306/mdas?useUnicode=true&characterEncoding=utf8", "mdas", "mdas",
				"10.56.200.191:9000|10.56.200.193:9000" };

		ProcessBuilder builder = new ProcessBuilder(arg0);
		try {
			Process process = builder.start();
			process.getInputStream();
			InputStreamReader isr = new InputStreamReader(process.getInputStream(), "UTF-8");
			BufferedReader br = new BufferedReader(isr);
			String line;
			System.out.printf("Output of running %s is:", Arrays.toString(arg0));
			while ((line = br.readLine()) != null) {
				System.out.println(line);
			}
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
```
如此，完美解决多进程提交spark app 的问题。:)

以上。