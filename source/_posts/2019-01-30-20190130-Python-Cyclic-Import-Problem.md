---
title: 20190130 Python Cyclic Import Problem
date: 2019-01-30 18:53:12
tags: Python
---
## 代码示例

```python
# p1.py
import p2
v1 = 1
v2 = p2.v2

#p2.py
import p1
v2 = 2
v1 = p1.v1
```

假如运行 p1.py 会报错 `AttributeError: module 'p2' has no attribute 'v2'`, 
运行 p2.py 会报错 `AttributeError: module 'p1' has no attribute 'v1'`

## import 是如何执行的
当遇到 import moduleA 时, 
1. 首先检查 sys.modules 中有没有 `key=moduleA` 且值不为 None 的条目
2. 如果有则直接使用该条目对应的 module object, 直接执行下一行
3. 如果没有, 则创建 `key=moduleA` 条目, 值为空的 module object, 并执行 moduleA 中的代码, 如果执行中遇到 import 语句, 转到步骤 1 

对于上面的示例代码中, 运行 p1.py 时流程如下:
1. 在 sys.modules 生成 `key='__main__'` 条目
2. 执行 p1.py 的 `import p2`, 这时 sys.modules 中没有 `key=='p2'`, 转到 p2.py 
3. 在 sys.modules 中生成 `key='p2'` 条目, 并执行 p2.py 的 `import p1`, 这时 sys.modules 中没有 `key=='p1'`, 重新执行 p1.py
4. 在 sys.modules 中生成 `key='p1'` 条目, 并执行 p1.py 的 `import p2`, 这时 `key=='p2'` 已经存在, 不用转到 p2.py 执行了, 接着执行 `v1 = 1`, 无异常; 执行 `v2 = p2.v2`, 此时 p2 内容为空(因为代码还没有执行到其他行, `v2` 还没有加入到 p2 的命名空间中), 无法解析 `v2`, 报错

究其原因是 import/class/def 等都是在 import 过程中直接执行的, 这时候遇到没有解析的对象找不到就会报错; 而方法体只有在调用的时候才会被执行

还有一种容易产生问题的写法, 就是循环引用又使用了 `from moduleA import B`, 且 B 是方法/类/变量时, 这就要求在执行到这一句的时候, moduleA 就已经解析了 B; 如下示例
```python
# p1.py
from p2 import f2
def f1():
    f2()
    
# p2.py
from p1 import f1
def f2():
    f1()
```


## 如何解决
一般这种现象的原因都是模块设计不合理, 最好是直接将公共依赖提取出来

临时解决方案包括在方法中 import, 或将 import 语句放在文件末尾

避免使用 import 直接引用类/方法/变量, [Google Python Style Guide](https://github.com/google/styleguide/blob/gh-pages/pyguide.md#22-imports) 提到:
>Use import statements for packages and modules only, not for individual classes or functions.


## 为什么 Java/Scala 中没有这种问题?
归根结底是加载方式不一样. 

在 Java 中使用一个类需要经过 加载->验证->准备->解析->初始化, 如果 Java 文件没有问题, 且在方法区生成了相应类的描述, 此时该类所有的引用都还是符号引用, 只需在初始化之前解析为直接引用即可, 只有真正不存在这个符号引用的对象才会出错, 而不会像 Python 中执行到这一行一定要找到该对象, 即使确实存在也因为没有解析而报错.

另外 Java 中没有像 Python 中这样的全局变量/方法.


---
### 参考:
https://docs.python.org/3/reference/import.html
http://effbot.org/zone/import-confusion.htm
https://realpython.com/absolute-vs-relative-python-imports/
https://stackoverflow.com/questions/744373/circular-or-cyclic-imports-in-python
https://www.zhihu.com/question/19887316