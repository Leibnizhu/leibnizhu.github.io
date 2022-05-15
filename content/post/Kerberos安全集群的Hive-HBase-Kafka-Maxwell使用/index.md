---
title: 'Kerberos集群的Sqoop,Hive,HBase,Kafka,Maxwell使用'
date: 2018-03-07T16:23:56+08:00
tags:
- Kerberos
- Sqoop
- Hive
- HBase
- Kafka
- Maxwell
- 大数据
- Hadoop
image: flower.jpg
---
介绍在部署了Kerberos的安全Hadoop集群中, Sqoop,Hive,HBase,Kafka,Maxwell使用方法.  
## Sqoop使用
配置好Kerberos之后, sqoop不能直接使用, 需要进行一些配置:  
1. 分配sqoop的组, 执行`usermod -a -G hdfs sqoop`加入到hdfs组, 使用`groups sqoop`确认执行成功; 
2. 进入Hue的用户管理界面, 新增sqoop用户, 在hdfs用户组中;
3. 在Hue的HDFS文件管理页面中, 创建/user/sqoop目录, 从属于sqoop:hdfs用户/用户组;
4. 进入cdh1, 创建Kerberos用户, 名为sqoop, 可以导出keytab;
5. 使用kinit命令切换到sqoop用户(在脚本中可以使用keytab切换)
6. 执行sqoop命令


## Spark访问HBase
1. 进入cdh1, 创建Kerberos用户, 名为hbase; 导出keytab, 名为hbase.keytab, 保存到本地;  
2. 下载krb5.conf到本地.  
3. 创建测试类, 并执行, 代码如下:
```scala
/*HBase测试*/
object KerberosHBaseTest {
  def main(args: Array[String]) {
    val zkHosts = "cdh2:2181,cdh3:2181,cdh4:2181"
    System.setProperty("java.security.krb5.conf", "/path/to/krb5.conf") //krb5.conf本地路径
    val sparkConf = new SparkConf().setAppName("KerberosHBaseTest").setMaster("local")
    val sc = new SparkContext(sparkConf)
    //配置HBase连接
    val hbaseConfig = HBaseConfiguration.create()
    hbaseConfig.set("hbase.zookeeper.quorum", zkHosts)
    hbaseConfig.set("zookeeper.znode.parent", "/hbase")
    //设置集群和hbase的安全模式为kerberos
    hbaseConfig.set("hadoop.security.authentication", "kerberos")
    hbaseConfig.set("hbase.security.authentication", "kerberos")
    hbaseConfig.set("hbase.master.kerberos.principal", "hbase/_HOST@TURINGDI.COM") //没有似乎也行
    hbaseConfig.set("hbase.regionserver.kerberos.principal", "hbase/_HOST@TURINGDI.COM") //必须有
    UserGroupInformation.setConfiguration(hbaseConfig)
    UserGroupInformation.loginUserFromKeytab("hbase", "/path/to/hbase.keytab") //Kerberos用户名, keytab本地路径
    //设置广播变量，解决序列化问题
    //HBase配置
    val broadcastHBaseConf = sc.broadcast(new SerializableWritable(hbaseConfig))
    //HBase连接工具类
    val hbaseConnection = sc.broadcast(HBaseConnection(broadcastHBaseConf))
    val result = scanByStartTimestamp(hbaseConnection, "t1", 0L)
    result.foreach(r => println(ConvertService.convertResultToHBaseRow(r)))
    sc.stop()
  }

  /**
    * 从HBase中scan指定表的所有列，从指定的时间戳开始
    *
    * @param hBaseConnection HBase连接
    * @param tableName       表名
    * @param starTimestamp   开始scan的时间戳，从该时间戳scan到当前时间
    * @return scan的结果，Result的List
    * @author Leibniz
    */
  def scanByStartTimestamp(hBaseConnection: Broadcast[HBaseConnection], tableName: String, starTimestamp: Long): ArrayBuffer[Result] = {
    val resultList = new ArrayBuffer[Result]()
    Try({
      val scan = new Scan()
      scan.setTimeRange(starTimestamp, System.currentTimeMillis)
      // 获取表
      val table = hBaseConnection.value.connection.getTable(TableName.valueOf(tableName))
      table.getScanner(scan).foreach(resultList.+=)
    }).recover({
      case e: Throwable => log.error("从HBase表{}中按时间戳({}->NOW)scan时抛出异常:{}", Seq[AnyRef](tableName, starTimestamp.toString, e.getMessage): _*)
    })
    resultList
  }
}
```

## Spark访问Hive
1. Hive可以沿用hbase的Kerberos用户, 也可以新建一个Hive用户及其对应keytab文件.  
2. 本地测试请将集群的`hive-site.xml`导出并保存在项目的`src/main/resources/`目录下;
3. 编写Spark测试程序:  
```scala
/*Hive测试*/
object KerberosHiveTest {
  def main(args: Array[String]) {
    System.setProperty("java.security.krb5.conf", "/path/to/krb5.conf") //krb5.conf本地路径
    val sparkConf = new SparkConf().setAppName("KerberosHiveTest").setMaster("local")
    val sc = new SparkContext(sparkConf)
    val config = HBaseConfiguration.create()
    config.set("hadoop.security.authentication", "kerberos") //必须有
    UserGroupInformation.setConfiguration(config)
    UserGroupInformation.loginUserFromKeytab("hbase", "/path/to/hbase.keytab") //Kerberos用户名, keytab本地路径
    val sparkSession = SparkSession.builder.master("local").appName("KerberosHiveTest").enableHiveSupport()
    .config("yarn.resourcemanager.principal", "rm/_HOST@TURINGDI.COM") //必须有
    //      .config("spark.yarn.keytab", "/path/to/hbase.keytab")
    //      .config("spark.yarn.principal", "hbase@TURINGDI.COM")
    .getOrCreate()
    val dataFrame = sparkSession.sql("select * from hivetest2")
    dataFrame.rdd.foreach(row => println(row.getInt(0) + " -> " + row.getString(1)))
    sc.stop()
  }
}
```

