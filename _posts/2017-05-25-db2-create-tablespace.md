---
layout: post
title: "db2 创建表空间"
author: 王一刀
description: ""
category: database
tags: [db2 tablespace]
---

网上可以查到很多创建表空间的文章，但是都是从头开始，创建数据库，创建缓冲池，创建表空间。但是没有在已有数据和缓冲池的情况下，创建表空间的例子。今天做了一次这样的操作，记录一下：

1. 查看已有表空间

``` shell
    db2pd -d tlb1405 -tablespaces
```

 [<img src="/images/2017-05-25/db2pd.jpg"  width="400"/>](/images/2017-05-25/db2pd.jpg)

2. 查看表空间使用缓冲池

``` shell
    SELECT tp.TBSPACE, tp.BUFFERPOOLID,bp.BPNAME FROM SYSCAT.TABLESPACES tp left join  SYSCAT.BUFFERPOOLS bp on tp.BUFFERPOOLID=bp.BUFFERPOOLID
```

 <img src="/images/2017-05-25/bufferpool.jpg"  width="400"/>

可以看到tlb_data_8k使用的是名为**DAT01**的缓冲池，我们创建新空间的时候也用这个

3. 创建表空间

``` shell
    db2 "create regular tablespace  loan_data_8k pagesize 8k managed by database using(file '/home/db2inst1/db2inst1/NODE0000/TLB1405/loan_data_8k' 1g) bufferpool DAT01"  

    db2 "create regular tablespace  loan_index_8k pagesize 8k managed by database using(file '/home/db2inst1/db2inst1/NODE0000/TLB1405/loan_index_8k' 1g) bufferpool DAT01"  
```

结束。