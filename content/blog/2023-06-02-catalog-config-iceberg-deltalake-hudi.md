+++
title = "Catalog config for Iceberg & Delta Lake & Hudi"
+++

Iceberg, Delta Lake and Hudi are three projects to build data lakes or Lakehouses. They all provide ACID transactions, metadata evolution, time travel and many other functions to Spark.

All of them use the spark session extension and spark sql catalog to augment Spark SQL’s functionality.

One background info: Spark has a default catalog named `spark_catalog`, users can use a custom implementation, and also create new catalogs.

Below are some config examples and comparisons using these 3 projects, all derived and modified from official documents.

Versions: Spark 3.3.2, Iceberg 1.3.0, Delta Lake 2.3.0, Hudi 0.13.1

# Iceberg

Iceberg’s config is very flexible, you can both replace the `spark_catalog` implementation and create new ones.

It has two spark catalog implementations:

- `org.apache.iceberg.spark.SparkSessionCatalog` adds support for Iceberg tables to Spark’s built-in catalog, and delegates to the built-in catalog for non-Iceberg tables. This one can only be used on `spark_catalog`
- `org.apache.iceberg.spark.SparkCatalog` supports a Hive Metastore or a Hadoop warehouse as a catalog. This one can be used on `spark_catalog` and other user named catalogs. But this catalog will only load iceberg tables, meaning you can’t see plain old hive tables

```
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:1.3.0\
    --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
    --conf spark.sql.catalog.hive=org.apache.iceberg.spark.SparkCatalog \
    --conf spark.sql.catalog.hive.type=hive \
    --conf spark.sql.catalog.hive.uri=thrift://localhost:9083
```

This example creates a new catalog named `hive`, connecting to Hive’s metastore. So there are two catalogs, `spark_catalog` and `hive`. 

- the `spark_catalog` can only handle non-iceberg tables, even if you created some iceberg tables in it using other config before
- the `hive` can only handle iceberg tables, if you created some non-iceberg tables in it using other config before

When creating a table without specifying catalog, it’ll use `spark_catalog`; when you want to create a iceberg table, you must use the catalog `hive`, for example 

```
# first
create database hive.iceberg_db;
# then
create table hive.iceberg_db.iceberg_table ...;

# or
use hive.iceberg_db;
create table iceberg_table ...; 
```

When creating table under iceberg’s catalog, the tables default to iceberg, so you don’t need to specify `using iceberg`, but I think adding it is a good practice.

One interesting thing is that Hive can read the iceberg table schema, as it’s stored in the metastore, but can’t read the data.

Another config example:

```
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:1.3.0\
    --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
    --conf spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog \
    --conf spark.sql.catalog.spark_catalog.type=hive \
    --conf spark.sql.catalog.spark_catalog.uri=thrift://localhost:9083 
```

There is only one catalog `spark_catalog`, it can handle both iceberg and non-iceberg tables.

A final example:

```
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:1.3.0\
    --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
    --conf spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog \
    --conf spark.sql.catalog.spark_catalog.type=hive \
    --conf spark.sql.catalog.local=org.apache.iceberg.spark.SparkCatalog \
    --conf spark.sql.catalog.local.type=hadoop \
    --conf spark.sql.catalog.local.warehouse=/Users/zhenzhang/Downloads/temp/iceberg/warehouse
```

- The `spark_catalog` is using the `org.apache.iceberg.spark.SparkSessionCatalog`, not `org.apache.iceberg.spark.SparkCatalog`, so it can handle both iceberg and non-iceberg tables. The metadata is saved in Hive’s metastore.
- Catalog `local` can only handle iceberg tables. The metadata is saved in local

Two different catalog implementations add some complexity in config, but once you understand them, it’s quite flexible to switch among different catalogs.

# Delta Lake

Compared to Iceberg, the config is simple, you can only replace the `spark_catalog` implementation, not adding new one:

```
spark-sql --packages io.delta:delta-core_2.12:2.3.0 \
  --conf spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension \
  --conf spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog
```

# Hudi

Same as Delta Lake, replacing `spark_catalog` 

```
spark-shell --packages org.apache.hudi:hudi-spark3.3-bundle_2.12:0.13.1 \
  --conf spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension \
  --conf spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog

```