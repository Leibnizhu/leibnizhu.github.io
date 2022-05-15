---
title: 跨Yarn集群提交spark任务
date: 2021-12-04 18:17:55
tags:
- Spark
- Hadoop
- Yarn
- 跨集群
categories:
    - Spark
image: 93990522.jpg
---
# 背景
之前写过一篇 [Spark动态加载hive配置的方案](/2020/05/06/动态加载hive配置文件的方案/) ，当时是为了spark应用的fat-jar里面已经有Hadoop相关xml配置文件的情况下，将数据输出到不是该配置的Hadoop集群的方案。  
现在这个需求有点类似，没有走spark-submit提交任务，而是在spark应用里面通过创建`SparkContext`的形式提交任务，而spark应用的fat-jar里面已经有Hadoop相关xml配置文件，在此情况下，想将Spakr任务提交到外部的Yarn集群（不是fat-jar里面配置文件对应的yarn集群）。  

# 思考一个问题
先思考一个问题，如果Spark应用的fat-jar里面有外部Yarn集群对应的配置文件(`core-site.xml`，`hdfs-site.xml`，`yarn-site.xml`等)，此时Spark应用代码里面创建`SparkContext`，是不是就一定能提交到那个集群里？  
可以做个实验，但实验不一定会cover到所有情况。  
直接给结论吧，不一定能提交过去，但自己做实验的话很可能还是能直接提交过去的，还是直接看代码吧（以`yarn-client`模式为例）。  

## Spark Yarn-client 默认提交任务简析
通过代码创建`SparkContext`后，其动态代码块会根据启动模式创建`SchedulerBackend`和`TaskScheduler`并启动：  
```scala
// org.apache.spark.SparkContext #501
    // Create and start the scheduler
    val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
    _schedulerBackend = sched
    _taskScheduler = ts
    _dagScheduler = new DAGScheduler(this)
    // start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler's
    // constructor
    _taskScheduler.start()
```
其中 `TaskScheduler` 是通过 `org.apache.spark.scheduler.cluster.YarnClusterManager#createTaskScheduler` 创建的，对应 yarn-client 创建的是`YarnScheduler`（继承了`TaskSchedulerImpl`），start()方法调用到`SchedulerBackend`的`start`方法，后者就会创建yarn模式下的Client客户端（`org.apache.spark.deploy.yarn.Client`，不是yarn自己那个client），并调用其`submitApplication`方法提交任务到Yarn：  
```scala
//org.apache.spark.scheduler.cluster.YarnClientSchedulerBackend#start
  override def start() {
    val driverHost = conf.get("spark.driver.host")
    val driverPort = conf.get("spark.driver.port")
    val hostport = driverHost + ":" + driverPort
    sc.ui.foreach { ui => conf.set("spark.driver.appUIAddress", ui.webUrl) }

    val argsArrayBuf = new ArrayBuffer[String]()
    argsArrayBuf += ("--arg", hostport)

    logDebug("ClientArguments called with: " + argsArrayBuf.mkString(" "))
    val args = new ClientArguments(argsArrayBuf.toArray)
    totalExpectedExecutors = YarnSparkHadoopUtil.getInitialTargetExecutorNumber(conf)
    client = new Client(args, conf)
    bindToYarn(client.submitApplication(), None)
    //………………
}
```
初始化`Client`的时候，会创建YarnConfiguration，此时就会读取到Configuration里面配置的默认资源，包括`yarn-site.xml`等；如果fatjar里面放的是外部集群的配置文件，那么对应的`YarnClient`就可以连接到外部Yarn集群。  
```scala
//org.apache.spark.deploy.yarn.Client
private[spark] class Client(
    val args: ClientArguments,
    val hadoopConf: Configuration,
    val sparkConf: SparkConf)
  extends Logging {
    //………………
  private val yarnClient = YarnClient.createYarnClient
  private val yarnConf = new YarnConfiguration(hadoopConf)
```
接着刚才说到`SchedulerBackend`调用`Client`的`submitApplication`方法:  
```scala
//org.apache.spark.deploy.yarn.Client#submitApplication
  def submitApplication(): ApplicationId = {
    var appId: ApplicationId = null
    try {
      launcherBackend.connect()
      // Setup the credentials before doing anything else,
      // so we have don't have issues at any point.
      setupCredentials()
      yarnClient.init(yarnConf)
      yarnClient.start()

      logInfo("Requesting a new application from cluster with %d NodeManagers"
        .format(yarnClient.getYarnClusterMetrics.getNumNodeManagers))

      // Get a new application from our RM
      //新建一个Application
      val newApp = yarnClient.createApplication()
      val newAppResponse = newApp.getNewApplicationResponse()
      appId = newAppResponse.getApplicationId()

      new CallerContext("CLIENT", sparkConf.get(APP_CALLER_CONTEXT),
        Option(appId.toString)).setCurrentContext()

      // Verify whether the cluster has enough resources for our AM
      verifyClusterResources(newAppResponse)

      // Set up the appropriate contexts to launch our AM
      //创建environment, java options以及启动AM的命令
      val containerContext = createContainerLaunchContext(newAppResponse)
      //创建提交AM的Context，包括名字、队列、类型、内存、CPU及参数
      val appContext = createApplicationSubmissionContext(newApp, containerContext)

      // Finally, submit and monitor the application
      logInfo(s"Submitting application $appId to ResourceManager")
      //向Yarn提交Application
      yarnClient.submitApplication(appContext)
      launcherBackend.setAppId(appId.toString)
      reportLauncherState(SparkAppHandle.State.SUBMITTED)

      appId
    } catch {
      case e: Throwable =>
        if (appId != null) {
          cleanupStagingDir(appId)
        }
        throw e
    }
  }
```
其中 createContainerLaunchContext 会创建environment, java options以及启动AM的命令等，也会收集本地资源（`prepareLocalResources`方法），其中包括`__spark_conf__.zip`，在`createConfArchive`方法中处理，压缩了本地的一些配置文件：  
```scala
  private def createConfArchive(): File = {
    val hadoopConfFiles = new HashMap[String, File]()

    // Uploading $SPARK_CONF_DIR/log4j.properties file to the distributed cache to make sure that
    // the executors will use the latest configurations instead of the default values. This is
    // required when user changes log4j.properties directly to set the log configurations. If
    // configuration file is provided through --files then executors will be taking configurations
    // from --files instead of $SPARK_CONF_DIR/log4j.properties.

    // Also uploading metrics.properties to distributed cache if exists in classpath.
    // If user specify this file using --files then executors will use the one
    // from --files instead.
    for { prop <- Seq("log4j.properties", "metrics.properties")
          url <- Option(Utils.getContextOrSparkClassLoader.getResource(prop))
          if url.getProtocol == "file" } {
      hadoopConfFiles(prop) = new File(url.getPath)
    }

    Seq("HADOOP_CONF_DIR", "YARN_CONF_DIR").foreach { envKey =>
      sys.env.get(envKey).foreach { path =>
        val dir = new File(path)
        if (dir.isDirectory()) {
          val files = dir.listFiles()
          if (files == null) {
            logWarning("Failed to list files under directory " + dir)
          } else {
            files.foreach { file =>
              if (file.isFile && !hadoopConfFiles.contains(file.getName())) {
                hadoopConfFiles(file.getName()) = file
              }
            }
          }
        }
      }
    }

    val confArchive = File.createTempFile(LOCALIZED_CONF_DIR, ".zip",
      new File(Utils.getLocalDir(sparkConf)))
    val confStream = new ZipOutputStream(new FileOutputStream(confArchive))
    //后面就是把这些文件写入到zip包的代码，略
}
```
可以看到，除了本地的`log4j.properties`和`metrics.properties`配置文件以外，还会读取`HADOOP_CONF_DIR`和`YARN_CONF_DIR`环境变量，读取对应目录下的文件放入`hadoopConfFiles`这个`HashMap`中，而这里面的文件都会压缩到`__spark_conf__.zip`中。  
再后续的代码就不分析了，可以参考网上其他文章。  
## 提交外部Yarn集群的障碍
所以，如果执行spark应用程序的机器中配置了 *HADOOP_CONF_DIR* 或 *YARN_CONF_DIR* 环境变量（如HDP的节点安装了对应客户端都会配置上），在Spark提交任务到外部yarn集群的时候，就会将里面的配置文件压缩传输到外部集群的Executor节点，这样Executor的各种操作都会使用原集群的配置，连接不到正确的Yarn服务，最后也就导致任务执行失败。  


