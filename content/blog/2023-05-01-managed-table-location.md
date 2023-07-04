+++
title = "Managed table’s location for Spark & Hive"
+++

When using Spark with Hive metastore, there are some differences in warehouse location configuration, which controls where to save the data for managed tables.

If not using correctly, the result may be confusing.
There are some machanisms behind the scenes:

- Hive uses `hive.metastore.warehouse.dir` in `hive-site.xml` to set warehouse path, the default value is `/user/hive/warehouse` in `hive-default.xml.template`.
- Spark uses `spark.sql.warehouse.dir`. [It used Hive’s config, but that’s deprecated since Spark 2.0.0.](https://spark.apache.org/docs/latest/sql-data-sources-hive-tables.html) The default value is `$PWD/spark-warehouse`.
- When you create a database, metastore will record the database location in metastore’s table `DBS` as `DB_LOCATION_URI`.
    - If the database is created in Spark, the `DB_LOCATION_URI` is Spark’s warehouse config.
    - If the database is create in Hive, the `DB_LOCATION_URI` is Hive’s warehouse config.
- Hive will create a default database named `default`. If you don’t specify any specific database, you are using `default`. The `default`’s `DB_LOCATION_URI` is Hive’s warehouse config.
- When creating managed tables in a database, the table location will inherit from this database, no matter what tools you are using to create this table. For example, in database `default`, create two managed table with Spark and Hive, the location’s parent path will be same as Hive’s config.

Enough for the background, now let’s see some different examples:

I create a HDFS 3.3.2 in my local machine, set up a metastore for Hive 2.3.7 and Spark 3.3.2, all using the default configs.

---

Then:

Use spark-sql to create a table, without specifying database. `create table spark_text_table ...;`

- the database is `default`
- the table location is `hdfs://localhost:9000/user/hive/warehouse/spark_text_table`

---

Use spark-sql to create a database `db1`, then create a table `db1.spark_text_table`:

- the database is `db1`
- the table location is `file:/Users/zhenzhang/Downloads/temp/spark/spark-warehouse/db1.db/spark_text_table` , which is under my working directory

---

Use Hive to create a database `db2`, then create a table `db2.hive_text_table`:

- the database is `db2`
- the table location is `hdfs://localhost:9000/user/hive/warehouse/db2.db/hive_text_table`. This location comes from Hive’s default config, then add a database path level `db2.db`

---

One caveat for the default Spark warehouse path `$PWD/spark-warehouse`: as it uses the present working directory as the warehouse path, if you call spark-sql/spark-shell at different directories and create new databases, you will save tables to different locations, which will greatly increase the management overhead. 

But there is still a good news: the databases and tables metadata is saved in Hive’s metastore, you can still query all the tables created before, even in different directories.