---
title: HBase外网同步脚本
date: 2017-04-22T19:34:10+08:00
tags:
- 脚本
- HBase
---

## 问题
我们用多台云服务器搭建了HDP的hadoop集群，为了方便测试，在本地用virtualbox虚拟机搭建了一个架构完全一样的集群。为了测试程序，训练模型，本地的集群的数据也需要与云服务器上集群的一样。  
MySQL数据库可以通过主从备份实时同步，而HBase数据在配置同步的过程中就遇到了问题，常规的方法无法完成同步。  

## 常规方法
HBase可以设置备份，然而只能在同一个内网，而我们不想搭建vpn。  
然后是借助sqoop之类的工具，我们尝试对zookeeper等相关使用的端口开启内网映射，然而还是无法从外网访问，检查了iptables，没发现问题，原因未知。

## 最终方案
最后采用了最暴力的方案，就是使用HBase自带的export和import功能，把云端HBase整个表导出到文件再导入到本地集群HBase，缺点是慢，因为是通过mapReduce操作完成的，数据量大的时候尤其慢。而且不知为何，无法直接导出到本地文件系统，只能通过HDFS文件系统中转，也就是说，整个同步流程是：  
1. 云端HBase数据库表export到云端HDFS  
2. 云端HDFS的导出文件导出到云端Linux文件系统  
3. 本地集群通过scp下载云端的导出文件（因为安全问题，这里还分了两部，先scp下载到我的电脑，再scp上传到本地集群）  
4. 本地集群HDFS导入下载到的备份文件  
5. 本地集群HBase数据库import备份文件

整个过程比较麻烦，所以我写了个脚本，通过crontab每15分钟定时执行，脚本内容如下：  
```bash
 #!/bin/bash
 ssh 服务器名 > /dev/null 2>&1 << eeooff
 set HADOOP_USER_NAME=hdfs
 rm -rf /root/share
 hadoop fs -rm -r -f -skipTrash /backup/表名
 whoami
 /usr/hdp/current/hbase-client/bin/hbase org.apache.hadoop.hbase.mapreduce.Driver export '表名' /backup/表名
 hadoop fs -get /backup/表名 ~/bakcup
 exit
 eeooff

 rm -rf /home/***/tmp/backup
 scp -r 服务器名:/root/backup/ /home/***/tmp/
 scp -r /home/***/tmp/backup 本地集群主机名:/home/hdfs/

 ssh 本地集群主机名 > /dev/null 2>&1 << eeooff
 set HADOOP_USER_NAME=hdfs
 su - hdfs
 hadoop fs -rm -r -f -skipTrash /backup/表名
 hadoop fs -put /home/hdfs/backup /test
 /usr/hdp/current/hbase-client/bin/hbase org.apache.hadoop.hbase.mapreduce.Driver import 'SHARE_CHAIN' /backup/表名
 eeooff
```