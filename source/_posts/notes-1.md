title: 大规模分布式系统_原理分析与架构实践

date: 2016-02-28 22:50:35

tags: 

---
## 1. 分布式存储定义
* 大量普通PC服务器通过internet互联，对外作为一个整体提供存储服务

## 2. 特性
* 扩展性，低成本，高性能，易用性

## 3. 挑战难点
+ 数据与状态信息的持久化
+ 在自动迁移与自读容错，并发读写的过程中保证数据的一致性

## 4. 数据类型分类
+ 非结构化数据
+ 结构化数据
+ 半结构化数据

## 5.分布式存储分类：

+ 分布式文件系统
+ 作为分布式表格系统与分布式数据库的的底层存储 
  - Facebook Haystack
  - TFS\GFS\Amazon EBS
+ 分布式键值系统
  - memcache\Tair\Amazon Dynamo
+ 分布式表格系统 
  - Google Bigtable
  - megastore
  - Microsoft azure table storage
  - Amazon DynamoDB
+ 分布式数据库
  - MySQL sharding
  - Amazon RDS
  - Microsoft SQL azure
  - Google spanner
  - OceanBase

## 6. OTHER

+ 性能瓶颈:  
  - 存储系统的性能瓶颈主要在于磁盘的随机读写
+ 优化方式：
  - 将随机写操作转化为顺序写
		>Log-structured Merge Tree 这套体系的核心思路就是，你的所有更新和删除操作，都会被转换为一个log并被追加到文件尾部，这样就可以将所有的原位更新都转换为顺序写入，代价则是更多的读取次数，不过因为NAND的读取不是瓶颈，所以代价是可以接受的.同时，后端有一组线程，从文件头开始顺序的读取所有的块，将块中未被标记为删除的数据找出来，凑够64个页后写到新的块中，然后调用块删除命令，清除那些已经完成誊写的数据块。

  - 通过缓存减少磁盘随机读操作
		>缓存


+ SSD随机读取友好，随机写入时存在写入放大问题


## 分布式系统衡量指标

+ **性能**
  - 吞吐能力与系统响应时间
+ 可用性
  - 指系统在面对各种异常时可以提供正常服务的能力，可用性可以用系统停服务的时间与正常服务的时间的比例来衡量
+ 一致性 
  - 越是强的一致性模型，用户使用起来越简单
+ 可扩展性


