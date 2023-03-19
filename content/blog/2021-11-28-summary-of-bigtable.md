+++
title = "Summary of Bigtable"
+++

A bigtable is a sparse, distributed, persistent multi-dimensional sorted map.

# Features

- Wide applicability: from backend bulk processing to realtime data serving
- Scalability: supports PB level data and thousands of commodity servers
- High performance and avaliability

# Data Model

(Row, column, time) -> value.

Columns are divided into different families for access control and locality refinement (see below).

Time is used to mark different versions of value.

# Building Blocks

- SSTable: immutable sorted string table. Use index to faster block locates
- Chubby: provide namespace of directory and files, callback notification, session control, distributed locks. (Similar to Zookeeper)

# Implementation

Three major components:

- Master server: assign tablets(key range), balance tablet servers load, handle schema changes
- Tablet server: serve read/write, split tablet when needed
- Client: most clients never communicate with Master server, so single-master is not a problem in system load



A write operation consists commit log -> memtable -> SSTable.

The immutable character of SSTable means it need compaction to optimize data. Three type of compactions:

- Minor: memtable -> one SSTable
- Merging: memtable + a few SSTables -> one SSTable
- Major: all SSTables -> one SSTable (really reclaim resources of deleted data)

# Refinements

- Locality groups: group column families that are always accessed together in same SSTable for read performance. Locality groups can be put into different places(RAM/Disk)
- Compression: two-pass custom compression scheme
- Caching
- Bloom filter: for fewer disk seek
- Single commit-log file
- Compaction to reduce recovery time
- Exploit SSTable immutablity for concurrency

# Performance

Random writes are better than random reads.

The aggregate performance is growing as cluster scaling, but single machine performance is degrading, especially read/write without RAM cache.

# Lessons

- Distributed systems are vulnerable to many types of failures, from network to other dependency modules
- Delay adding new featuers until it is clear how the new features will be used
- Build proper system-level monitoring
- Keep design simple

# Features related to Cassandra

- Every value also has a timestamp, but this timestamp is used to resolve conflict
- Persistent level contains commit-log/memtable/SSTable
- SSTable compaction for recovery

  
