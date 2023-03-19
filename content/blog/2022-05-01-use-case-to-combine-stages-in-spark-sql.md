+++
title = "Use CASE to combine stages in Spark SQL"
[taxonomies]
tags = ["Spark"]
+++

Suppose you have a table containing a column whose type is a map, then you want to count the number of empty and non-empty collections. 

A straightforward way would be:
```SQL
SELECT COUNT(*) FROM some_table WHERE size(map_col) > 0;
SELECT COUNT(*) FROM some_table WHERE size(map_col) <= 0 or size(map_col) IS NULL;
```
This is easy. But Spark will create two stages, scan `some_table` twice, although we human knows one single pass can compute all the results we want, Spark isn't smart enough to do that, or we human didn't make Spark smart enough.

If the table is small, then scanning a table twice won't do any harm, the SQL query is easy to understand, this is even the recommended way. But if the table is derived from another table, very large, and column is nested, then scanning only once is more attracting.

A small trick to do this is using `CASE WHEN`:
```SQL
SELECT SUM(CASE WHEN size(map_col) > 0 THEN 1 ELSE 0 END) as non_empty_count, 
       SUM(CASE WHEN size(map_col) > 0 THEN 0 ELSE 1 END) as empty_count 
FROM some_table;
```

This is not a Spark trick, but a SQL trick, so any SQL engine can benefit from it.
