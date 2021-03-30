---
title: 20210330 A Modified ClickHouse Writer for DataX
date: 2021-03-30 11:08:18
tags: Java
---

[DataX](https://github.com/alibaba/DataX) is a framework to transfer data between different databases. It gains most of the power from plugins. Each plugin corresponds to a reader or writer for a database. For example, with a MongoDB reader and a ClickHouse writer, we can use DataX to transfer/sync data from MongoDB to ClickHouse.

From the commit history, I think DataX is not actively developed now, and the ClickHouse writer plugin is a very naive/simple implementation. It has a few drawbacks:

- Each write will create a new database connection
- It extends `CommonRdbmsWriter.Task`, so when batch write fails, it roll back to single record writes. This behavior is against ClickHouse's design
- It will write null to ClickHouse (ClickHouse doesn't recommends to use null)

To make it suitable for my needs, I made a few changes:

- Copy `CommonRdbmsWriter.Task` into `ClickHouseWriter`, rename it to `ClickHouseTask`, delete some unused database types in code
- Remove rollback logic, when batch write fails, it aborts
- Add a new config `fillNullWithDefault` for writing default value (0 for int ...), not default null
- Add a connection pool

The data conversion logic is not changed.

<details>
  <summary>Click to expand the code</summary>
{% gist 18a28346b5632ac1c913aa83d5df8451 %}
</details>