# 解决方案
所以解决整个提交外部集群的问题，有两个问题要处理：
1. Spark应用代码使用外部集群的配置文件进行任务提交
   1. 一种方案是启动Spark应用后，创建`SparkContext`之前，将外部集群的配置写入当前classpath的前面（如classpath是`.:xxx.jar`，那么放在当前目录就可以）
   2. 另一种方案是启动Spark应用前，将外部集群的配置写入当前目录，并通过`jar uvf`打入jar包中；当然只是针对当前问题的话，无需打入jar包
2. Spark准备executor的资源时，使用外部集群配置文件
   1. 一种方案是，创建`SparkContext`之前，将`HADOOP_CONF_DIR`和`YARN_CONF_DIR`环境变量删除，提交任务后再恢复环境变量；这样不会把集群配置传给Executor，Executor使用的是fatjar包里面的配置文件，需要提前替换。
   2. 另一种方案是，将外部集群的配置写入一个目录，并在创建`SparkContext`之前，将`HADOOP_CONF_DIR`和`YARN_CONF_DIR`环境变量改为那个目录；这样正确的配置会传给Executor使用。

结合起来最终的方案： 
1. 外部集群的配置文件统一一个地方存储，可以直接存储在RDB。
2. 启动Spark应用的时候，检查需要提交到的Yarn集群，如果是外部集群，那么：
   1. 下载外部集群的配置文件到当前目录，同时复制到一个子目录里面
   2. 将`HADOOP_CONF_DIR`和`YARN_CONF_DIR`环境变量改为那个子目录（不能用当前目录，因为当前目录包含fat-jar，根据代码jar包也会打包过去Executor）
   3. 正常创建`SparkContext`
   4. 恢复环境变量

具体实现不外乎一些黑魔法（环境变量在JVM里面修改不了，但可以修改JVM用到的那个环境变量Map），再考虑下要不要放上来吧，反正这个最主要是思路和里面的坑。
