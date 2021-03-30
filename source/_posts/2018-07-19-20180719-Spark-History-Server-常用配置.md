---
title: 20180719 Spark History Server 常用配置
date: 2018-07-19 21:37:37
tags: Spark
---

一般配置在 `$SPARK_HOME/conf/spark-defaults.conf` 中，分为两类，history server 的配置和 spark client 的配置；基本原理就是 Spark 把 log 写在磁盘上，然后启动的 history server 会读取 log 文件，重现 Spark UI。

## History Server 读取的配置
spark.history.ui.port   1234
spark.history.fs.logDirectory  hdfs://xxx
spark.history.retainedApplications    200 

## Spark Client 读取的配置
spark.eventLog.enabled true
spark.eventLog.dir hdfs://xxx (和上面logDirectory对应）
spark.yarn.historyServer.address ip:port (和上面端口对应）
spark.eventLog.compress true

## 起停
`$SPARK_HOME/sbin/start-history-server.sh`
`$SPARK_HOME/sbin/stop-history-server.sh`

---
### 详细参考
https://spark.apache.org/docs/latest/running-on-yarn.html#spark-properties
https://spark.apache.org/docs/latest/monitoring.html#spark-configuration-options
### 注意
有时候为了方便切换集群，用 spark-submit 提交任务时增加参数 `--properties-file some-spark-defaults.conf`，会覆盖默认配置