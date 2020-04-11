---
title: 20190504 Spark ShuffleManager
date: 2019-05-04 16:14:40
tags: Spark
---
Spark 版本: 2.4.2
* ShuffleManager 负责注册 shuffle, 以及获取对应分区的 writer/reader
* 每一个 ShuffleDependency 在初始化时会注册 shuffle 并获得一个 ShuffleHandle
* ShuffleManager 现在只有一个实现: SortShuffleManager
* 根据 ShuffleHandle 的不同, 获取不同的writer, 也就对应不同的 shuffle write 方式
## BypassMergeSortShuffleHandle -> BypassMergeSortShuffleWriter
条件: 
dep.mapSideCombine == false 
&& dep.partitioner.numPartitions <= SHUFFLE_SORT_BYPASS_MERGE_THRESHOLD(200)
方式:
根据下游分区, 每个分区的数据写入一个文件, 最后合并文件
不进行 map side combine, 所以必须设定为 false
同时打开文件流/serializer 不能太多, 所以设定了一个阈值
## SerializedShuffleHandle -> UnsafeShuffleWriter
条件: 
dependency.serializer.supportsRelocationOfSerializedObjects 
&& dependency.mapSideCombine == false
&& numPartitions <= MAX_SHUFFLE_OUTPUT_PARTITIONS_FOR_SERIALIZED_MODE(16777216)
方式:
不使用 map side combine, mapper 获取到一条数据后立刻序列化
所有操作都是在序列化格式下, 比如排序/合并等, 所以需要序列化后的格式支持这些操作, 以避免重复序列化/反序列化
减少了内存占用/GC
## BaseShuffleHandle -> SortShuffleWriter
不满足以上条件时使用
使用了 ExternalSorter 先对数据根据 partition 分区, 然后溢写到磁盘时内部排序( if dep.mapSideCombine == true)
combine 时使用 PartitionedAppendOnlyMap, 否则使用 PartitionedPairBuffer
