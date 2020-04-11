---
title: 20180916 Dataset.col(colName) 无法区分衍生表的列
date: 2018-09-16 19:34:36
tags: Spark
---
### 问题：
若数据集1产生数据集2，则两者进行 join 然后使用 ds.col(colName) select 的时候结果中的列可能并非想选择的，例如想选择 left outer join 后右表的同名列：

```scala
object ScalaTest {
  case class Item(cuid: String, query: String)
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder().master("local").getOrCreate()
    import spark.implicits._
    val ds1 = spark.range(2)
    //+---+
    //| id|
    //+---+
    //|  0|
    //|  1|
    //+---+
    val ds2 = ds1.filter('id < 1)
    //+---+
    //| id|
    //+---+
    //|  0|
    //+---+
    val ds3 =  ds1.join(ds2, ds1.col("id") === ds2.col("id"), "left_outer")
    //+---+----+
    //| id|  id|
    //+---+----+
    //|  0|   0|
    //|  1|null|
    //+---+----+
    val ds4 = ds3.select(ds1.col("id")) // 选取第一列
    //+---+
    //| id|
    //+---+
    //|  0|
    //|  1|
    //+---+
    val ds5 = ds3.select(ds2.col("id")) // 原意是选取第二列，但结果还是第一列
    //+---+
    //| id|
    //+---+
    //|  0|
    //|  1|
    //+---+
  }
}
```
ds2 由 ds1 衍生，其中 `ds1.col("id") === ds2.col("id")` 这一句会产生警告 
```
WARN Column: Constructing trivially true equals predicate, 'id#0L = id#0L'. Perhaps you need to use aliases.
```
实际上这两个 column 的对象是同一个，在 Column 的 `===` 方法中会输出这一句警告。假如直接按照语义处理则会变成笛卡尔积的形式，这也是早期版本的处理方式，如 [这个问题](https://stackoverflow.com/questions/32190828/spark-sql-performing-carthesian-join-instead-of-inner-join/32191266) 中的情况，而后来 Dataset.join 中会特殊处理成 self equal join on key。
```
  def join(right: Dataset[_], joinExprs: Column, joinType: String): DataFrame = {
    // Note that in this function, we introduce a hack in the case of self-join to automatically
    // resolve ambiguous join conditions into ones that might make sense [SPARK-6231].
    // Consider this case: df.join(df, df("key") === df("key"))
    // Since df("key") === df("key") is a trivially true condition, this actually becomes a
    // cartesian join. However, most likely users expect to perform a self join using "key".
    // With that assumption, this hack turns the trivially true condition into equality on join
    // keys that are resolved to both sides.
```
所以上面的 ds3 是符合我们预期的，但 ds5 选择的也是 ds3 的第一列，因为 `ds1.col("id")` 与 `ds2.col("id")` 是同一个对象，所以 ds5 结果与 ds4 相同；利用 explain 可看出这一点：
```
== Physical Plan == // ds1
*Range (0, 2, step=1, splits=1)
== Physical Plan == // ds2
*Filter (id#0L < 1)
+- *Range (0, 2, step=1, splits=1)
== Physical Plan == // ds3
*BroadcastHashJoin [id#0L], [id#4L], LeftOuter, BuildRight
:- *Range (0, 2, step=1, splits=1)
+- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
   +- *Filter (id#4L < 1)
      +- *Range (0, 2, step=1, splits=1)
== Physical Plan == // ds4
*Project [id#0L]
+- *BroadcastHashJoin [id#0L], [id#4L], LeftOuter, BuildRight
   :- *Range (0, 2, step=1, splits=1)
   +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
      +- *Filter (id#4L < 1)
         +- *Range (0, 2, step=1, splits=1)
== Physical Plan == // ds5
*Project [id#0L]
+- *BroadcastHashJoin [id#0L], [id#4L], LeftOuter, BuildRight
   :- *Range (0, 2, step=1, splits=1)
   +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
      +- *Filter (id#4L < 1)
         +- *Range (0, 2, step=1, splits=1)
```

___
### 解决办法：
1. 利用 withColumnRenamed 或 as 重命名，新列名可以与原来相同，只是借助重命名这个动作使其产生一个新的引用对象
```scala
val ds2 = ds1.filter('id < 1).withColumn("id",'id)
```
此时执行计划就发生了改变，可以看出这一次 ds4 输出的是列 [id#0L]，而 ds5 是 [id#4L]，正好分别是 ds3 中的两列。
```
== Physical Plan == // ds1
*Range (0, 2, step=1, splits=1)
== Physical Plan == // ds2
*Filter (id#0L < 1)
+- *Range (0, 2, step=1, splits=1)
== Physical Plan == // ds3
*BroadcastHashJoin [id#0L], [id#4L], LeftOuter, BuildRight
:- *Range (0, 2, step=1, splits=1)
+- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
   +- *Project [id#0L AS id#4L]
      +- *Filter (id#0L < 1)
         +- *Range (0, 2, step=1, splits=1)
== Physical Plan == // ds4
*Project [id#0L]
+- *BroadcastHashJoin [id#0L], [id#4L], LeftOuter, BuildRight
   :- *Range (0, 2, step=1, splits=1)
   +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
      +- *Project [id#0L AS id#4L]
         +- *Filter (id#0L < 1)
            +- *Range (0, 2, step=1, splits=1)
== Physical Plan == // ds5
*Project [id#4L]
+- *BroadcastHashJoin [id#0L], [id#4L], LeftOuter, BuildRight
   :- *Range (0, 2, step=1, splits=1)
   +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
      +- *Project [id#0L AS id#4L]
         +- *Filter (id#0L < 1)
            +- *Range (0, 2, step=1, splits=1)
```
2. 使用 sql string，不会有任何问题；但是否应该在代码中使用大量的 sql 语句呢？一个可能的问题是维护困难，没有编译期检查，多行 sql 之间容易发生错误。
```scala
spark.sql(" SELECT ds2.id FROM ds1 LEFT OUTER JOIN ds2 ON ds1.id = ds2.id ")
```

---
[另一个类似的问题](http://mail-archives.apache.org/mod_mbox/spark-user/201510.mbox/%3CCAFQ3t_zgNka1fOZQZNUqyO-6F9VqF7TLHOCqDFfMAzckX1hoFA@mail.gmail.com%3E)