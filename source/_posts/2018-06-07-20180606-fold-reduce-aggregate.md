---
title: 20180606 fold reduce aggregate
date: 2018-06-07 08:10:34
tags: Scala
---
1. fold, reduce, aggregate
```scala
fold // 需要初始参数，运算符参数必须为集合元素父类
foldLeft // 区别是运算符参数不需要是集合元素父类
foldRight // 同上

reduce // 不需要初始参数，因此集合为空会报错，运算符参数必须为集合元素父类
reduceLeft 
reduceRight

reduceOption // 同reduce，但返回Option，集合为空不报错返回None
reduceLeftOption
reduceRightOption

aggregate // 需要初始参数，聚合结果与集合元素运算符，聚合结果之间的运算符

// 可以将集合元素聚合为非父类结果的操作：foldLeft, foldRight, aggregate
```