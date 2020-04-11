---
layout: single
title:  "Split in Java"
date:   2017-11-18 22:35:05 +0800
tags: Java
---

在 Java 中处理字符串时，`split` 是一个很常用的操作，但是这一简单的操作，却经常有意想不到的结果，就拿Guava库官方教程中的一个例子来说，`",a,,b,".split(",")` 的结果是？
```
1. "", "a", "", "b", ""
2. null, "a", null, "b", null
3. "a", null, "b"
4. "a", "b"
5. None of the above
```

正确答案应该是 5，以上都不对；正确结果是 `["", "a", "", "b"]`。

正是因为 JDK 自带的 `split` 这种奇怪的现象，其他开源库也都给出了自己的 `split` 方法，如 Apache Commons Lang 和上文中的 Guava 。

## split in JDK8 ##
`String` 类包含两个 `split` 重载方法，`public String[] split(String regex)` 和 `public String[] split(String regex, int limit)`，调用前者就相当于默认 `limit = 0`，而上面的例子中奇怪的现象就和这个 `limit` 有关。

JDK 文档中是这么解释的：

> 1. 当 `limit` 即 `n` 大于 0 时，会返回至多 `n` 项，最后一项会包含所有未被拆分的部分
> 2. 当 `n` 小于 0 时，会返回所有拆分后的结果
> 3. 当 `n` 等于 0 时，会返回所有拆分后的结果，但是最后跟着的空字符串会被删除

由于使用了单参数的 `split` 方法，`n == 0`，于是就产生了如上的结果。关于这一部分的 JDK 中的源码部分如下：
```java
// Construct result
int resultSize = list.size();
if (limit == 0) {
    while (resultSize > 0 && list.get(resultSize - 1).length() == 0) {
        resultSize--;
    }
}
```

平常在分析一些具有固定格式的数据时，比如每一行都是 `tab` 分割的，且有固定列数，那么进行解析时可以使用 `s.split("\t", -1)` 来进行操作。这样会保存所有的分割项，包含任意部位的空字符串，比如 
```java
":a::b::".split(":", -1) => ["", "a", "", "b", "", ""]
```

另外一个需要注意的地方是，`split` 接收的参数是一个正则表达式，这一点经常容易忽略。比如 `"a.b.c".split(".")` 的结果是 `[]`，长度为 0 ，因为首先 `.` 匹配任意字符，所以原字符串中每一个都是分割符，这就产生了 6 个空字符串， 然后 `limit` 默认为 0 ，从后往前删除空字符串，结果就为空。

## split in Commons Lang ##
JDK 中的方法毕竟还是简单了一些，不能满足我们一些特殊需求，或者说不想使用正则，那么可以使用 Commons Lang 库中的方法。这些 `split` 方法有以下特点：
1. 如果没有指定结果个数，都默认输出最多项
2. 如果没有 `PreserveAllTokens` 后缀，默认将多个连续分割符视为 1 个，不保留任意位置空字符串
比如：
```java
StringUtils.split("::a::b::", ":") => ["a", "b"]
```
 

需要注意的是 `split(String str, String separatorChars)` 方法中第二个参数的意义是每一个字符都被当成分割符，比如：
```java
StringUtils.split(":a:b:", "ab") => [":", ":", ":"]
```
那么假如我想用 `"ab"` 整体作为分割符呢，可以使用 `splitByWholeSeparator` 方法：
```java
StringUtils.splitByWholeSeparator("abcabc","ab") => ["c", "c"]
```
但这个方法有一个和其他方法表现不一致的地方，它保留了末尾的空字符串，且只保留一个。
```java
StringUtils.splitByWholeSeparator("abb", "bb") => ["a", ""]
StringUtils.splitByWholeSeparator("bba", "bb") => ["a"]
StringUtils.splitByWholeSeparator("abbbbabbbb", "bb") =>["a", "a", ""]
```

另外一个我觉得很有用的就是一系列 `splitPreserveAllTokens` 重载函数了，因为默认输出所有结果，且保留了空字符串。和 JDK 中的 `limit = -1` 结果一致，但更易读一些。

## split in Guava ##
假如你已经被上面这些特殊情况都绕晕了，不妨试试 Guava 库，它没有提供简单的一系列重载 `split` 方法，而是提供了一系列的工厂方法，采用链式调用，从而从方法名上就能看出结果，不用苦思冥想到底有没有陷阱。
```java
Splitter.on(",")
        .trimResults(CharMatcher.is(','))
        .omitEmptyStrings()
        .limit(2)
        .split("a,,,,,b,,,,c,,,")
       
=> ["a", "b,,,,c"]
```
除了按照分割符外，还可以按照长度：
```java
Splitter.fixedLength(3).split("abcde") => ["abc", "de"]
```
不像 JDK 和 Commons Lang 中的返回数组，Guava 返回 `Iterable` 和 `List`，而且这个 `Iterable` 已经重载了 `toString`，可以方便地进行打印测试。

