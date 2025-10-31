+++
title = "The Evolution of Big Data Table Management"

+++

This is just a summary of my understanding of how to organize bigdata in tables. So the level arrangement is rather random.

# Level 1
No tables, just a bunch of files in a directory, read a single file or whole directory to process.

- Pros: easy to use, quick for testing.
- Cons: no optimizations.

# Level 1.5
Hive style partitioned/bucketed directories.

- Pros: easy to understand, prune data with partitions.
- Cons: partition evolution is hard, file listing is slow in S3, and may reach API limits.

# Level 2
Use Hive(or Glue...) metastore.

- Pros: one place to find and manage tables.
- Cons: same as Level 1.5.

# Level 2.5
Zorder.

- Pros: more efficient for different filter combinations.
- Cons: not a table property, need to manually organize the data, write amplification.

# Level 3
Table formats: Iceberg, Delta Lake, Hudi...

- Pros: ACID, schema evolution management, data updates, time travel, branch/tags...
- Cons: lost control at file level, every task need to go through these table formats.

# Level 3.5
Liquid clustering, auto clustering based on query usage, incremental.

Auto compaction on Tables.

- Pros: more efficient on query filter, automatic optimization.
- Cons: closed source, vendor lock-in.


----
# References
- [https://www.dremio.com/blog/how-z-ordering-in-apache-iceberg-helps-improve-performance/](https://www.dremio.com/blog/how-z-ordering-in-apache-iceberg-helps-improve-performance/)
- [https://docs.databricks.com/aws/en/delta/clustering](https://docs.databricks.com/aws/en/delta/clustering)
- [https://delta.io/blog/liquid-clustering/](https://delta.io/blog/liquid-clustering/)
- [https://delta.io/blog/2023-06-03-delta-lake-z-order/](https://delta.io/blog/2023-06-03-delta-lake-z-order/)
- [https://aws.amazon.com/blogs/aws/aws-glue-data-catalog-now-supports-automatic-compaction-of-apache-iceberg-tables/](https://aws.amazon.com/blogs/aws/aws-glue-data-catalog-now-supports-automatic-compaction-of-apache-iceberg-tables/)
