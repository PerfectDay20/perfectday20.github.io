---
layout: single
title:  "使用Python和Java调用Shell脚本时的死锁陷阱"
date:   2018-01-21 11:12:05 +0800
tags: 
- Java 
- Python
- Shell
---

最近有一项需求，要定时判断任务执行条件是否满足并触发 Spark 任务，平时编写 Spark 任务时都是封装为一个 Jar 包，然后采用 Shell 脚本形式传入所需参数执行，考虑到本次判断条件逻辑复杂，只用 Shell 脚本完成不利于开发测试，所以调研使用了 Python 和 Java 分别调用 Spark 脚本的方法。

使用版本为 Python 3.6.4 及 JDK 8

## Python
主要使用 subprocess 库。Python 的 API 变动比较频繁，在 3.5 之后新增了 `run` 方法，这大大降低了使用难度和遇见 Bug 的概率。
```python
subprocess.run(["ls", "-l"])
subprocess.run(["sh", "/path/to/your/script.sh", "arg1", "arg2"])
```

为什么说使用 `run` 方法可以降低遇见 Bug 的概率呢？
在没有 `run` 方法之前，我们一般调用其他的高级方法，即 Older high-level API，比如 `call`，`check_all`，或者直接创建 `Popen` 对象。因为默认的输出是 console，这时如果对 API 不熟悉或者没有仔细看 doc，想要等待子进程运行完毕并获取输出，使用了 `stdout = PIPE` 再加上 `wait` 的话，当输出内容很多时会导致 Buffer 写满，进程就一直等待读取，形成死锁。在一次将 Spark 的 log 输出到 console 时，就遇到了这种奇怪的现象，下边的脚本可以模拟：
```shell
# a.sh
for i in {0..9999}; do
    echo '***************************************************'
done 
```
```python
p = subprocess.Popen(['sh', 'a.sh'], stdout=subprocess.PIPE)
p.wait()
```
而 `call` 则在方法内部直接调用了 `wait` 产生相同的效果。
要避免死锁，则必须在 `wait` 方法调用之前自行处理掉输入输出，或者使用推荐的 `communicate` 方法。 `communicate` 方法是在内部生成了读取线程分别读取 `stdout` `stderr`，从而避免了 Buffer 写满。而之前提到的新的 `run` 方法，就是在内部调用了 `communicate`。
```python
stdout, stderr = process.communicate(input, timeout=timeout)
```

## Java
说完了 Python，Java 就简单多了。
Java 一般使用 `Runtime.getRuntime().exec()` 或者 `ProcessBuilder` 调用外部脚本：
```java
Process p = Runtime.getRuntime().exec(new String[]{"ls", "-al"});
Scanner sc = new Scanner(p.getInputStream());
while (sc.hasNextLine()) {
    System.out.println(sc.nextLine());
}
// or
Process p = new ProcessBuilder("sh", "a.sh").start();  
p.waitFor(); // dead lock    
```
需要注意的是，这里 `stream` 的方向是相对于主程序的，所以 `getInputStream()` 就是子进程的输出，而 `getOutputStream()` 是子进程的输入。

基于同样的 Buffer 原因，假如调用了 `waitFor` 方法等待子进程执行完毕而没有及时处理输出的话，就会造成死锁。
由于 Java API 很少变动，所以没有像 Python 那样提供新的 `run` 方法，但是开源社区也给出了自己的方案，如[commons exec](https://commons.apache.org/proper/commons-exec/)，或 [http://www.baeldung.com/run-shell-command-in-java](http://www.baeldung.com/run-shell-command-in-java)，或 alvin alexander 给出的[方案](https://alvinalexander.com/java/java-exec-processbuilder-process-1)（虽然不完整）。
```java
// commons exec，要想获取输出的话，相比 python 来说要复杂一些
CommandLine commandLine = CommandLine.parse("sh a.sh");
        
ByteArrayOutputStream out = new ByteArrayOutputStream();
PumpStreamHandler streamHandler = new PumpStreamHandler(out);
        
Executor executor = new DefaultExecutor();
executor.setStreamHandler(streamHandler);
executor.execute(commandLine);
        
String output = new String(out.toByteArray());
```
但其中的思想和 Python 都是统一的，就是在后台开启新线程读取子进程的输出，防止 Buffer 写满。

另一个统一思想的地方就是，都推荐使用数组或 `list` 将输入的 shell 命令分隔成多段，这样的话就由系统来处理空格等特殊字符问题。


参考：

[https://dcreager.net/2009/08/06/subprocess-communicate-drawbacks/](https://dcreager.net/2009/08/06/subprocess-communicate-drawbacks/)
[https://alvinalexander.com/java/java-exec-processbuilder-process-1](https://alvinalexander.com/java/java-exec-processbuilder-process-1)
[https://www.javaworld.com/article/2071275/core-java/when-runtime-exec---won-t.html](https://www.javaworld.com/article/2071275/core-java/when-runtime-exec---won-t.html)