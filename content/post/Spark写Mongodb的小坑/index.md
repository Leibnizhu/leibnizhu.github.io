---
title: Spark写Mongodb的小坑
date: 2020-01-15T15:47:27+08:00
tags:
- Spark
- MongoDB
categories:
    - Spark
image: zelda.jpg
---


# Spark写Mongodb的小坑
首先证明我还活着。  
因为Spark老集群版本限制(参见:[https://docs.mongodb.com/spark-connector/master/](https://docs.mongodb.com/spark-connector/master/))，`mongodb-connector`用的版本是`1.1.0`，以下坑基于该版本出现，新版本未验证。  
## 用MongoSpark.save写入RDD报错E11000 duplicate key error
观察源码
```scala
  def save[D: ClassTag](rdd: RDD[D], writeConfig: WriteConfig): Unit = {
    val mongoConnector = MongoConnector(writeConfig.asOptions)
    rdd.foreachPartition(iter => if (iter.nonEmpty) {
      mongoConnector.withCollectionDo(writeConfig, { collection: MongoCollection[D] =>
        iter.grouped(DefaultMaxBatchSize).foreach(batch => collection.insertMany(batch.toList.asJava))
      })
    })
  }
```
Mongo的逻辑是，纯RDD，没有schema，那么无法得知_id类型信息，于是直接insertMany。当Document数据里有_id字段时，insertMany可能就会发生_id冲突，报E11000 duplicate key error异常错误。  
解决方案：  
1. 改用 `MongoSpark.save(dataFrame: DataFrame)`;
2. 爆改`MongoSpark`, 将`MongoSpark.save(rdd: RDD[D])`改成根据Docuemnt是否有_id而分别生成`ReplaceOneModel`和`InsertOneModel`，再批量插入。

我一开始用了方案1，后来还是觉得限制太多，但也没有直接爆改`MongoSpark`，而是直接在代码里实现，不用`MongoSpark`了。  

## List含null时写入失败
原因是`com.mongodb.spark.sql.MapFunctions`的`arrayTypeToBsonValue`方法没有处理List里面的空元素，而是直接每个元素调`convertToBsonValue`方法，后者再调用`elementTypeToBsonValue`的时候，在模式匹配里面就匹配不上，进入最后的默认分支，抛`MongoTypeConversionException`异常。   
解决方案很简单：  
1. 对数组/List做预处理，过滤null元素，但可能这些null元素是有用的，此时此法无用；
2. 爆改`MapFunctions`：

其实也不算爆改，小改几个地方即可：
```scala
  private def arrayTypeToBsonValue(elementType: DataType, containsNull: Boolean, data: Seq[Any]): BsonValue = {
    val internalData = elementType match {
      /*....省略无关代码....*/
      case _                        => data.map(x => if(x == null && containsNull) new BsonNull() else convertToBsonValue(x, elementType)).asJava
    }
    new BsonArray(internalData)
  }

  private def elementTypeToBsonValue(element: Any, elementType: DataType): BsonValue = {
    elementType match {
      /*....省略无关代码....*/
      case arrayType: ArrayType => arrayTypeToBsonValue(arrayType.elementType, arrayType.containsNull, element.asInstanceOf[Seq[_]])
      /*....省略无关代码....*/
    }
  }

  private def mapTypeToBsonValue(valueType: DataType, data: Map[String, Any]): BsonValue = {
    val internalData = valueType match {
      /*....省略无关代码....*/
      case subArray: ArrayType      => data.map(kv => new BsonElement(kv._1, arrayTypeToBsonValue(subArray.elementType, subArray.containsNull, kv._2.asInstanceOf[Seq[Any]])))
      /*....省略无关代码....*/
    }
    new BsonDocument(internalData.toList.asJava)
  }
```

## ObjectId对象写入
Mongodb的_id默认是用ObjectId类型，如果在修改数据后重新写入Mongodb，也需要使用同样的ObjectId对象。  
在直接使用Mongodb的SDK的情况，这个很简单，直接new一个`org.bson.types.ObjectId`对象即可。  
如果是使用DataFrame，则structType和具体的Row数据构造都有一小点麻烦，直接上代码吧。  
构建Schema：  
```scala
var structFieldList = List()
/*....其他字段schema信息....*/
val idDataType = StructType(List(StructField("oid", StringType, true, Metadata.empty)))
structFieldList += StructField("_id", idDataType, true, Metadata.empty))
StructType(structFieldList);
```
写入数据的处理：
```scala
GenericRow(Array("5e1db87e5d080f6e7eb7b067")) //这是一个Row里面的一个字段值
```