---
title: Kettle Java API处理Hadoop数据
date: 2018-05-15T17:29:56+08:00
tags:
- Kettle
- Hadoop
image: yyz.jpg
---
## 前言
最近因为数据处理的需求, 用到Kettle进行MySQL到HDFS的数据导入,而Kettle的GUI界面导入比较繁琐,不易于复用,所以考虑其Java API.  
但是,网上的资料实在少得可怜, 而官网的文档也仅仅给出了一个例子,而且是版本很旧的.  于是只能根据这个很旧的版本, 加上Maven仓库摸索, 再加上官方最新版API文档,慢慢摸出来.  

## 代码清单
最后的结果就是这篇文章, 废话也不想多说了,也懒得打字,就是普通的Maven项目,主要三个文件:
1. `pom.xml`: Kettle依赖的版本比较麻烦,这个是个坑;
2. 一个Java示例文件, 放了一个`MySQL => MySQL` 和一个 `MySQL => HDFS` 的例子,详见注释;
3. Java里面写了一个自动读取数据源的方法, 把所有用到的数据源按下文给定的xml格式配置好,放到`resources/db`下面即可被程序读取.

## 目录结构
```bash
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── turingdi
    │   │           └── kettle
    │   │               └── demo
    │   │                   └── App.java
    │   └── resources
    │       ├── db
    │       │   └── 235test.xml
    │       └── log4j.properties
    └── test
        └── java
            └── com
                └── turingdi
                    └── kettle
                        └── demo
                            └── AppTest.java
```
## 代码
### pom.xml
给出核心的变量和依赖部分:  
```xml
<properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <pentaho.kettle.version>4.1.0-stable</pentaho.kettle.version>
        <pentaho.kettle.plugin.version>8.0.0.4-247</pentaho.kettle.plugin.version>
    </properties>

    <repositories>
        <repository>
            <snapshots>
                <enabled>true</enabled>
                <updatePolicy>daily</updatePolicy>
            </snapshots>
            <id>pentaho</id>
            <url>http://nexus.pentaho.org/content/groups/omni/</url>
        </repository>
    </repositories>

    <dependencies>
        <dependency>
            <groupId>pentaho-kettle</groupId>
            <artifactId>kettle-core</artifactId>
            <version>${pentaho.kettle.plugin.version}</version>
        </dependency>
        <dependency>
            <groupId>pentaho-kettle</groupId>
            <artifactId>kettle-db</artifactId>
            <version>4.4.0-stable</version>
        </dependency>
        <dependency>
            <groupId>pentaho-kettle</groupId>
            <artifactId>kettle-engine</artifactId>
            <version>${pentaho.kettle.plugin.version}</version>
        </dependency>
        <dependency>
            <groupId>pentaho-kettle</groupId>
            <artifactId>kettle-ui-swt</artifactId>
            <version>${pentaho.kettle.plugin.version}</version>
        </dependency>
        <dependency>
            <groupId>pentaho</groupId>
            <artifactId>pentaho-big-data-kettle-plugins-hdfs</artifactId>
            <version>${pentaho.kettle.plugin.version}</version>
        </dependency>
        <dependency>
            <groupId>pentaho</groupId>
            <artifactId>pentaho-big-data-api-hdfs</artifactId>
            <version>${pentaho.kettle.plugin.version}</version>
        </dependency>
        <dependency>
            <groupId>pentaho</groupId>
            <artifactId>pentaho-big-data-impl-cluster</artifactId>
            <version>${pentaho.kettle.plugin.version}</version>
        </dependency>

        <!--插件所需依赖开始-->
        <dependency>
            <groupId>pentaho</groupId>
            <artifactId>metastore</artifactId>
            <version>${pentaho.kettle.plugin.version}</version>
        </dependency>
        <dependency>
            <groupId>org.pentaho</groupId>
            <artifactId>pentaho-hadoop-shims-api</artifactId>
            <version>8.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>commons-configuration</groupId>
            <artifactId>commons-configuration</artifactId>
            <version>1.9</version>
        </dependency>
        <!--插件所需依赖结束-->

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.6.0-cdh5.9.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-hdfs</artifactId>
            <version>2.6.0-cdh5.9.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.6.0-cdh5.8.4</version>
        </dependency>

        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.2</version>
        </dependency>

        <dependency>
            <groupId>commons-vfs</groupId>
            <artifactId>commons-vfs</artifactId>
            <version>1.0</version>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>
        <dependency>
            <groupId>org.scannotation</groupId>
            <artifactId>scannotation</artifactId>
            <version>1.0.3</version>
            <exclusions>
                <exclusion>
                    <groupId>javassist</groupId>
                    <artifactId>javassist</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.22.0-GA</version>
        </dependency>

        <dependency>
            <groupId>com.turingdi</groupId>
            <artifactId>commonutils</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.41</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

### Java的示例
包含`MySQL => MySQL`和`MySQL => HDFS` 的方法:  
```java
package com.turingdi.kettle.demo;

