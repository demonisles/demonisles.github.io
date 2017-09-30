---
layout: post
title: "一个hprof文件的分析过程"
description: ""
category: 
tags: []
---
9月30日发现jboss bin目录下在9月15日生成了hprof文件：java_pid2827.hprof


<img src="/images/2017-09-30/hprof1.jpg"  width="400"/>

将hprof文件下载到本地，使用mat分析工具分析问题产生原因。

工具：

eclipse Version: Mars.2 Release (4.5.2)
mat插件：1.6.1

<img src="/images/2017-09-30/mat.png"  width="400"/>

因为hprof是2G，需要在eclipse.ini 中将eclipse启动内存改为3G
<img src="/images/2017-09-30/ini.png"  width="400"/>

将hprof文件导入mat：



<img src="/images/2017-09-30/fix1.png"  width="400"/>

如图所示：仅 http-0.0.0.0-8080-14 一个线程就占用1.6G内存

查看内存泄漏嫌疑报告


<img src="/images/2017-09-30/fix2.png"  width="400"/>
<img src="/images/2017-09-30/fix3.png"  width="400"/>

查看堆栈信息 基本可以定位
at com.tonghuafund.loan.webservice.impl.WeixinLoanMsgPushWebServiceImpl.pushWeixinLoanMessage()V (WeixinLoanMsgPushWebServiceImpl.java:126)

调用com.allinpay.fmp.loan.business.service.impl.ApmsMchtServiceImpl.getApmsMchts 查询时导致内存溢出。

接下来是查看代码进一步定位问题。

以上。
