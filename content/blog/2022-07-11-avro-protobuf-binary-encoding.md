+++
title = "Binary Encoding Comparison of Avro and Protocol Buffer"
+++

Version used:
[Apache Avro 1.11.1](https://avro.apache.org/docs/1.11.1/specification/)
[Protocol Buffer 3](https://protobuf.dev/programming-guides/proto3/)

| data type      | Avro                                | Protobuf                                                                                                                                           |
| -------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| null           | zero bytes                          |                                                                                                                                                    |
| boolean        | 1 byte                              | as int32, 1 byte                                                                                                                                   |
| int, long      | variable-length zig-zag             | int32(two's complement), sint32(zig-zag), uint32, fixed32, sfixed32                                                                                |
| float          | 4 bytes                             | 4 bytes                                                                                                                                            |
| double         | 8 bytes                             | 8 bytes                                                                                                                                            |
| bytes          | a long + bytes                      | varint + bytes                                                                                                                                     |
| string         | a long + UTF-8                      | varint + UTF-8                                                                                                                                     |
| record/message | encoded fields, no length/separator | [Tag-Length-Value](https://en.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value), varint key: <code>(field_number << 3) &#124; wire_type</code> |
| enum           | as int                              | as int32                                                                                                                                           |
| array/repeated | blocks (a long + items)             | primitive numeric types are packed:  varint + items; other types just repeats                                                                      |
| map            | blocks (a long + items)             | as repeated nested tuple                                                                                                                           |
| fixed          | fixed size bytes                    |                                                                                                                                                    |
| union          | an int (position) + value           |                                                                                                                                                    |

For scalar types, Avro and ProtoBuf use similar encodings;
while for complex types(record/message, array/repeated, map), Avro uses a packed encoding, 
Protobuf just writes the k-v multiple times.

Most of the differences come from this design choice:
- Avro encode every record fields (so no need for field name/index as keys)
- Protobuf only encode non-default fields for `singular` fields, and all fields for `optional` fields.

This make Protobuf more suitable for spare messages.

Other differences:
### Nullable
  
Avro uses `union` to support nullable values.
Then if a type is nullable, every encoded value will contain 1 byte as the type position in
`union`'s schema.

For Protobuf, we can use `optional` to mimic the null value.

### Default values

In Avro, a default value is only used when reading instances that lack the field for schema evolution purposes. 
The presence of a default value does not make the field optional at encoding time. 
Avro encodes a field even if its value is equal to its default.

Protobuf3 only supports default zero-false-empty values.

So if we have a very sparse record/message that defines 100 nullable fields, using `union` in Avro and `optional` in Protobuf,
how an instance is encoded when all fields are null?

Avro: 100 position `union` index, so 100 bytes.

Protobuf: no fields are encoded.

### Submessages

Because Protobuf uses Tag-Length-Value, so nested submessage fields must use the `LEN` wire type
for the parser to know how long the encoded field is. 

While Avro encodes all submessage's fields, no length is needed.


### Field presence
Avro record fields are always present in wire format.
Protobuf has more complicated [field presence](https://protobuf.dev/programming-guides/field_presence/).

|                         | singular   | optional   |
| ----------------------- | ---------- | ---------- |
| not write any value     | not-encode | not-encode |
| write default value     | not-encode | encode     |
| write non-default value | encode     | encode     |

---
So when to use which?
- If most of the record fields are always default or null or empty, use ProtoBuf. 
- If most of the record fields are always present and non-default, use Avro.
- For other cases, test before decide.