## Spark访问Kafka
1. 进入Cloudera Manager的Kafka配置页面, 搜索'Inter Broker Protocol', 更改为'SASL_PLAINTEXT';
2. 重启Kafka配置;
3. 进入cdh1, 创建Kerberos用户, 名为kafka; 导出keytab, 名为kafka.keytab, 并保存到本地(测试用);
4. cdh1中新建一个jaas.conf配置文件, 并复制到本地(注意修改keyTab), 内容如下:
```bash
KafkaClient {
  com.sun.security.auth.module.Krb5LoginModule required
  doNotPrompt=true
  useTicketCache=true
  useKeyTab=true
  principal="kafka@TURINGDI.COM" #根据实际修改
  serviceName="kafka" ## 固定
  client=true
  keyTab="/path/to/kafka.keytab"; ## keytab路径,节点和本地按实际路径填写
};
```
5. cdh1中新建一个kafka.properties文件, 内容如下: 
```conf
security.protocol=SASL_PLAINTEXT
sasl.kerberos.service.name=kafka
sasl.mechanism=GSSAPI
security.inter.broker.protocol=SASL_PLAINTEXT
```
6. 编写Spark程序进行测试:  
```scala
object KerberosKafkaTest {
  def main(args: Array[String]) {
    val zkHosts = "cdh2:2181,cdh3:2181,cdh4:2181"
    val kafkaBrokers = "cdh2:9092,cdh3:9092,cdh4:9092"
    val topics = List("maxwell")
    System.setProperty("java.security.krb5.conf", "/path/to/krb5.conf") //本地krb5.conf路径
    System.setProperty("java.security.auth.login.config", "/path/to/jaas.conf")//本地jaas.conf路径
    // 创建流处理上下文，并以启动参数指定的秒数为时间间隔做一次批处理。
    val sparkConf = new SparkConf().setAppName("KerberosKafkaTest").set("spark.streaming.kafka.consumer.poll.ms", KAFKA_CONSUMER_POLL_MS).setMaster("local")
    val ssc = new StreamingContext(sparkConf, Seconds(10))
    // 配置并创建Kafka输入流
    // 如果zookeeper没有offset值或offset值超出范围，就给个初始的offset
    // 有earliest、largest可选，分别表示给当前最小的offset、当前最大的offset。默认largest
    val kafkaParams = Map[String, Object](
      "auto.offset.reset" -> "earliest",
      "bootstrap.servers" -> kafkaBrokers,
      "group.id" -> "testGroup",
      "enable.auto.commit" -> (false: java.lang.Boolean), //禁用自动提交Offset，否则可能没正常消费完就提交了，造成数据错误
      "key.deserializer" -> classOf[StringDeserializer],
      "value.deserializer" -> classOf[StringDeserializer],
      "sasl.kerberos.service.name" -> "kafka", //必须有   
      "security.protocol" -> "SASL_PLAINTEXT") //与Kafka配置一致
    val kafkaStream = KafkaUtils.createDirectStream(ssc, PreferConsistent, ConsumerStrategies.Subscribe(topics, kafkaParams))
    kafkaStream.foreachRDD(rdd => {
      log.info("接收到{}条Kafka消息", rdd.count)
      rdd.foreach(message => {
          println("partition=" + message.partition + ", value=" + message.value + ", offset=" + message.offset.toString)
        })
    })
    ssc.start()
    ssc.awaitTermination()
  }
}
```
7. kafka自带的命令, 如kafka-console-consumer, kafka-topics还不能使用, 若要使用, 需要先执行:
```bash
export KAFKA_OPTS="-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/path/to/jaas.conf"
```
注意修改其中的jass.conf路径, 并确保其中配置的keytab存在; 再执行相应的kafka命令.  
如果觉得麻烦, 也可以编辑`/opt/cloudera/parcels/KAFKA-3.0.0-1.3.0.0.p0.40/lib/kafka/bin/kafka-run-class.sh`, 在`exec $JAVA`后面增加:  
```bash
-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/root/jaas.conf
```

## Maxwell配置
1. 编辑${MAXWELL_HOME}/bin/maxwell, 在文件尾部附件的`exec $JAVA $JAVA_OPTS`后面增加:  
```bash
-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/root/jaas.conf
```
2. 编辑一个config.properties文件, 内容如下:  
```conf
kafka.security.protocol=SASL_PLAINTEXT
kafka.sasl.kerberos.service.name=kafka
kafka.sasl.mechanism=GSSAPI
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=PLAIN
```
3. 在maxwell启动命令中增加参数:  
```bash
--config /path/to/config.properties
```