---
title: 20180906 Spark Serialization Problem Revisited
date: 2018-09-06 08:20:30
tags: Spark
---
嵌套内部类可能会引起序列化问题
```java
public class SparkLocalTest {
    @Test
    public void localTest() {
        SparkSession spark = SparkSession.builder().master("local").getOrCreate();
        Dataset<Row> ds = spark.range(1000).toDF();
        ds.transform(flat2()).show(); // 正常
        ds.transform(flat1()).show(); // 报错

    }

    private static Function1<Dataset<Row>, Dataset<Row>> flat1() {
        return new AbstractFunction1<Dataset<Row>, Dataset<Row>>() {
            @Override
            public Dataset<Row> apply(Dataset<Row> v1) {
                return v1.flatMap(new FlatMapFunction<Row, String>() {
                    @Override
                    public Iterator<String> call(Row row) throws Exception {
                        return Stream.of("" + row.getLong(0)).iterator();
                    }
                }, Encoders.STRING()).toDF();
            }

        };
    }
// 与上面的方法区别仅在于使用了 lambda 简化代码
    private static Function1<Dataset<Row>, Dataset<Row>> flat2() {
        return new AbstractFunction1<Dataset<Row>, Dataset<Row>>() {
            @Override
            public Dataset<Row> apply(Dataset<Row> v1) {
                return v1.flatMap((FlatMapFunction<Row, String>) row -> Stream.of("" + row.getLong(0)).iterator(),
                                  Encoders.STRING()).toDF();
            }
        };
    }
}
```

虽然两个方法都是静态的，但是编译后会产生多个 class 文件：
 ```
 SparkLocalTest$1.class
 SparkLocalTest$1$1.class
 SparkLocalTest$2.class
 SparkLocalTest.class
 ```
 - transform 方法定义 `def transform[U](t: Dataset[T] => Dataset[U]): Dataset[U] = t(this)`，所以是用 flat 方法在 driver 端产生了一个 Function 对象并仅在 driver 端使用，所以该 Function 对象可以不实现序列化
 - flat1 由于嵌套，产生了两个文件，而且内部类 `SparkLocalTest$1$1` 还引用了外部类 `SparkLocalTest$1.class`，由于外部类是 AbstractFunction1 的子类，并没有实现序列化接口，由于要发送到 executor 进行操作，所以使用时会报错 `Task not serializable: java.io.NotSerializableException: SparkLocalTest$1`
 - 而 flat2 中的 lambda 编译后是静态域，`private static java.lang.Object $deserializeLambda$(java.lang.invoke.SerializedLambda);` 就没有序列化问题
 - 另一种方案是创建一个实现序列化的类：`abstract static class SerializableFunction1<T1, R> extends AbstractFunction1<T1, R> implements Serializable`
 ---
 从 [Java Nested Classes: Behind the Scenes](https://www.yihangho.com/java-nested-classes-behind-the-scenes/) 学到的：
- 内部类其实就是一种语法糖，对于 JVM 来说是透明的，JVM 看到的就是普通的类
- 编译器在编译时将内部类提取出来，因为要访问包围类的域，所以内部类要保存包围类的引用
- 当包围类的域是 private 的，包围类会产生一个静态访问方法供内部类使用；否则会直接通过引用访问


