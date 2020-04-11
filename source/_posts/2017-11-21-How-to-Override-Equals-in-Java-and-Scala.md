---
layout: single
title:  "How to Override Equals in Java and Scala"
date:   2017-11-21 22:35:05 +0800
tags: 
- Java 
- Scala
---

相信读过 《Effective Java》 的读者都已经知道编写 `equals` 方法的作用与重要性，基本概念不多做解释，这里就总结一下如何编写正确的 `equals` 方法。

`equals` 在 Java 和 Scala 中含义相同，都需要满足以下五个条件：
1. 自反性
2. 对称性
3. 传递性
4. 一致性
5. `anyObject.equals(null) == false`

现在我们有三个问题：
1. 假如我们只有一个类 `Person`，如何写？
2. 假如 `Person` 类有一个子类 `Student`，相互不能判断（一定返回 `false`），如何写？相互可以判断，如何写？
3. 假如 `Person` 和 `Student` 可以相互判断，但另一子类 `Teacher` 只能和同类判断，如何写？

## Java ##

《Effective Java》 中最后推荐的写法步骤是：
1. 通过 `==` 判断是否是同一个对象
2. 用 `instanceof` 判断是否是正确的类型，注意这里已经包含了 `null` 的情况，所以不用单独另写
3. 将对象转换成正确的类型
4. 对需要判断的域分别进行对比

需要注意，基本类型用 `==` 判断，例外是 `float` 用 `Float.compare`，`double` 用 `Double.compare`，因为有 `NaN` 等特殊值存在。

上述第二步中还有另一个变种，是使用 `getClass` 进行类型判断，这样的话只有类型完全一致才能返回 `true`，如果只是单一的类还好，要是涉及类之间的继承，则违背了 Liskov Substitution Principle，所以最后书中的结论是：

> There is no way to extend an instantiable class and add a value component while preserving the equals contract.


由于现在的 IDE 例如 IntelliJ IDEA 已经可以自动为我们生成 `equals` 方法，还可以选择是否允许子类判断，是否可为 `null` 等判断，所以我们就不必手动编写了，但是生成的结果也是符合上面的 4 步的：

![](/images/20171121/equals.png)
```java
class Person{
    private String name;
    private int age;

    @Override 
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) { // 不涉及继承，问题 1 和 问题 2 前半的写法
            return false;
        }

        Person person = (Person) o;

        if (age != person.age) {
            return false;
        }
        return name != null ? name.equals(person.name) : person.name == null;
    }
    
    @Override 
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof Person)) { // 涉及继承，使得与子类之间也可以判断，问题 2 后半的写法
            return false;
        }

        Person person = (Person) o;

        if (age != person.age) {
            return false;
        }
        return name != null ? name.equals(person.name) : person.name == null;
    }
}

```


## Scala ##
scala 中编写的方式大致相同，但是结合其语法，相比似乎又简单又繁琐。
简单是指当没有子类，或和子类判断一定为 `false` 时（违反LSP），可以这样写：
```scala
class Person(val name: String, val age: Int) { 
  override def equals(other: Any): Boolean = other match { // 问题 1 的写法
    case that: this.getClass == that.getClass &&
                Person => name == that.name && 
                age == that.age
    case _ => false
  }
}
```
繁琐是指假如这时出现了一个子类 `Student` 且增加了一个域 `sid`，假如我们需要两个类可相互判断，则上述方法在判断一个 `Person` 对象和一个 `Student` 对象时一定会返回 `false`。

因此《Programming in Scala》中建议采用如下的编写方式：
```scala
class Person(val name: String, val age: Int) {
  def canEqual(other: Any): Boolean = other.isInstanceOf[Person]

  override def equals(other: Any): Boolean = other match { // 问题 2 的写法
    case that: Person =>
      (that canEqual this) &&
        name == that.name &&
        age == that.age
    case _ => false
  }
}

class Student(override val name: String, override val age: Int, val sid: Int) extends Person(name, age){
}
```
上面 `canEqual` 方法的作用和 Java 代码中判断 `instanceof` 的作用是一致的，但比 Java 中的判断更加灵活，比如可以限定不同子类与父类的判断关系。

比如有一个 `Person` 的子类 `Teacher`，我们希望它只能和 `Teacher` 类进行判断，与 `Person` 和 `Student` 判断都返回 `false`，该如何写呢？一种错误的写法如下：
```scala
class Teacher(override val name: String, override val age: Int, val tid: Int) extends Person(name, age){
  override def equals(other: Any): Boolean = other match {
    case that: Teacher =>
      this.getClass == that.getClass &&
        name == that.name &&
        age == that.age
    case _ => false
  }
}

val s1 = new Student("z", 1, 2)
val t1 = new Teacher("z", 1, 2)
println(s1 == t1) // true
println(t1 == s1) // false 违反了对称性
```
正确的写法应该是：
```scala
class Teacher(override val name: String, override val age: Int, val tid: Int) extends Person(name, age){
  override def canEqual(other: Any): Boolean = other.isInstanceOf[Teacher]

  override def equals(other: Any): Boolean = other match { // 问题 3 的写法
    case that: Teacher =>
      super.equals(that) &&
        (that canEqual this) &&
        name == that.name &&
        age == that.age &&
        tid == that.tid
    case _ => false
  }
}
```
注意只覆盖了 `canEqual` 方法也会违反对称性。在 Java 中要实现相同的效果，则也需要编写类似的 `canEqual` 方法，就留给读者自己考虑了。

总之，在编写单个类的 `equals` 方法时比较简单，当涉及子类继承时，就要多考虑一下了。

另外不要忘记覆盖 `hashcode` 方法哦。
