---
title: 20180611 Trait Linearization
date: 2018-06-12 07:49:46
tags: Scala
---
1. Scala 中 trait linearization 可以这样理解，`class Cat extends Animal with Furry with FourLegged` 中 `extends` 后面的 `Animal with Furry with FourLegged` 是一个列表，而 `Cat` 将要跟在这个列表后面，如 Animal <- Furry <- FourLegged <- Cat，把这个列表“当成”父类到子类的列表，所以调用后面任意一个trait或类中的方法，都会通过 `super` 方法向左传递，直到找到一个有实际实现的方法体。而初始化是从左到右，符合从父类到子类的规则
```Scala
  trait Animal {
    def desc: String = "It's an animal"
  }
  trait Furry extends Animal {
    override def desc: String = super.desc + ", and it's furry"
  }
  trait HasLegs extends Animal {
    override def desc: String = super.desc + ", and it has legs"
  }
  trait FourLegged extends HasLegs {
    override def desc: String = super.desc + ", and the number of legs is four"
  }
  class Cat extends Animal with Furry with FourLegged{
    override def desc: String = super.desc +", and it's a cat!"
  }

  val cat = new Cat
  println(cat.desc) // It's an animal, and it's furry, and it has legs, and the number of legs is four, and it's a cat!
```