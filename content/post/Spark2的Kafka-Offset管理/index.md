---
title: Spark2的Kafka消费Offset管理
date: 2017-12-22T08:46:44+08:00
tags:
- Spark
- Scala
- Kafka
categories:
    - Spark
image: sunup.png
---
## 前言
网上流传一篇关于Spark Streaming消费Kafka时用Zookeeper保存Kafka队列offset的文章，如[https://www.2cto.com/net/201710/692443.html](https://www.2cto.com/net/201710/692443.html)，最初源头没找了，亲测在Spark1.6是可以用的。  
然而在Spark2中，这种方法的`KafkaManager`类中所依赖的`KafkaCluster`等等的类并不存在，因此此法无法直接套用到Spark中；此外，如果使用Cloudera的CDH集群的Spark2，其API更为缺少。因此，本文给出一种在CDH5.13的Spark2中通过Zookeeper管理Kafka消费Offset的方法。

## 环境
- 集群：`Cloudera CDH`（`Cloudera Manager` 5.13.0）
- `Spark`：2.1.0 cloudera2
- `Scala`：2.11.8
- `Java`：1.8.0_u91

## Maven依赖
```xml
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <scala.version>2.11.8</scala.version>
    <spark.version>2.1.0.cloudera2</spark.version>
    <kafka.version>0.11.0-kafka-3.0.0</kafka.version>
    <scala-test.version>3.0.0</scala-test.version>
</properties>

<repositories>
    <repository>
        <id>cloudera</id>
        <url>https://repository.cloudera.com/artifactory/cloudera-repos/</url>
    </repository>
    <repository>
        <id>aliyun</id>
        <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>org.scala-lang</groupId>
        <artifactId>scala-library</artifactId>
        <version>${scala.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.11</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_2.11</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming-kafka-0-10_2.11</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>${kafka.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-streams</artifactId>
        <version>${kafka.version}</version>
    </dependency>
</dependencies>
```

## 管理Kafka消费Offset
### 使用方法
#### 创建KafkaManager对象
使用类：
```scala
/**
  * Kafka的连接和Offset管理工具类
  *
  * @param zkHosts     Zookeeper地址
  * @param kafkaParams Kafka启动参数
  * @author Leibniz
  */
class KafkaManager(zkHosts: String, kafkaParams: Map[String, Object]) extends Serializable
```
如：
```scala
val zkHosts = "localhost:2181"
val kafkaParams = Map[String, Object](
    "auto.offset.reset" -> "latest",
    "bootstrap.servers" -> kafkaBrokers,
    "group.id" -> MAXWELL_KAFKA_GROUP,
    "enable.auto.commit" -> (false: java.lang.Boolean), //禁用自动提交Offset，否则可能没正常消费完就提交了，造成数据错误
    "key.deserializer" -> classOf[StringDeserializer],
    "value.deserializer" -> classOf[StringDeserializer])
val km = new KafkaManager(zkHosts, kafkaParams)
```
#### 创建Kafka输入流
```scala
/**
* 包装createDirectStream方法，支持Kafka Offset，用于创建Kafka Streaming流
*
* @param ssc         Spark Streaming Context
* @param topics      Kafka话题
* @tparam K Kafka消息Key类型
* @tparam V Kafka消息Value类型
* @return Kafka Streaming流
* @author Leibniz
*/
def createDirectStream[K: ClassTag, V: ClassTag](ssc: StreamingContext, topics: Seq[String]): InputDStream[ConsumerRecord[K, V]
```
如：
```scala
val kafkaStream = km.createDirectStream[String, String](ssc, kafkaTopics.split(",").toSeq)
```
#### 操作完毕后更新Offset
```scala
  /**
    * 保存Kafka消息队列消费的Offset
    *
    * @param rdd            SparkStreaming的Kafka RDD，RDD[ConsumerRecord[K, V]]
    * @param storeEndOffset true=保存结束offset， false=保存起始offset
    * @author Leibniz
    */
  def persistOffsets[K, V](rdd: RDD[ConsumerRecord[K, V]], storeEndOffset: Boolean = true): Unit
```
如：
```scala
km.persistOffsets[String, String](rdd)
```  
### 详细代码
```scala
package com.turingdi.enmonster.nrt.common

import java.lang.Object

import com.turingdi.enmonster.nrt.common.Constants._
import kafka.utils.{ZKGroupTopicDirs, ZkUtils}
import org.apache.kafka.clients.consumer.{ConsumerRecord, KafkaConsumer}
import org.apache.kafka.common.TopicPartition
import org.apache.spark.rdd.RDD
import org.apache.spark.streaming.StreamingContext
import org.apache.spark.streaming.dstream.InputDStream
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010.{ConsumerStrategies, HasOffsetRanges, KafkaUtils}
import org.slf4j.LoggerFactory

import scala.collection.JavaConversions._
import scala.reflect.ClassTag
import scala.util.Try

/**
  * Kafka的连接和Offset管理工具类
  *
  * @param zkHosts     Zookeeper地址
  * @param kafkaParams Kafka启动参数
  * @author Leibniz
  */
class KafkaManager(zkHosts: String, kafkaParams: Map[String, Object]) extends Serializable {
  //Logback日志对象，使用slf4j框架
  @transient private lazy val log = LoggerFactory.getLogger(getClass)
  //建立ZkUtils对象所需的参数
  val (zkClient, zkConnection) = ZkUtils.createZkClientAndConnection(zkHosts, ZK_SESSION_TIMEOUT, ZK_CONNECTION_TIMEOUT)
  //ZkUtils对象，用于访问Zookeeper
  val zkUtils = new ZkUtils(zkClient, zkConnection, false)

  /**
    * 包装createDirectStream方法，支持Kafka Offset，用于创建Kafka Streaming流
    *
    * @param ssc    Spark Streaming Context
    * @param topics Kafka话题
    * @tparam K Kafka消息Key类型
    * @tparam V Kafka消息Value类型
    * @return Kafka Streaming流
    * @author Leibniz
    */
  def createDirectStream[K: ClassTag, V: ClassTag](ssc: StreamingContext, topics: Seq[String]): InputDStream[ConsumerRecord[K, V]] = {
    val groupId = kafkaParams("group.id").toString
    val storedOffsets = readOffsets(topics, groupId)
    log.info("Kafka消息偏移量汇总(格式:(话题,分区号,偏移量)):{}", storedOffsets.map(off => (off._1.topic, off._1.partition(), off._2)))
    val kafkaStream = KafkaUtils.createDirectStream[K, V](ssc, PreferConsistent, ConsumerStrategies.Subscribe[K, V](topics, kafkaParams, storedOffsets))
    kafkaStream
  }

  /**
    * 从Zookeeper读取Kafka消息队列的Offset
    *
    * @param topics  Kafka话题
    * @param groupId Kafka Group ID
    * @return 返回一个Map[TopicPartition, Long]，记录每个话题每个Partition上的offset，如果还没消费，则offset为0
    * @author Leibniz
    */
  def readOffsets(topics: Seq[String], groupId: String): Map[TopicPartition, Long] = {
    val topicPartOffsetMap = collection.mutable.HashMap.empty[TopicPartition, Long]
    val partitionMap = zkUtils.getPartitionsForTopics(topics)

    // /consumers/<groupId>/offsets/<topic>/
    partitionMap.foreach(topicPartitions => {
      val zkGroupTopicDirs = new ZKGroupTopicDirs(groupId, topicPartitions._1)
      topicPartitions._2.foreach(partition => {
        val offsetPath = zkGroupTopicDirs.consumerOffsetDir + "/" + partition

        val tryGetKafkaOffset = Try {
          val offsetStatTuple = zkUtils.readData(offsetPath)
          if (offsetStatTuple != null) {
            log.info("查询Kafka消息偏移量详情: 话题:{}, 分区:{}, 偏移量:{}, ZK节点路径:{}", Seq[AnyRef](topicPartitions._1, partition.toString, offsetStatTuple._1, offsetPath): _*)
            topicPartOffsetMap.put(new TopicPartition(topicPartitions._1, Integer.valueOf(partition)), offsetStatTuple._1.toLong)
          }
        }
        if(tryGetKafkaOffset.isFailure){
          //http://kafka.apache.org/0110/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html
          val consumer = new KafkaConsumer[String, Object](kafkaParams)
          val partitionList = List(new TopicPartition(topicPartitions._1, partition))
          consumer.assign(partitionList)
          val minAvailableOffset = consumer.beginningOffsets(partitionList).values.head
          consumer.close()
          log.warn("查询Kafka消息偏移量详情: 没有上一次的ZK节点:{}, 话题:{}, 分区:{}, ZK节点路径:{}, 使用最小可用偏移量:{}", Seq[AnyRef](tryGetKafkaOffset.failed.get.getMessage, topicPartitions._1, partition.toString, offsetPath, minAvailableOffset): _*)
          topicPartOffsetMap.put(new TopicPartition(topicPartitions._1, Integer.valueOf(partition)), minAvailableOffset)
        }
      })
    })
    topicPartOffsetMap.toMap
  }

  /**
    * 保存Kafka消息队列消费的Offset
    *
    * @param rdd            SparkStreaming的Kafka RDD，RDD[ConsumerRecord[K, V]]
    * @param storeEndOffset true=保存结束offset， false=保存起始offset
    * @author Leibniz
    */
  def persistOffsets[K, V](rdd: RDD[ConsumerRecord[K, V]], storeEndOffset: Boolean = true): Unit = {
    val groupId = kafkaParams("group.id").toString
    val offsetsList = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
    offsetsList.foreach(or => {
      val zkGroupTopicDirs = new ZKGroupTopicDirs(groupId, or.topic)
      val offsetPath = zkGroupTopicDirs.consumerOffsetDir + "/" + or.partition
      val offsetVal = if (storeEndOffset) or.untilOffset else or.fromOffset
      zkUtils.updatePersistentPath(zkGroupTopicDirs.consumerOffsetDir + "/" + or.partition, offsetVal + "" /*, JavaConversions.bufferAsJavaList(acls)*/)
      log.debug("保存Kafka消息偏移量详情: 话题:{}, 分区:{}, 偏移量:{}, ZK节点路径:{}", Seq[AnyRef](or.topic, or.partition.toString, offsetVal.toString, offsetPath): _*)
    })
  }

}
```