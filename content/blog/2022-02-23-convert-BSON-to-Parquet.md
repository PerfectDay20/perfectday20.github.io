+++
title =  "Convert BSON to Parquet"
[taxonomies]
tags = ["Spark"]
+++

MongoDB is an OLTP database, when using it as an OLAP for huge data, it’ll take a long time to finish the job. It’s not the MongoDB’s fault, because it's not what it is designed for. In this case, we want to make the data available in other OLAP, such as Hive, Presto, Athena.

MongoDB stores data in BSON, which is the binary representation of JSON, so typically it contains many nested structures. There are two choices for the modern columnar format, Parquet and ORC. Since Parquet uses Dremel algorithm to handle the nested structure, Parquet is the most suitable choice.

Parquet is deeply rooted in Hadoop ecosystem. Using other tools(like Hive, Spark) to create parquet files is simple.  But writing Parquet without Hadoop dependency is nearly impossible. Luckily there are some third format converters provided in their [Github Repository](https://github.com/apache/parquet-mr).

# Avro

We can create the data schema with Avro in JSON, compile the schema to Java classes, then use the `AvroParquetWriter` as below:

```java
ParquetWriter<Struct> writer = AvroParquetWriter.<Struct>builder(new Path("avro.parquet"))
                .withSchema(Struct.getClassSchema())
                .build();
```

This works.

Presto and Hive can read the data. One drawback of this approach is that the schema is very verbose, especially for deeply nested data type:

```json
{"namespace": "com.example.avro",
  "type": "record",
  "name": "Struct",
  "fields": [
    {"name": "_id", "type": {
      "namespace": "com.example.avro",
      "type": "record",
      "name": "ID",
      "fields": [
        {"name": "a", "type": ["null","long"]},
        {"name": "b", "type": ["null","int"]},
        {"name": "c", "type": ["null","int"]}
      ]
    }},
    {"name": "value", "type": ["null", {
      "namespace": "com.example.avro",
      "type": "record",
      "name": "Value",
      "fields": [
        {"name": "d", "type": ["null","long"]},
        {"name": "e", "type": ["null","long"]},
        {"name": "f", "type": ["null", {"type": "map", "values": ["null", "long"]}]},
        {"name": "g", "type": ["null", {"type": "map", "values": ["null", {
          "namespace": "com.example.avro",
          "type": "record",
          "name": "r",
          "fields": [
            {"name": "h", "type": ["null","long"]},
            {"name": "i", "type": ["null","long"]}
          ]
        }]}]}
      ]
    }]}
  ]
}
```

# Protobuf

Another way is using protobuf converter. The mechanism is same as Avro: define the schema, compile to Java source code, create the converter:

```java
ParquetWriter<MyStruct> parquetWriter = new ParquetWriter<>(
        new Path("test.parquet"),
        new ProtoWriteSupport<>(MyStruct.class)
);
```

This time, the schema is very brief:

```proto
message MyStruct {
    Id _id = 1;
    Value value = 2;
}
 
message Id {
    int64 a = 1;
    int32 b = 2;
    int32 c = 3;
}
 
message Value {
    int64 d = 1;
    int64 e = 2;
    map<string, int64> f = 3;
    map<string, Cr> g = 4;
 
    message Cr {
        int64 h = 1;
        int64 i = 2;
    }
}
```

But in my test, the nested part create by the converter is not in compliance with [Parquet specification](https://github.com/apache/parquet-format/blob/master/LogicalTypes.md#maps). So Hive&Presto just doesn’t recognize them.

The difference can be shown by parquet-tools to parse the schema:

```java
// expected map structure
optional group f (MAP) {
      repeated group key_value (MAP_KEY_VALUE) {
        required binary key (STRING);
        optional int64 value;
      }
    }

// created map structure
repeated group f = 3 {
      optional binary key (STRING) = 1;
      optional int64 value = 2;
    }
```

# Flink

Flink can handle both unbounded and bounded dataset, it’s a widely used BigData tool, so I assume it must have an easy-to-use Sink to write Parquet files, right?

But to my surprise, there is no such Sink. All the examples I found are using Avro converters.

# Spark

So finally, the Spark.

Using Spark has many advantages:

- The native interface to write Parquet and ORC (native API of Spark, not in the sense of JNI)
- No need to define schema with other formats(Protobuf, Avro, or self-invented), POJO Bean is enough
- Parallel processing is just a few lines of code

```java
@Data
public class Struct {
    ID _id; // Same name in mongo
    Value value;
    int pdate;
 
    @Data
    public static class ID {
        Long a;
        Integer b;
        Integer c;
    }
 
    @Data
    public static class Value {
        Long d;
        Long e;
        Map<String, Long> f;
        Map<String, Cr> g;
    }
 
    @Data
    public static class Cr {
        Long h;
        Long i;
    }
}
 
    public static void main(String[] args) throws IOException {
        SparkSession spark = SparkSession.builder().getOrCreate();
 
        Dataset<Struct> dataset = spark.createDataset(Arrays.asList(createEmptyStruct(), createEmptyStruct()),
                Encoders.bean(Struct.class));
 
        dataset.repartition(1).write()
                .option("compression", "gzip")
                .mode(SaveMode.Overwrite)
                .partitionBy("pdate")
                .parquet("parquet");
    }
```

One caveat of using `Encoders.bean()` to create the schema: the fields are sorted by name, because it uses `TreeMap` to parse the class. So in schema evolution, the new columns are not always append to the last.
