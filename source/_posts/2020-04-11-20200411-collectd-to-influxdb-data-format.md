---
title: 20200411 collectd to InfluxDB data format
date: 2020-04-11 10:17:33
tags:
---
![](/images/20200411/collectd_influxdb.png)

InfluxDB 的数据格式是:
```
measurement, tag_key:tag_value..., field_key:field_value..., timestamp
```
collectd exec 的数据格式是:
```
PUTVAL Identifier [OptionList] Valuelist
```
其中 Identifier 的格式是:
```
host/plugin-plugin_instance/type-type_instance
```
InfluxDB 可以直接通过UDP读取 collectd network plugin 发送的数据, 并且将格式转换为自己的, 源码 [https://github.com/influxdata/influxdb/blob/1.7/services/collectd/service.go](https://github.com/influxdata/influxdb/blob/1.7/services/collectd/service.go) (默认是 split 转换)

转换过程中需要参照 collectd 的 types.db 的类型定义
```
host → tag: host=[host]

plugin → measurement first part

plugin_instance → tag: instance=[plugin_instance]

type → tag: type=[type]

type_instance → tag: type_instance=[type_instance]
```
而measurement的第二部分就是由 types.db 中对应type的name来获得的

OptionList现在没有用到

比如一个 `PUTVAL [example.com/a-b/memory-d](http://example.com/a-b/memory-d) N:123`
经过对比 types.db 中 memory类型: `memory value:GAUGE:0:281474976710656`
就会转换为:
```
measurement: a_value (下划线后面的"value", 就对应的memory类型中值的name)
tags: host=example.com, instance=b, type=memory, type_instance=d
fields: value=123
time: insert time (因为使用的是"N")
```
对于types.db 中有多个值的类型, 比如 `load shortterm:GAUGE:0:5000, midterm:GAUGE:0:5000, longterm:GAUGE:0:5000`
比如一个 `PUTVAL [example.com/a-b/load-d](http://example.com/a-b/memory-d) N:1:U`
就会转换为3个measurement, 名字的第二部分就对应type中的各个部分
```
    > select * from a_shortterm
    name: a_shortterm
    time			host		instance	type	type_instance	value
    ----			----		--------	----	-------------	-----
    1584690315135039000	example.com	b		load	d		1
    
    > select * from a_midterm
    > select * from a_longterm
    name: a_longterm
    time			host		instance	type	type_instance	value
    ----			----		--------	----	-------------	-----
    1584690315135039000	example.com	b		load	d		0
```
其中因为我们传递第一个值为1, 如实记录, 第二个是U, 就省略不写入(只有GAUGE类型是这样, 其他类型会报错), 第三个值没有写, 所以就是0

# Tips:
使用 `collectdctl` 可以方便地进行 putval 测试, 使用时需要打开 collectd unixsock plugin

# 参考:
[https://docs.influxdata.com/influxdb/v1.7/supported_protocols/collectd/](https://docs.influxdata.com/influxdb/v1.7/supported_protocols/collectd/)
[https://collectd.org/documentation/manpages/collectd-exec.5.shtml](https://collectd.org/documentation/manpages/collectd-exec.5.shtml)
[https://linux.die.net/man/1/rrdcreate](https://linux.die.net/man/1/rrdcreate)
[https://collectd.org/wiki/index.php/Plugin:Exec](https://collectd.org/wiki/index.php/Plugin:Exec)
[https://collectd.org/wiki/index.php/Plain_text_protocol](https://collectd.org/wiki/index.php/Plain_text_protocol)
[https://segmentfault.com/a/1190000012993990](https://segmentfault.com/a/1190000012993990)