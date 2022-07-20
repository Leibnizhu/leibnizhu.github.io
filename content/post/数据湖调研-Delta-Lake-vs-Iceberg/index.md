---
title: "数据湖调研：Delta Lake vs Iceberg"
date: 2022-07-20T10:03:41+08:00
draft: true
---

## Delta Lake

Delta Lake是Databricks推出的流批一体存储层，分为开源版和商业版，深度绑定自家Spark。

### Lakehouse论文

请先快速阅读Databricks的论文 [Lakehouse: A New Generation of Open Platforms that Unify Data Warehousing and Advanced Analytics](https://databricks.com/wp-content/uploads/2020/12/cidr_lakehouse.pdf)。

这篇论文主要讲了传统的数仓和数据湖模型的缺陷，在此基础上提出湖仓一体的LakeHouse模型，并阐述了LakeHouse的一些设计细节。下面挑一些要点来讲讲。

LakeHouse是Databricks提出的一种将在未来几年取代传统数仓结构的架构。传统数仓将业务数据库数据收集到集中式的数仓，在写入时针对下游BI消费对schema进行优化；然而传统数仓将计算与存储耦合，且对非结构化数据支持不佳。于是出现了数据湖，以开放格式（如Parquet、ORC）存储在低成本的存储系统（如S3），湖中的数据ETL到下游数仓，再提供给BI使用。但这样的两层结构带来了一下问题：

- **可靠性**：难以维持数据湖与下游数仓的数据一致性
- **数据不及时**：需要两次ETL才到BI，对于传统数仓而言，甚至是开历史倒车
- **对高级分析支持不佳**：机器学习Tensorflow、PyTorch等，需要通过非SQL处理大量数据，无法通过JDBC/ODBC高效运行
- **总成本高**

于是噔噔瞪瞪，LakeHouse出来了：基于开放文件格式（如ORC/Parquet）的低成本存储，同时提供数仓的管理功能、传统DBMS的管理特性（如ACID事务、版本控制、审计、索引、缓存等）、及for高级分析（如机器学习）的高速I/O，的高性能系统。同时在设计上适应可存储计算隔离的云计算环境。LakeHouse解决了以下关键问题：

- **可靠的数据管理**：传统数据湖只是管理了一堆半结构化文件，在事务、回滚等方面的能力太差。
- **支持机器学习**：支持DataFrame API；而ML系统可以直接读取数据湖的格式，很多也支持DataFrame
- **SQL查询性能**：归功于过去对ORC/Parquet做SQL查询的性能优化经验

下面是这三代数仓/数据湖结构对比：

![](lakehouse1.png)

下面是LakeHouse架构和实现要点。

 **元数据层**：

LakeHouse设计理念是使用标准文件格式以低成本存储，在存储的顶层设计实现传统的元数据层，可提高抽象级别、实现ACID事务、版本控制等特性。DeltaLake _以Parquet格式在数据湖本身的存储上存放事务日志_ ，可以存储哪些对象属于表的信息：

- 实践证明这套设计在性能类似或更优于原始Parquet/ORC的同时，提供了事务等管理功能
- 元数据层同时可以保证数据质量，如强制schema（schema enforcement）可以保障新增数据必须匹配已有schema，并提供了API允许配置数据约束（如某列数据需要满足枚举要求）。
- 元数据层可实施ACL、日志审计等功能

**SQL性能优化**：

- 缓存：热数据缓存到更快的存储设备，如SSD、RAM
- 辅助数据：Lakehouse虽然使用了公开的文件格式（Parquet/ORC），但可以通过辅助数据帮助优化查询速度，如构建索引、维护统计信息。DeltaLake在上面提到的事务日志中维护列的最大最小值信息，方便查询时跳过；并实现了基于布龙过滤器的索引
- 数据布局优化：在Parquet格式不变的基础上，还是有优化空间，如记录排序，让数据聚拢便于读取。DeltaLake使用Z-order和Hilbert曲线对记录排序。

**针对高级分析的优化**：

通过声明式的DataFrame API加速高级分析。DataFrame API在DeltaLake的位置：

![](lakehouse2.png)

下图是Spark MLlib中执行声明式的DataFrame API。用户的DataFrame调用是lazy的，执行时Spark引擎获取执行计划，传给DeltaLake的库，后者读取元数据层，优化查询执行。

![](lakehouse3.png)

### Delta Lake白皮书

请先快速阅读Databricks的论文 [Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores](https://databricks.com/wp-content/uploads/2020/08/p975-armbrust.pdf) 。

**纯Parquet/ORC的缺陷**：

- 多对象更新不是原子的，所以查询之间没有隔离；回滚写操作也很困难：如果更新查询崩溃，则表处于损坏状态。
- 元数据操作的成本很高
- 大多数企业数据集是不断更新的，因此它们需要一个原子性的写入解决方案

Delta Lake应运而生，它是一个云对象存储上的ACID表存储层。核心思想：

- 用ACID方式在云对象中存储日志（write-ahead），以纪录哪些对象是Delta表的一部分
- 日志还包含元数据，例如每个数据文件的最小/最大统计信息，用于查询优化。
- 将所有元数据都存在底层对象存储中，并且使用针对对象存储的乐观并发协议实现事务，意味着不需要运行服务器来维护Delta表的状态

**特性**：

- Time travel
- UPSERT, DELETE and MERGE operations
- Efficient streamingI/O
- Caching
- Data layout optimization
- Schema evolution
- Audit logging

使用DeltaLake的例子：

![](deltalake1.png)

### 基本概念与使用

[官方文档](https://docs.delta.io/latest/delta-intro.html)
 
## Iceberg

[Data Science DC Nov 2021 Meetup: Apache Iceberg - An Architectural Look Under the Covers](https://www.youtube.com/watch?v=N4gAi_zpN88)

## Delta Lake vs Iceberg