import com.turingdi.commonutils.basic.FileUtils;
import org.pentaho.big.data.api.cluster.NamedCluster;
import org.pentaho.big.data.impl.cluster.NamedClusterImpl;
import org.pentaho.big.data.impl.cluster.NamedClusterManager;
import org.pentaho.big.data.impl.cluster.NamedClusterServiceOsgiImpl;
import org.pentaho.big.data.kettle.plugins.hdfs.trans.HadoopFileOutputMeta;
import org.pentaho.di.cluster.ClusterSchema;
import org.pentaho.di.core.Const;
import org.pentaho.di.core.KettleEnvironment;
import org.pentaho.di.core.NotePadMeta;
import org.pentaho.di.core.database.Database;
import org.pentaho.di.core.database.DatabaseMeta;
import org.pentaho.di.core.exception.KettleException;
import org.pentaho.di.core.plugins.PluginFolder;
import org.pentaho.di.core.plugins.StepPluginType;
import org.pentaho.di.core.util.EnvUtil;
import org.pentaho.di.trans.Trans;
import org.pentaho.di.trans.TransHopMeta;
import org.pentaho.di.trans.TransMeta;
import org.pentaho.di.trans.step.StepMeta;
import org.pentaho.di.trans.steps.selectvalues.SelectValuesMeta;
import org.pentaho.di.trans.steps.tableinput.TableInputMeta;
import org.pentaho.di.trans.steps.tableoutput.TableOutputMeta;
import org.pentaho.di.trans.steps.textfileoutput.TextFileField;
import org.pentaho.runtime.test.action.impl.RuntimeTestActionServiceImpl;
import org.pentaho.runtime.test.impl.RuntimeTesterImpl;

import java.io.DataOutputStream;
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

public class App {
    //存放读取到的xml字符串
    private static String[] databasesXML;
    //kettle插件的位置
    private static final String KETTLE_PLUGIN_BASE_FOLDER = "/Users/leibnizhu/Downloads/kettle/plugins";

    public static void main(String[] args) throws KettleException, IOException {
        // 这几句必须有, 官网例子是错的, 用来加载插件的
        System.setProperty("hadoop.home.dir", "/");
        StepPluginType.getInstance().getPluginFolders().add(new PluginFolder(KETTLE_PLUGIN_BASE_FOLDER,true, true));

        EnvUtil.environmentInit();
        KettleEnvironment.init();

        // 加载db目录下的所有存放数据库配置的xml文件
        // 这个在官方例子也是没有的, 自己写的, 而且没给出xml的例子, google到的一篇博客里面的, 坑死了
        String rootPath = App.class.getResource("/").getPath();
        File dbDir = new File(rootPath, "db");
        List<String> xmlStrings = new ArrayList<>();
        for (File xml : Objects.requireNonNull(dbDir.listFiles())) {
            if (xml.isFile() && xml.getName().endsWith(".xml")) {
                xmlStrings.add(FileUtils.ReadFile(xml.getAbsolutePath()));
            }
        }
        databasesXML = xmlStrings.toArray(new String[0]);

        // 调用下面的方法, 创建一个复制数据库表的Transform任务
//        TransMeta transMeta = buildCopyTable(
//                "trans",
//                "235test",
//                "user",
//                new String[]{"id", "name"},
//                "235test",
//                "user2",
//                new String[]{"id2", "name2"}
//        );

        TransMeta transMeta = buildCopyTableToHDFS(
                "trans",
                "235test",
                "user",
                new String[]{"id", "name"}
        );

        // 把以上transform保存到文件, 这样可以用Spoon打开,检查下有没有问题
        String fileName = "/Users/leibnizhu/Desktop/test.ktr";
        String xml = transMeta.getXML();
        DataOutputStream dos = new DataOutputStream(new FileOutputStream(new File(fileName)));
        dos.write(xml.getBytes("UTF-8"));
        dos.close();
        System.out.println("Saved transformation to file: " + fileName);

        // 生成SQL,用于创建表(如果不存在的话)
        String sql = transMeta.getSQLStatementsString();
        // 执行以上SQL语句创建表
        Database targetDatabase = new Database(transMeta.findDatabase("235test"));
        targetDatabase.connect();
        targetDatabase.execStatements(sql);

        // 执行transformation...
        Trans trans = new Trans(transMeta);
        trans.execute(null);
        trans.waitUntilFinished();

        //  断开数据库连接
        targetDatabase.disconnect();
    }


