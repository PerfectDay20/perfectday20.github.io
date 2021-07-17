---
layout: single
title:  "Spark 中的序列化陷阱"
date:   2018-01-19 22:35:05 +0800
tags: 
- Java 
- Spark
---

Spark 的代码分为 Driver 端执行的部分和 Executor 端执行的部分，Driver 端分发任务的同时，会通过序列化传送 Executor 需要的对象，由于 Java 序列化的一些特性，初学者在使用时容易碰到一些陷阱。
## 陷阱1: 没有序列化
最常见的一个错误就是传递的类不可序列化，如下面的例子：

```java
package test;
import ...
/**
 * Created by PerfectDay20.
 */
public class Main {
    public static void main(String[] args) {
        SparkConf conf = new SparkConf().setAppName("test");
        JavaSparkContext javaSparkContext = new JavaSparkContext(conf);

        JavaRDD<Integer> rdd =
                javaSparkContext.parallelize(IntStream.range(1, 10000).boxed().collect(Collectors.toList()), 10);

        Util util = new Util();
        rdd.map(util::process); // 序列化错误
    }

}

class Util implements Serializable{
    public int process(int i) {
        return i + 1;
    }
}
```
这里的 `Util` 类没有实现 `Serializable` 接口，由 Driver 创建实例后，在 `map` 中传递给各个 Executor，导致序列化失败报错：
```
Exception in thread "main" org.apache.spark.SparkException: Task not serializable
	at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:298)
	at org.apache.spark.util.ClosureCleaner$.org$apache$spark$util$ClosureCleaner$$clean(ClosureCleaner.scala:288)
	at org.apache.spark.util.ClosureCleaner$.clean(ClosureCleaner.scala:108)
	at org.apache.spark.SparkContext.clean(SparkContext.scala:2094)
	at org.apache.spark.rdd.RDD$$anonfun$map$1.apply(RDD.scala:370)
	...
Caused by: java.io.NotSerializableException: test.Util
Serialization stack:
	- object not serializable (class: test.Util, value: test.Util@1290ed28)
	...
```

这种错误根据不同的需求有不同的解决方法：
1. 最简单的方法就是让`Util`类可序列化： `class Util implements Serializable`
2. 如果是工具类，比如上例，没有必要创建`Util`实例，直接将`process`替换为静态方法：`public static int process(int i)`，然后在`map`方法中：`rdd.map(Util::process)`
3. 如果调用的方法比较简单，就不用创建`Util`类，直接在`map`中写 lambda 表达式即可：`rdd.map( i -> i + 1 )`；这种方法其实是创建了一个实现`Function`接口的匿名类，而`Function`接口的定义是：`public interface Function<T1, R> extends Serializable`，所以自然就可序列化了
4. 另外可以在`map`中创建`Util`实例，这样的话，每个实例都是在 Executor 端创建的，因为不需要序列化传递，就不存在序列化问题了：
```java
        rdd.map(i->{
            Util util = new Util();
            LOG.info(""+util);
            return util.process(i);
        })
```
但是这种情况对于每一个`i`都要创建一个实例，在一些重量级操作，比如创建数据库链接时，可以考虑采用`mapPartition`，这样如上面的例子，就只需要创建10个`Util`实例：
```java
        rdd.mapPartitions(iterator->{
            Util util = new Util();
            List<Integer> list = new ArrayList<>();
            iterator.forEachRemaining(i -> list.add(util.process(i)));
            return list.iterator();
        })
```

## 陷阱2: 更改静态域导致结果不一致
Java 的序列化结果中，只包括类的实例域部分，静态域在恢复实例时是由本地的 JVM 负责创建的，所以，假如在 Driver 端更改了静态域，而在 Driver 端是看不到的。所以要在 Executor 端使用的静态域，就不要在 Driver端更改，这和`Broadcast`创建后不要更改的要求是类似的。由于出现这种问题一般不会报异常，只会体现在结果中，所以比较难以发现。
```java
package test;
import ...
/**
 * Created by PerfectDay20.
 */
public class Main {
    private static final Logger LOG = LoggerFactory.getLogger(Main.class);
    private static String word = "hello";
    public static void main(String[] args) {
        SparkConf conf = new SparkConf().setAppName("test");
        JavaSparkContext javaSparkContext = new JavaSparkContext(conf);
        JavaRDD<Integer> rdd =
                javaSparkContext.parallelize(IntStream.range(1, 10000).boxed().collect(Collectors.toList()), 10);
        word = "world";
        rdd.foreach(i -> LOG.info(word));
    }
}
```

上面的例子中，`word`初始化为`"hello"`，在 Driver 端的`main`方法中修改为`"world"`，但该值并没有序列化到 Executor 端，Executor 本地仍然是`"hello"`，输出的 log 结果自然也全都是 `"hello"`。

解决方案：
1. 最好一次性初始化好静态域，修饰为`final` ，避免二次更改
2. 在 Executor 端修改静态域，如
```java
        rdd.foreach(i -> {
            word = "world";
            LOG.info(word);
        });
```

假如要在 Executor 端使用一个大的对象，比如一个`Map`，最好的方法还是利用`Broadcast`。

此外，由于多个 task 可能在同一 JVM 中运行，使用静态域可能会导致多线程问题，这也是需要注意的地方。

参考链接：

[Spark Code Analysis][1]

[Understanding Spark Serialization][2]


  [1]: http://bytepadding.com/big-data/spark/spark-code-analysis/
  [2]: http://bytepadding.com/big-data/spark/understanding-spark-serialization/