---
title: 20180827 Lambda & Effective Final
date: 2018-08-28 09:00:00
tags: Java
---
>Inner classes can refer to mutable instance fields 
if the inner-class expression occurs in an instance context, 
but this is not the same thing. 
A good way to think of this is that a reference inside an inner class to the field x
from the enclosing class is really shorthand for Outer.this.x, 
where Outer.this is an implicitly declared final local variable.


[Language designer's notebook: First, do no harm](https://www.ibm.com/developerworks/java/library/j-ldn2/j-ldn2-pdf.pdf)

1. 局部变量的的寿命和包围的 block 是相同的，但 lambda 可以被保存在变量中，进而可以在 block 之外运行。当被 lambda 捕获时，则扩大了局部变量的寿命，与局部变量原有的表现不同
2. 局部变量不会有竞争出现，因为只用一个线程可以访问，如果允许捕获可变局部变量，将打破这一局面
3. lambda 可以捕获 field，没有打破之前的规则
4. 只能捕获 effective final 的一个好的副作用是，引导写出更加函数式的代码