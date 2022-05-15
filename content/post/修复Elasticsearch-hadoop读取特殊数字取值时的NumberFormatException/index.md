---
title: 修复Elasticsearch-hadoop读取特殊数字取值时的NumberFormatException
date: 2021-06-19T23:40:50+08:00
tags:
- Elasticsearch
- ES
- Spark
- 大数据
categories:
    - Spark
image: bg2.jpg
---
## 故障背景
众所周知，`Elasticsearch`（下文简称`"ES"`）的数值类型字段是支持一些特殊格式的。比如，`integer` 类型的字段取值可以是浮点数的字符串，如 `"2.0"`；`long` 类型的字段取值可以是科学计数法的字符串，如 `"2E3"`，诸如此类的一些。不同于时间类型的字段可以 [通过 `mapping` 的 `format` 属性配置取值格式](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html) ，数值字段的取值没有格式的配置、上面举例的取值都是可以直接索引文档的。

然而，通过 [ES-Hadoop](https://www.elastic.co/guide/en/elasticsearch/hadoop/current/spark.html) （亦可参考 [Github](https://github.com/elastic/elasticsearch-hadoop) ） 查询ES时，这些特殊格式的取值往往会导致报错，如：
```shell
java.lang.NumberFormatException: For input string: '2E3'
    at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
    at java.lang.Long.parseLong(Long.java:441)
    at java.lang.Long.parseLong(Long.java:483)
    at org.elasticsearch.hadoop.serialization.builder.JdkValueReader.parseLong(JdkValueReader.java:296)
    at org.elasticsearch.hadoop.serialization.builder.JdkValueReader.longValue(JdkValueReader.java:288)
…………
```

## 故障根源分析
阅读 `JdkValueReader` 源码，以读取integer类型字段为例：
```java
protected Object intValue(String value, Parser parser) {
    Integer val = null;

    if (value == null || isEmpty(value)) {
        return nullValue();
    }
    else {
        Token tk = parser.currentToken();

        if (tk == Token.VALUE_NUMBER) {
            val = parser.intValue();
        }
        else {
            val = parseInteger(value);
        }
    }

    return processInteger(val);
}

protected Integer parseInteger(String value) {
    return Integer.parseInt(value);
}
```

可以看到字段取值直接调用 `Integer.parseInt` 方法解析，且没捕获异常。  
不知道这样设计是处于什么考虑，但这个 `NumberFormatException` 异常会打断读取的流程：出现一条有问题的数据时，会影响整个查询任务的执行，在 Es-spark 使用于离线批处理的场景，是不恰当的，所以有必要进行调整。

## 解决方案
### 自定义 ValueReader
进一步阅读 ES-spark 源码可以发现，用户可以自己实现 `org.elasticsearch.hadoop.serialization.builder.ValueReader` 接口，并通过 `es.ser.reader.value.class` 配置项（常量`org.elasticsearch.hadoop.cfg.ConfigurationOptions.ES_SERIALIZATION_READER_VALUE_CLASS`）配置自定义的 `ValueReader` 实现，从而可以实现对这些特殊取值的读取解析。 当然，后来在 [官方文档](https://www.elastic.co/guide/en/elasticsearch/hadoop/current/configuration.html) 中也印证了这一点。

这样实际处理下来，基本是要拷贝 `JdkValueReader` 源码进行修改作为自定义的 `ValueReader` 实现；显然，这样就不能随 ES-spark 升级而自动升级对应实现，同时，在代码中，自定义的修改也和原 `JdkValueReader` 的实现混杂在一起，给升级带来困难；因此考虑使用 `Javassist` 进行patch。

### Javassist Patch
`Javassist` 入门和介绍的文章在网上已经很多了，在此不再赘述。  
列举一下patch过程中遇到的一些坑，或者说，`Javassist` 的一些使用注意事项：

1. 不支持泛型，请自行强转；
2. 类要用全限定类名，没有import；
3. $0=this, $1/$2/$3=方法的第1/2/3个参数；
4. 代码块前后要用{}包裹；
5. 不支持增强for、lambda等高级语法，需要手动转成基本语法。

最后给出针对Elasticsearch-hadoop读取特殊数字取值的 `Javassist` patch代码：
```java
ClassPool pool = ClassPool.getDefault();

try {
    //这里必须用类全限定名，而不是JdkValueReader.class.getName(),否则会先加载类，后面的修改就没用了
    CtClass cc = pool.get("org.elasticsearch.hadoop.serialization.builder.JdkValueReader");
        
    //修复 parseInteger 方法 
    CtMethod parseInteger = cc.getDeclaredMethod("parseInteger");
    CtClass exceptionClass = pool.get(Exception.class.getName());
    String catchParseIntegerException = "try{return new java.lang.Integer(java.lang.Double.valueOf($1).intValue());}catch(java.lang.Exception e){e.printStackTrace();return null;}";
    parseInteger.addCatch("{" + catchParseIntegerException + "}", exceptionClass);

    //修复 parseLong 方法 
    CtMethod parseLong = cc.getDeclaredMethod("parseLong");
    String catchParseLongExpSrc = "try{return new java.lang.Long(java.lang.Double.valueOf($1).longValue());}catch(java.lang.Exception e){e.printStackTrace();return null;}";
    parseLong.addCatch("{" + catchParseLongExpSrc + "}", exceptionClass);
    
    cc.toClass().newInstance();
    log.info("完成对 JdkValueReader 进行 parseInteger() 和 parseLong() 的pack");
} catch (Exception e) {
    log.error("给 JdkValueReader 进行 parseInteger() 和 parseLong() 的pack失败:" + e.getMessage(), e);
}
```

在此基础上还可以做成按配置动态patch等骚操作。最后编译运行，Pass。