---
layout: single
title:  "Java对象占用内存计算"
date:   2018-01-19 22:35:05 +0800
tags: Java
---


## 计算前提
1. JDK 版本，不同版本的类可能会有变化
2. 要区分是 32bit 还是 64bit 系统
3. 是否开启压缩指针（默认开启，指针为 4Byte，否则为 8Byte）
4. 是否数组，数组对象头多了一个长度值，占 4Byte

## 计算方法
对象所占内存 = 对象头 + 所有域 + 填充
其中，若域为另一个对象，即非基本类型，则需递归计算

## 对象头
对象头分为3部分：
1. mark word：同步状态、GC状态、hashcode 等
2. klass pointer: 指向本身的类对象
3. 数组类型的长度

| |_mark|_kclass|Array Length|
|--|--|--|--|
|32bit|4|4|4|
|64bit|8|8|4|
|64+comp|8|4|4|

[https://mechanical-sympathy.blogspot.com/2011/07/false-sharing.html](https://mechanical-sympathy.blogspot.com/2011/07/false-sharing.html)
[http://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html](http://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html)

## 不同状态下的对象头
```
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
```
[http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/87ee5ee27509/src/share/vm/oops/markOop.hpp#l29](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/87ee5ee27509/src/share/vm/oops/markOop.hpp#l29)


## 对象的域在内存中的顺序：
域的顺序并不是在类中定义的顺序，而是经过了调整；每个对象都是 8Byte 对齐的，不是倍数的话会在最后填充，具体顺序如下：
1. doubles (8) and longs (8)
2. ints (4) and floats (4)
3. shorts (2) and chars (2)
4. booleans (1) and bytes (1)
5. references (4/8)
6. repeat for sub-class fields

[https://zeroturnaround.com/rebellabs/dangerous-code-how-to-be-unsafe-with-java-classes-objects-in-memory/5/](https://zeroturnaround.com/rebellabs/dangerous-code-how-to-be-unsafe-with-java-classes-objects-in-memory/5/)
## 不同域的大小

| |Bytes|
|--|--|
|boolean|1|
|byte|1|
|char|2|
|short|2|
|int|4|
|float|4|
|long|8|
|double|8|
|reference|4|

## 具体例子
64bit 压缩指针 JDK8 中 `String s = "abc"`，对象 `s` 的大小： 48Bytes
![](/images/20180119/abc.png)

注意：不同版本的 JDK String 类的域不同，比如 JDK6 中有 `offset`、`count`，JDK7 中有 `hash32`。
具体验证可以使用 `jol` 库：
[http://openjdk.java.net/projects/code-tools/jol/](http://openjdk.java.net/projects/code-tools/jol/)
```java
System.out.println(GraphLayout.parseInstance("abc").toPrintable());
==>
java.lang.String@2ff4acd0d object externals:
          ADDRESS       SIZE TYPE             PATH                           VALUE
        795707020         24 java.lang.String                                (object)
        795707038         24 [C               .value                         [a, b, c]
```
