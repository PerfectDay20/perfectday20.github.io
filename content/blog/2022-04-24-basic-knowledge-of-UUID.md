+++
title = "Basic knowledge of UUID"
+++

I have already seen “UUID” many times in many different places, like some file system info, network interface info. This time when I'm reading [Cassandra’s doc](https://cassandra.apache.org/doc/latest/cassandra/cql/definitions.html), this word comes again. 

Every time I see it I think it represents a unique ID, maybe just random created string with user defined length. This time with enough curiosity and time to find out what UUID really means, let’s dig in.

The main resource is [Wikipedia](https://en.wikipedia.org/wiki/Universally_unique_identifier).

As it turns out, UUID(universally unique Identifier) has strictly defined standard, just like IPv4. It’s 128 bit long, with format of `xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx`, `M` and `N` represent different versions and variants. For example, `123e4567-e89b-12d3-a456-426614174000`.

Not all UUIDs are randomly created. Because it has so many bits, it can encode many different info to discriminate UUIDs created by different nodes to minimize the possibility of collision.

- Version1 is based on time and MAC address.
- Version4 is random created.
- Version3/5 are based on MD5/SHA-1.

Different bit length of some well-known protocols:

| IPv4 | 32bit |
| --- | --- |
| IPv6 | 128bit = 16byte |
| UUID | 128bit |
| MD5 | 128bit |
| SHA-1 | 160bit = 20byte |

One of the best part of this wiki is the explanation of collision: 

> For example, the number of random version-4 UUIDs which need to be generated in order to have a 50% probability of at least one collision is 2.71 quintillion...
This number is equivalent to generating 1 billion UUIDs per second for about 85 years. A file containing this many UUIDs, at 16 bytes per UUID, would be about 45 [exabytes](https://en.wikipedia.org/wiki/Exabyte).

1 exabyte (EB) = 1000 PB

This is much more intuitive than just giving a number.

In JDK11, UUID class is implemented as 2 `long`. Use `UUID.randomUUID()` to create a version4 UUID, `UUID.nameUUIDFromBytes()` to create a version3 UUID.