    /**
     * Creates a new Transformation using input parameters such as the tablename to read from.
     *
     * @param transformationName The name of the transformation
     * @param sourceDatabaseName 数据源, 对应xml里面的name字段
     * @param sourceTableName    The name of the table to read from
     * @param sourceFields       The field names we want to read from the source table
     * @param targetDatabaseName 复制的去向, 对应xml里面的name字段
     * @param targetTableName    The name of the target table we want to write to
     * @param targetFields       The names of the fields in the target table (same number of fields as sourceFields)
     * @return A new transformation metadata object
     * @throws KettleException In the rare case something goes wrong
     */

    private static TransMeta buildCopyTable(String transformationName,
                                            String sourceDatabaseName, String sourceTableName, String[] sourceFields,
                                            String targetDatabaseName, String targetTableName, String[] targetFields)
            throws KettleException {
        try {
            // 创建transformation...
            TransMeta transMeta = new TransMeta();
            transMeta.setName(transformationName);

            // 增加数据库连接的元数据
            for (String aDatabasesXML : databasesXML) {
                DatabaseMeta databaseMeta = new DatabaseMeta(aDatabasesXML);
                transMeta.addDatabase(databaseMeta);
            }
            DatabaseMeta sourceDBInfo = transMeta.findDatabase(sourceDatabaseName);
            DatabaseMeta targetDBInfo = transMeta.findDatabase(targetDatabaseName);

            // 增加备注
            String note = "Reads information from table [" + sourceTableName + "] on database [" + sourceDBInfo + "]" + Const.CR + "After that, it writes the information to table [" + targetTableName + "] on database [" + targetDBInfo + "]";
            NotePadMeta ni = new NotePadMeta(note, 150, 10, -1, -1);
            transMeta.addNote(ni);

            // 创建读数据库的step
            String fromStepName = "read from [" + sourceTableName + "]";
            TableInputMeta tii = new TableInputMeta();
            tii.setDatabaseMeta(sourceDBInfo);
            tii.setSQL("SELECT " + Const.CR + String.join(",", sourceFields) + " " + "FROM " + sourceTableName);

            StepMeta fromStep = new StepMeta(fromStepName, tii);
            //以下几句是给Spoon看的, 用处不大
            fromStep.setLocation(150, 100);
            fromStep.setDraw(true);
            fromStep.setDescription("Reads information from table [" + sourceTableName + "] on database [" + sourceDBInfo + "]");
            transMeta.addStep(fromStep);

            // 创建一个修改字段名的step
            SelectValuesMeta svi = new SelectValuesMeta();
            //配置字段名修改的规则, 这里跟官方例子差别很大, 坑不少
            svi.allocate(sourceFields.length, 0, 0);
            for (int i = 0; i < sourceFields.length; i++) {
                svi.getSelectName()[i] = sourceFields[i];
                svi.getSelectRename()[i] = targetFields[i];
            }

            String selStepName = "Rename field names";
            StepMeta selStep = new StepMeta(selStepName, svi);
            //以下几句是给Spoon看的, 用处不大
            selStep.setLocation(350, 100);
            selStep.setDraw(true);
            selStep.setDescription("Rename field names");
            transMeta.addStep(selStep);

            //建立读数据库step与修改字段名step的连接,增加到transformation中
            TransHopMeta shi = new TransHopMeta(fromStep, selStep);
            transMeta.addTransHop(shi);

            // 创建一个输出到表的step
            String toStepName = "write to [" + targetTableName + "]";
            TableOutputMeta toi = new TableOutputMeta();
            toi.setDatabaseMeta(targetDBInfo);
            toi.setTablename(targetTableName);
            toi.setCommitSize(200);
            toi.setTruncateTable(true);
            toi.setSchemaName("test");
            toi.setTruncateTable(false);

            StepMeta toStep = new StepMeta(toStepName, toi);
            //以下几句是给Spoon看的, 用处不大
            toStep.setLocation(550, 100);
            toStep.setDraw(true);
            toStep.setDescription("Write information to table [" + targetTableName + "] on database [" + targetDBInfo + "]");
            transMeta.addStep(toStep);

            // 建立修改字段名step到输出到数据库step的连接
            TransHopMeta hi = new TransHopMeta(selStep, toStep);
            transMeta.addTransHop(hi);

            // 返回
            return transMeta;
        } catch (Exception e) {
            throw new KettleException("An unexpected error occurred creating the new transformation", e);
        }
    }

