+++
title = "Spark mapPartitions and Iterator"
[taxonomies]
tags = ["Spark", "Scala", "Java"]
+++

In Spark, `mapPartitions` is a good alternative of `map` if you need to do some heavy initialization for some processing and only want to init it only once for each partition, not each record.

But there exists a small detail, as shown in the method signature:
```scala
def mapPartitions[U : Encoder](func: Iterator[T] => Iterator[U]): Dataset[U]
```
This method takes a function which convert an iterator to another.

In Scala, iterator is versatile enough to have many transform methods, such as `map`, `flatMap`, `filter`. 
An important aspect of these methods is that they all will only create a lazy view on the iterator, instead of creating an intermediate collection to hold the transformed data.
So these methods make the processing both smooth and memory efficient.
Most of the time the Spark user may not even notice this lazy view characteristic.

But in Java, iterator is not that powerful. Spark provides a `mapPartition` in Java version:
```Scala
def mapPartitions[U](f: MapPartitionsFunction[T, U], encoder: Encoder[U]): Dataset[U]
```
Then this `MapPartitionsFunction` also takes an iterator and return an iterator.
```Java
public interface MapPartitionsFunction<T, U> extends Serializable {
  Iterator<U> call(Iterator<T> input) throws Exception;
}
```
Java's iterator doesn't have any of `map`, `flatMap`, `filter`. So for new Java programmers in Spark world, they may first convert the iterator to a collection, do the transform, the return the collection's iterator.
This works good on small dataset, but in large dataset, this may cause OOM.

Why? Because `mapPartitions` will use the input iterator to process each record of a partition in memory, if you turn this iterator into a collection, then the whole partition needs to be hold by the heap, which may cause the trouble.

A possible solution is using `org.apache.commons.collections4.IteratorUtils`, such as `transformedIterator`, `filteredIterator`. Alternatively, you can write your own lazy transform iterator wrapper.

---
So what's the takeaway from this issue? 
- For API users, it's not enough to use a method correctly solely relying on the method signature. You must understand the underlying logic.
- For API writers, providing same signature for different languages and retaining the same accessibility can be challenging.