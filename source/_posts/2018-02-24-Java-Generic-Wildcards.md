---
layout: single
title:  "一图看懂Java泛型通配符"
date:   2018-02-22 22:40:05 +0800
tags: Java
---

![](/images/20180224/generic.jpg)

当使用 `<? super MyClass>` 的时候，表明未知类的继承结构处于 `Object` 和 `MyClass` 之间，这时
- 编译器只能确定任何返回该未知类型的方法，返回的变量都是 `Object` 的子类，所以返回的类型就确定为 `Object`，比如 `getter` 方法
- 使用该未知类型作为参数的方法，该参数一定是 `MyClass` 的父类，所以可以传递 `MyClass` 及其子类进去，比如 `setter` 方法

而使用 `<? extends MyClass>` 的时候，未知类型一定是 `MyClass` 的子类，但向下延伸到无穷尽，无法判断
- 所以返回未知类型的方法的返回类型有一个上界，就是 `MyClass`，即返回类型确定为 `MyClass`
- 但是使用未知类型的方法，因为向下继承无限延伸，无法判断下界，所以不能使用该方法，比如 `setter`(可以 `set(null)`)

使用 `<?>` 的时候，可以当作 `<? extends Object>`，即上界是 `Object`，可以使用 `getter` 方法，不可以使用 `setter` 方法。

根据上面这些原则，一个简单的例子如下：

```java
@Data // lombok，省略了 getter 和 setter
class Holder<T>{
    private T t;

    public <U extends MyClass> void testSetter(Holder<? super MyClass> holder, U u) {
        holder.setT(u); // 可以输入任何 MyClass 及子类的对象
        holder.setT(null);
    }

    public  void testGetter1(Holder<? extends MyClass> holder) {
        MyClass obj = holder.getT(); // 能确定返回的对象一定是 MyClass 或父类的对象
    }

    public void testGetter2(Holder<?> holder) {
        Object obj = holder.getT(); // 只能确定返回的对象一定是 Object
    }
}

class MyClass{}
```
选择限定通配符时的快速判断方法：
>get-put principle:
Use an extends wildcard when you only get values out of a structure, use a super wildcard when you only put values into a structure, and don't use a wildcard when you do both.

参考：
[https://www.ibm.com/developerworks/java/library/j-jtp07018/index.html](https://www.ibm.com/developerworks/java/library/j-jtp07018/index.html)