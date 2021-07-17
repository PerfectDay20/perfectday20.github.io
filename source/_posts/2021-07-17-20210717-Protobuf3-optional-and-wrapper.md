---
title: 20210717 Protobuf3 optional and wrapper
date: 2021-07-17 11:40:16
tags: Java
---



Protobuf 3 didn't have `optional`, and scalar types didn't have `hasXXX()` methods, so it wasn't convenient to test whether a scalar field was set or not.

One way to solve this is to use wrapper type as in `import "google/protobuf/wrappers.proto";`.

But since 3.12 (experimental) and 3.15 (formal), Protobuf finally add `optional` back, so which one should we use, wrapper or `optional`?

After some simple tests, `optional` is the better choice when you need to use `hasXXX()`: API is simpler than wrappers, and serialized size is smaller.

----

## Test details:

|                                                           | Scalar int32 | Wrapper int32                                                | Optional int32 |
| --------------------------------------------------------- | ------------ | ------------------------------------------------------------ | -------------- |
| has `hasXXX()` method                                     | no           | yes                                                          | yes            |
| if field not set, no data will be serialized              | yes          | yes                                                          | yes            |
| if field set dafault value(0), no data will be serialized | yes          | no                                                           | no             |
| If field is set, total serialized size                    | small        | large(because wrapper is embeded message, and is treated as string, has length field) | small          |



```protobuf
syntax = "proto3";

option java_package = "test";
option java_outer_classname = "TestProto";
import "google/protobuf/wrappers.proto";


message ScalarInt{
    int32 a = 1;
}

message OptionalInt {
    optional int32 a = 1;
}

message WrapperInt {
    google.protobuf.Int32Value a = 1;
}

```



```java
package test;

import com.google.protobuf.Int32Value;

public class Main {
    public static void main(String[] args) throws Exception {
        testScalarInt(0);
        testOptionalInt(0);
        testWrapperInt(0);
        testScalarInt(150);
        testOptionalInt(150);
        testWrapperInt(150);
    }

    private static void testScalarInt(int n){
        TestProto.ScalarInt scalarInt = TestProto.ScalarInt.newBuilder().setA(n).build();
        // scalarInt.hasA();
        printInfo(scalarInt.toByteArray(), "ScalarInt " + n);
    }

    private static void testOptionalInt(int n){
        TestProto.OptionalInt optionalInt = TestProto.OptionalInt.newBuilder().setA(n).build();
        optionalInt.hasA();
        printInfo(optionalInt.toByteArray(), "OptionalInt " + n);
    }

    private static void testWrapperInt(int n){
        TestProto.WrapperInt wrapperInt = TestProto.WrapperInt.newBuilder().setA(Int32Value.of(n)).build();
        wrapperInt.hasA();
        printInfo(wrapperInt.toByteArray(), "wrapperInt " + n);
    }

    private static void printInfo(byte[] bytes, String type) {
        StringBuilder sb = new StringBuilder(type)
                .append(". Size: ")
                .append(bytes.length)
                .append(". Data: ");
        for (byte b : bytes) {
            sb.append(String.format("%08d ", Integer.parseInt(Integer.toBinaryString(b&0xff))));
        }
        System.out.println(sb);
    }
}

```



Output:

```
ScalarInt null. Size: 0. Data: 
OptionalInt null. Size: 0. Data: 
WrapperInt null. Size: 0. Data: 

ScalarInt 0. Size: 0. Data: 
OptionalInt 0. Size: 2. Data: 00001000 00000000 
WrapperInt 0. Size: 2. Data: 00001010 00000000 

ScalarInt 150. Size: 3. Data: 00001000 10010110 00000001 
OptionalInt 150. Size: 3. Data: 00001000 10010110 00000001 
WrapperInt 150. Size: 5. Data: 00001010 00000011 00001000 10010110 00000001 
```

To decode the bytes result, refer to: https://developers.google.com/protocol-buffers/docs/encoding

- When field not set: nothing is serialized

- When set to 0:
  - `OptionalInt`: `00001`  = field number, `000` = wire type varint, `00000000` = varint value 0 (this is the 0 we set to `a`).
  - `WrapperInt`: `00001` = field number, `010` = wire type length-delimited, `00000000` = varint length 0 (not value of `a`, `a` is default value 0, so is not serialized)
- When set to 150:
  - `ScalarInt`: `00001` = field number, `000` = wire type varint, `10010110 00000001` = varint 150
  - `OptionalInt`: same as `ScalarInt`
  - `WrapperInt`: `00001` = field number, `010` = wire type length-delimited, `00000011` = varint length 3, `00001000 10010110 00000001` = full data same as `ScalaInt`



