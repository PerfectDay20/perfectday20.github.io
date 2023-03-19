+++
title = "A Journey of Memcached"
+++

> This is by no means a tutorial or comprehensive introduction to memcached, it's just what I learnt and felt about memcached along the learning journey.

I didn't remember when I first encountered the memcached, but it certainly left me a great impression in [http://boringtechnology.club/](http://boringtechnology.club/), the author's story is fascinating. After reading that story for several years, I finally want to know more about memcached, just for curiousity.

After reading through the [official wiki](https://github.com/memcached/memcached/wiki), I learnt something about memcached:

It really is JUST a cache. No persistence, no replica, no communications among servers, the client chooses which server to set and get values. 

It fulfills Unix philosophy, do one thing and do it well.

# About the protocols

The wiki is very old, so conflicts exist. The server client communication protocol is spread in a few pages:

- [https://github.com/memcached/memcached/wiki/Protocols](https://github.com/memcached/memcached/wiki/Protocols)
- [https://github.com/memcached/memcached/blob/master/doc/protocol.txt](https://github.com/memcached/memcached/blob/master/doc/protocol.txt)
- [https://github.com/memcached/memcached/wiki/BinaryProtocolRevamped](https://github.com/memcached/memcached/wiki/BinaryProtocolRevamped)
- [https://github.com/memcached/memcached/wiki/ProtocolV3](https://github.com/memcached/memcached/wiki/ProtocolV3)
- [https://github.com/memcached/memcached/wiki/MetaCommands](https://github.com/memcached/memcached/wiki/MetaCommands)

It has two kinds of protocols, text and binary. Text protocol is human friendly, it can be used in telnet, so we can easily try and debug, but text is always slower than binary, so we should use binary protocol in production, right? 

Guess again. The first link said "binary affords us many new abilities", but this page is last updated in 2016.  From 4th link MetaCommands wiki above, which updated within a month, it says "binary protocol is deprecated", and now we should use text protocol. 

The MetaCommands is not as human friendly as the old text protocol. In old text protocol, we can use `get` `set` `add` `append` , etc. But in MetaCommands, we use `mg` instead of `get`, `ms` instead of `set`, and many flags to control the behavior.

# About the deployment

You can deploy it within the same machine of webserver, or dedicated servers, but don't do this in the database machine.

Maybe because the simplicity of the server side, it's very easy to use the CLI, just type `memcached` and here we go. The command line options are very simple too, like:

- -m: memory MB
- -d: daemon
- -v: verbose, type multiple times to get more information
- -p: port

# Performance and Internal

Performance may be the most important reason to decide whether to use memcached. They have a [page](https://github.com/memcached/memcached/wiki/Performance) to explain it in detail.

The internal of memcached is [here](https://github.com/memcached/memcached/wiki/UserInternals), memcached divide RAM into different parts, they call them slabs. Each slab is further divided into chunks, a chunk's size is not changed after created. When saving data, the server will find a nearest fit chunk, and there will be overhead. For example, there are 3 size of chunks, 80, 104 ,136 bytes, then to save an item(key + misc data + value) of 106 bytes, we need a chunk of 136 bytes, and the overhead is 30 bytes.

Just like the load factor in Java's HashMap, there is a growth factor `-f` to control the different chunk size level.

# Tutorial

MySQL official doc even has a [tutorial](https://dev.mysql.com/doc/refman/5.6/en/ha-memcached.html) for memcached! Maybe the best one even it's a little old.

Very useful, detailed command line examples, like how to use Unix socket instead of network ports, page, chunk, growth factor, scenario of wasting memory and how to improve it.

If I need to use memcached in future, I'll definitely  read this tutorial again.

[Cheatsheet](https://lzone.de/cheat-sheet/memcached)

# Client Library

Next, let's check the client library. Since I mainly use Java, I searched key word "[memcached](https://mvnrepository.com/search?q=memcached)" in maven central, the result is so poor, some results are too old, other results just don't look like a client library. Is this because memcached it too matured, or no one in Java world use memcached? After some searches in Google, finally I found [one promising](https://mvnrepository.com/artifact/net.spy/spymemcached), last updated May, 2017. 

Because the server only handle cache, the clients need to take care of all other stuffs, like choosing servers, encode and decode, consistent hashing and so on.

A simple way to locate the target server node by key is using mod, which is implemented in `ArrayModNodeLocator`, the core part is `int rv = (int) (hashAlg.hash(key) % nodes.length);`

Another way is Ketama, the blog link in the source file is outdated, the available one is [here](https://www.last.fm/user/RJ/journal/2007/04/10/rz_libketama_-_a_consistent_hashing_algo_for_memcache_clients), and the repository is [here](https://github.com/RJ/ketama). Ketama is a ring hash, when setup, put many virtual nodes on the ring, then to get the node of a key, hash the key and find the next node on the ring. This implementation uses a `TreeMap` to store and find the nodes, and supports node weights (though the javadocs says not).

For default serializer, the compression threshold is 16KB, beyond this the data will be gzipped.

For consistent hashing , this is a good [article](https://dgryski.medium.com/consistent-hashing-algorithmic-tradeoffs-ef6b8e2fcae8). There are so many different algorithms, each with their own trade offs.

# Cloud Services

Many cloud providers have memcached as a service.

## AWS

AWS has ElastiCache for Memcached, and even writes [a good comparison with Redis](https://aws.amazon.com/elasticache/redis-vs-memcached/).

AWS adds auto discovery to its service, clients can get the updates by a special key(`AmazonElastiCache:cluster`) or `config` command depends on your deployed version.

So AWS provides memcached as a service, what about the client library, did they write a new one? Unfortunately no, they modified the net.spy library above, and added other functions like auto discovery. See [this](https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/AutoDiscovery.Using.ModifyApp.Java.html) and [this](https://github.com/awslabs/aws-elasticache-cluster-client-memcached-for-java).

## Google

Google has Memorystore for Memcached, it also adds auto discovery. So can we say lack of auto discovery is the pain point of the memcached?

For the golang [client library](https://github.com/google/gomemcache), they forked [a open source one](https://github.com/bradfitz/gomemcache), and added auto discovery. For Java, I don't find any clue in the doc. Interesting as they say "The Auto Discovery service is also compatible with most clients supporting AWS Elasticache auto discovery". So the auto discovery protocol is same with AWS? The data part of AWS auto discovery protocol is `hostname|ip-address|port`, but Google has `node1-ip|node1-ip|node1-port`, the first part is not same.

Another difference is Google warns this, but AWS doesn't:

> You should use the Auto Discovery endpoint for its intended purpose, and not to run Memcached commands such as get, set, and delete.

From doc, it looks like Google uses one dedicate server to store cluster nodes config, but AWS save it to all cluster nodes.

Finally, the two [best](https://cloud.google.com/memorystore/docs/memcached/best-practices) [practice](https://cloud.google.com/memorystore/docs/memcached/memory-management-best-practices) pages are very worth reading if you're using Memorystore for Memcached. 

## Azure

Azure doesn't implement the service itself, but uses Memcached Cloud as a [store add-on](https://azure.microsoft.com/en-us/updates/memcached-cloud-available-in-the-azure-store/).

## [Memcached Cloud](https://redis.com/lp/memcached-cloud/)

This one is not really a memcached, but Redis. Nice trick!

# Anecdotes

memcached is used in DDoS, because in old version the UDP port is open by default, it will respond to spoof requests and cause an amplification attack.

[https://github.com/memcached/memcached/wiki/DDOS](https://github.com/memcached/memcached/wiki/DDOS)

[https://www.cloudflare.com/zh-cn/learning/ddos/memcached-ddos-attack/](https://www.cloudflare.com/zh-cn/learning/ddos/memcached-ddos-attack/)

[https://blog.cloudflare.com/memcrashed-major-amplification-attacks-from-port-11211/](https://blog.cloudflare.com/memcrashed-major-amplification-attacks-from-port-11211/)

# When to use memcached?

This question should be put in the head of this article, but after having so many materials above, we can easily derive the answer from them:

Since MySQL has a dedicated tutorial for memcached, they must be good friends.

And from [https://cloud.google.com/memorystore/docs/memcached/memcached-overview](https://cloud.google.com/memorystore/docs/memcached/memcached-overview)

> Some of the common Memcached use cases include caching of reference data, database query caching, and, in some cases, use as a session store.