    private static TransMeta buildCopyTableToHDFS(String transformationName,
                                            String sourceDatabaseName, String sourceTableName, String[] sourceFields)
            throws KettleException {
        try {
            // 创建transformation...
            TransMeta transMeta = new TransMeta();
            transMeta.setName(transformationName);

            // 增加数据库连接的元数据
            for (String aDatabasesXML : databasesXML) {
                DatabaseMeta databaseMeta = new DatabaseMeta(aDatabasesXML);
                transMeta.addDatabase(databaseMeta);
            }
            DatabaseMeta sourceDBInfo = transMeta.findDatabase(sourceDatabaseName);

            // 增加备注
            String note = "Reads information from table [" + sourceTableName + "] on database [" + sourceDBInfo + "]" + Const.CR + "After that, it writes the information to HDFS ]";
            NotePadMeta ni = new NotePadMeta(note, 150, 10, -1, -1);
            transMeta.addNote(ni);

            // 创建读数据库的step
            String fromStepName = "read from [" + sourceTableName + "]";
            TableInputMeta tii = new TableInputMeta();
            tii.setDatabaseMeta(sourceDBInfo);
            tii.setSQL("SELECT " + Const.CR + String.join(",", sourceFields) + " " + "FROM " + sourceTableName);

            StepMeta fromStep = new StepMeta(fromStepName, tii);
            //以下几句是给Spoon看的, 用处不大
            fromStep.setLocation(150, 100);
            fromStep.setDraw(true);
            fromStep.setDescription("Reads information from table [" + sourceTableName + "] on database [" + sourceDBInfo + "]");
            transMeta.addStep(fromStep);

            NamedClusterManager clusterManager = new NamedClusterManager();
            NamedCluster cluster = new NamedClusterImpl();
            cluster.setStorageScheme("hdfs");
            cluster.setHdfsHost("bitest01");
            cluster.setHdfsPort("8020");
            cluster.setName("cloudera");
            cluster.setHdfsUsername("");
            cluster.setHdfsPassword("");
            clusterManager.setClusterTemplate(cluster);
//            transMeta.setNamedClusterServiceOsgi();
//            clusterManager.getClusterTemplate().setHdfsHost("bitest01");
            HadoopFileOutputMeta hadoopOut = new HadoopFileOutputMeta(clusterManager, null, null);
//                    new RuntimeTestActionServiceImpl(null, null),
//                    new RuntimeTesterImpl(null, null, "test"));
            hadoopOut.setOutputFields(new TextFileField[]{});
            hadoopOut.setFilename("hdfs://bitest01:8020/tmp/aa");
            hadoopOut.setExtension("txt");
            hadoopOut.setFileCompression("None");
            hadoopOut.setSourceConfigurationName("Cloudera");
            hadoopOut.setSeparator(",");
            hadoopOut.setFileFormat("UNIX");
            hadoopOut.setEncoding("UTF-8");
            StepMeta hadoopStep = new StepMeta("HDFSOutput", hadoopOut);
            hadoopStep.setLocation(550, 100);
            hadoopStep.setDraw(true);
            transMeta.addStep(hadoopStep);
            TransHopMeta hhm = new TransHopMeta(fromStep, hadoopStep);
            transMeta.addTransHop(hhm);

            // 返回
            return transMeta;
        } catch (Exception e) {
            throw new KettleException("An unexpected error occurred creating the new transformation", e);
        }
    }
}
```

### 存储数据库源配置的xml文件
放在`resources/db`下面:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<connection>
    <name>test</name>
    <server>192.168.1.*</server>
    <type>MySQL</type>
    <access>Native</access>
    <database>test</database>
    <port>3306</port>
    <username>root</username>
    <password>root</password>
    <servername/>
    <data_tablespace/>
    <index_tablespace/>
    <attributes>
        <attribute>
            <code>EXTRA_OPTION_MYSQL.defaultFetchSize</code>
            <attribute>500</attribute>
        </attribute>
        <attribute>
            <code>EXTRA_OPTION_MYSQL.useCursorFetch</code>
            <attribute>true</attribute>
        </attribute>
        <attribute>
            <code>EXTRA_OPTION_MYSQL.useSSL</code>
            <attribute>false</attribute>
        </attribute>
        <attribute>
            <code>EXTRA_OPTION_MYSQL.useUnicode</code>
            <attribute>true</attribute>
        </attribute>
        <attribute>
            <code>EXTRA_OPTION_MYSQL.characterEncoding</code>
            <attribute>UTF-8</attribute>
        </attribute>
    </attributes>
</connection>
```