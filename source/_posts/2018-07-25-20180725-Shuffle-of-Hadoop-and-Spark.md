---
title: 20180725 Shuffle of Hadoop and Spark
date: 2018-07-25 20:09:18
tags: Spark
---
## Hadoop Shuffle
### map
1. 每个 mapper 有一个环形缓存，当数据大小达到阈值时，溢写到磁盘（会运行combiner）
2. 溢写到磁盘的数据根据 reducer 进行分区且文件内按照 key 进行排序
3. 多次溢写生成多个文件，最后合并数据文件（文件数大于3运行combiner）和索引文件

### reduce
1. reducer 读取相应 mapper 的分区文件
2. 读取文件太大或文件数多于阈值，会溢写到磁盘（会运行combiner）
3. 合并所有读取的文件，归并排序中的merge


## Spark Shuffle
### HashShuffleManager
-  每个 mapper 根据数据 hash 写入不同的分区文件 
-  每个 mapper task 产生下一步 task 个文件，共 Mappers * Reducers
-  优化（consolidateFiles）后，每个 core 产生下一步 task 个文件， E \* (C / T) \* R（E: –num-executors, C: –executor-cores, T: spark.task.cpus; 一般 spark.task.cpus = 1，就变成 总cpu * Reducers）

### SortShuffleManager
#### normal
- 和 Hadoop 的 shuffle 过程很像，不过 Hadoop 是环形缓存区，Spark 是根据 reduce 算子不同选择不同结构，比如 reduceByKey 选择 Map，这点又和 combiner 原理相同；Hadoop 是在内存中数据大小超过阈值溢写，Spark是默认超过 10000 条
- 最后每个 task 只产生一个数据文件和一个索引文件

#### bypass
- reduce task 小于 spark.shuffle.sort.bypassMergeThreshold 且 非聚合算子
- 不进行排序，一个 map task 中根据 hash 生成 reduce 个文件，最后进行合并；和 HashShuffleManager 类似

——————
参考：
https://0x0fff.com/hadoop-mapreduce-comprehensive-description/
https://tech.meituan.com/spark_tuning_pro.html
https://0x0fff.com/spark-architecture-shuffle/