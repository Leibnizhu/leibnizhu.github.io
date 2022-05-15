---
title: 解决Maxwell发送Kafka消息数据倾斜问题
date: 2018-01-03T17:34:30+08:00
tags:
- Kafka
- Maxwell
- 数据倾斜
- Spark
image: lotus.jpg
---
## 问题
最近用`Maxwell`解析`MySQL`的Binlog，发送到`Kafka`进行处理，测试的时候发现一个问题，就是`Kafka`的Offset严重倾斜，三个partition，其中一个的offset已经快200万了，另外两个offset才不到两百。  
Kafka数据倾斜的问题一般是由于生产者使用的`Partition`接口实现类对分区处理的问题，一般是对key做hash之后，对分区数取模。当出现数据倾斜时，小量任务耗时远高于其它任务，从而使得整体耗时过大，未能充分发挥分布式系统的并行计算优势（参考[Apache Kafka 0.10 技术内幕：数据倾斜详解](http://ningg.top/apache-kafka-10-best-practice-tips-data-skew-details/)）。  
而使用`Maxwell`解析`MySQL`的Binlog发送到`Kafka`的时候，生产者是`Maxwell`，那么数据倾斜的问题明细就是`Maxwell`引起的了。  

## 原因
在`Maxwell`官网查文档（[Producers:kafka-partitioning Maxwell's Daemon](http://maxwells-daemon.io/producers/#kafka-partitioning)）得知，在`Maxwell`没有配置的情况下，默认使用数据库名作为计算分区的key，并使用Java默认的hashcode算法进行计算：  
```
A binlog event's partition is determined by the selected hash function and hash string as follows
|  HASH_FUNCTION(HASH_STRING) % TOPIC.NUMBER_OF_PARTITIONS
The HASH_FUNCTION is either java's hashCode or murmurhash3. The default HASH_FUNCTION 
is hashCode. Murmurhash3 may be set with the kafka_partition_hash option. 
…………
The HASH_STRING may be (database, table, primary_key, column). The default HASH_STRING 
is the database. The partitioning field can be configured using the 
producer_partition_by option.
```
而在很多业务系统中，不同数据库的活跃度差异是很大的，主体业务的数据库操作频繁，产生的Binlog也就很多，而`Maxwell`默认使用数据库作为key进行hash，那么显而易见，Binglog的操作经常都被分到同一个分区里面了。  

## 解决
于是我们在`Maxwell`启动命令中加入对应参数即可，这里我选择了Rowkey作为分区key，同时选用murmurhash3
哈希算法，以获得更好的效率和分布：
```bash
nohup /opt/maxwell-1.11.0/bin/maxwell --user='maxwell' --password='***' --host='***' 
--exclude_dbs='/^(mysql|maxwell|test)/' --producer=kafka --kafka.bootstrap.servers=*** 
--kafka_partition_hash=murmur3 --producer_partition_by=primary_key >> /root/maxwell.log &

```

用此命令重新启动`Maxwell`之后，观察Offset的变化，隔一段时间之后，各分区Offset的增量基本一致，问题解决！