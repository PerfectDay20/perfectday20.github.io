---
title: 20180821 Spark & Scala Lambda Serialization
date: 2018-08-22 08:33:37
tags:
- Spark
- Scala
- Java
---
 [Lambda serialization](https://www.lyh.me/lambda-serialization.html)
这篇文章的思路是通过加上 @transient 来使得不用的 field 避免序列化，副作用就是假如用到了，就会出现 NPE，解决办法就是再加上 lazy

---
[Hygienic Closures for Scala Function Serialization](http://erikerlandson.github.io/blog/2015/03/31/hygienic-closures-for-scala-function-serialization/)
其实思路和 [Spark RDD Programming Guide](http://spark.apache.org/docs/latest/rdd-programming-guide.html#passing-functions-to-spark) 中的一样，都是先将需要对象中的 field 复制到 local variable 中，这样就避免了序列化整个对象
不过该文中实现了一个更通用的方法来包装整个过程，更方便使用

---
在 Spark 传递 functions 时，
- Scala 建议使用 [Anonymous function syntax](https://docs.scala-lang.org/tour/basics.html#functions) 和 Object 中的 static method；当传递类实例中的方法时，就需要注意序列化问题；可以用 anonymous function 是因为其被编译后的代码实现了序列化接口
- Java 建议时使用对应的 lambda 用来实现 Spark 提供的各种继承序列化接口的 Function 接口；或者写出继承类；**这里需要注意，如果使用匿名类或内部类，由于其拥有外部对象的引用，所以外部类必须实现序列化，或者用 static 内部类；但 non-capturing lambda 是被编译成外部类的 static field，所以可以直接使用，外部类也不用序列化（在 Scala 2.12, 2.13 中所有的 lambda 都被编译成 static 方法）**
>Compiler by default inserts constructor in the byte code of the
 Anonymous class with reference to Outer class object .
The outer class object is used to access the instance variable
The outer class is serialized and sent along with the serialized object of the inner anonymous class

[understanding-spark-serialization](https://stackoverflow.com/questions/40818001/understanding-spark-serialization)

>Non-capturing lambdas are simply desugared into a static method having exactly the same signature of the lambda expression and declared inside the same class where the lambda expression is used. 

[Java 8 Lambdas - A Peek Under the Hood
](https://www.infoq.com/articles/Java-8-Lambdas-A-Peek-Under-the-Hood)

>Note that lambda body methods are [always static](https://github.com/scala/scala/commit/0533a3df71) (in Scala 2.12, 2.13). If a closure captures the enclosing instance, the static body method has an additional `$this` parameter. The Java 8 compiler handles this differently, it generates instance methods if the lambda accesses state from the enclosing object.

[Remaining issues for Spark on 2.12](https://docs.google.com/document/d/1fbkjEL878witxVQpOCbjlvOvadHtVjYXeB-2mgzDTvk/edit#)

- Python 同样是 lambda，但多了 local defs 和 top-level functions

---
值得一读：
[Understanding Spark Serialization](http://bytepadding.com/big-data/spark/understanding-spark-serialization/)
[
What Lambda Expressions are compiled to? What is their runtime behavior?](https://www.logicbig.com/tutorials/core-java-tutorial/java-8-enhancements/java-lambda-functional-aspect.html)