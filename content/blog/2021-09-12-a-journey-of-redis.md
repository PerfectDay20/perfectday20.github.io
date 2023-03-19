+++
title = "A Journey of Redis"
+++

> This is by no means a tutorial or comprehensive introduction to Redis, it's just what I learnt and felt about Redis along the learning journey.

Redis, although known the name for many years, its full name is still new to me: REmote DIctionary Server, abbreviate to Redis.

Compared to memcached, Redis has so many functions. I first thought Redis as an alternative to memcached, but after looking through the documents, Redis is more like a swiss knife, and caching is only a small part of it.

# [The playground](http://try.redis.io/)

The official set provides a good online playground. More and more modern technologies provide online playground for new-comers to get hands wet, like golang, rust.

For the first taste of Redis in the playground, the command is even simpler than memcached: `SET key value` instead of `SET key 0 0 5\r\n value\r\n`. A second thought of the function: Redis is persistent, so add TTL flag to `set` is an optional choice, but keep the interface to its most frequently used function is the best choice.

OK, after a few pages in playground, here EXPIRE comes, `set key value EX 5`

After the playground, Redis has so many functions, no wonder it affords to build up a company, while memcached remains a tool (no offense, it's a great tool!)

# [Clients](https://redis.io/clients)

Compared to memcached, the ecosystem is very robust. Many clients for different languages, amazing. No need to search through the Maven Central and disappointed by the results

# [Introduction to Redis data types](https://redis.io/topics/data-types-intro)

This link is the tutorial, but the Redis team didn't put it in the first part of the [documentation](https://redis.io/documentation), even not the second part (it's third!). So after reading some dedicated function pages(Pipelining, Pub/Sub, Expires...), I have many questions, like what is a `database` within the notion of Redis. (Well, even after reading the tutorial, I still don't find the definition of database, it's hided in the [SELECT](https://redis.io/commands/select) command) Finally I find out the pages I have read were not "tutorial", but some more high level docs. Maybe they think the "Programming with Redis" and "Redis modules API" are more popular than the tutorial? 

OK, let's dive into the tutorial.

## String

Maximum allowed key and value sizes are both 512MB, huge enough.

Just like memcached, `INCR` parse the string as integer and increase it.

The `set` command is not just set "value", it sets "string value". So `set a 1` equals to `set a "1"`. INCR doc says "Redis does not have a dedicated integer type" and "Redis stores integers in their integer representation, so for string values that actually hold an integer, there is no overhead for storing the string representation of the integer". 

Why no integer types in both Memcached and Redis? The first thought is integer type is more efficient. Hard to implement?

## List

Linked list, not array list.

Although the `add` for array list runs in amortized constant time, a single `add` may take `O(n)` time. Redis thinks this is unbearable for a database system.

This list can be used in consumer-producer pattern between processes.

By using `BLPOP` `BRPOP`, blocking left/right pop, List can act like Java's BlockingQueue, at the consumer side. Since push is not blocked.

We don't need to create empty list, just use the key to push or pop or len or everything else, the result will be same as the list already exists.

## Hash

Works as expected.

## Set

In Java, I use Set mostly to filter out duplicates and test whether it contains an item. In Redis, they write long story about how to perform intersection, unions, difference and extract random element.

## Sorted set

Redis sorted set → Java's SortedSet, score → hashcode, values can have same score → Java objects can have same hashcode.

Redis sorted set is implemented by skip list and hash table. It can update the score with O(log(N)) time complexity. Very suitable for rank problems.

## Bitmap

Just like Java's BitSet. 

Bitmap is not a type, it's a string. In the above they mentioned maximum size of a string is 512MB, then 4G bits, 4 billion bits. With each bit we can save a flag, so a Bitmap can store flags for 4 billion users.

Since Bitmap is a string, we can use `GET` to get the actual value and parse the value by ourselves. The bit order is big endian, for example, after `setbit a 0 1` and `get a`, I get `"\x80"`.

## HyperLogLog

Trade memory for precision, just like Morris counter in the LRU doc part.

The error is within 1%, and worst case memory is 12KB.

It saves as bytes, so actually is string again.

The commands it uses are `PFADD PFCOUNT`, the awkward `PF` prefix is [in honor of HyperLogLog's inventor Philippe Flajolet](http://antirez.com/news/75).

(Oh, I kind of understand why Redis doesn't support other primitive data types like integer. String is bytes, the only primitive data type Redis supports is bytes, and many other types are relied on bytes, like Bitmap and HyperLogLog. Parse string to integer is not computation intensive, and store integer as string doesn't use much more memory, since most time we store much more other strings or bytes, the memory save for integer is negligible. Only support bytes makes the interface unified, easier to implement, maintain and expand.)

# [Introduction to Redis streams](https://redis.io/topics/streams-intro)

Redis streams works just like Kafka, it has three modes:

- multiple consumers see same messages, use `XREAD` → in Kafka, each consumer belongs to different group
- as a time series store, iterator by range, use `XRANGE` or `XREVRANGE` → in Kafka, this is not convenient
- multiple consumers divide messages, use `XGROUP`, `XREADGROUP`, `XACK` → in Kafka, all consumer belongs to one group

The entry ID works like Kafka's offset, but we can manually set the entry ID. Entry IDs must be monotonically increasing, and server auto generated entry ID is based on time, so mixing manually created entry ID with auto generated entry ID is not easy to handle.

When query by range, we can leverage the timestamp-based entry ID to filter by time range. Also the iterator can return elements in reverse order, which is not the main usage for Kafka.

For consumer group pattern, Redis is like a lightweight Kafka, no partitions, so no rebalance, consumer can join and leave easily. Besides, Redis tracks pending messages that are sent but not acknowledged. Redis stream can even use `XDEL` to delete a single item.

I wonder will anyone use Redis stream seriously in production instead of Kafka? When we already have a cluster of Redis and don't want to introduce a new tech stack?

# [Partitioning](https://redis.io/topics/partitioning)

[Redis is an in-memory but persistent on disk database](https://redis.io/topics/faq), all data is live in the memory, so when dataset is too large for a single instance, we need to partition it.

When used as a cache, we can just use client-side partitioning just like using memcached. Also [proxy assisted partitioning](https://github.com/twitter/twemproxy) is a good choice, the proxy handles choosing job for client, and client know nothing about the partitions.

When used as a datastore, things get complicated. We can't easily add and remove an instance,  or change the mapping between key and node. One way to mitigate this problem is to use presharding: just create way more partitions than we needed at first, say 32 or 64 in one machine, each Redis instance memory overhead is pretty low(1~3MB), then when we need more memory than the original machine, just plugin in another, and move some of the instances to the new one(by replication). This trick works, but not so flexible.

Any other choices for using Redis as a store? Luckily yes! They provided Redis cluster. But before that, we have to learn about replication.

# [Replication](https://redis.io/topics/replication)

There are two ways to replication. When a replica requests data that's still in the backlog buffer, server send the data directly, this is partial sync (quite familiar, where did I see this strategy before?). Otherwise, server need to dump the whole dataset to disk(can skip the disk too) then send the full dataset, this is full sync.

How replicas know how much data they need? Use the offset, but only offset is not enough. To prevent splitting brain when network partition, We also need a replication id to identify the current dataset, just like zookeeper's generation id, the replication id changes when a replica is promoted to a leader. Then there comes another question, when replication id changes, how all replicas with old id communication with the new leader, prevent a full sync? Use the old replication id! Yes, leader need to remember both the old and new id. So actually there are two replication ids.

The replication is asynchronous because of performance, but can be synchronous for less data loss in extreme condition.

We can disable leader's persistence for more performance, but everything comes with cost, we have to be careful to also disable auto restart. If leader auto restart with an empty dataset, then replicas will replicate the empty state too, all data will be lost.

The replicas can serve reading, and even writing in some use cases, for example computing slow Set or Sorted set operations and storing them into local keys

In Kafka, we can make sure only write to broker when they have more than a specific number of replicas in the ISR. Redis also has this config. It uses `min-replicas-to-write` to control the condition of minimal number of replicas, and `min-replicas-max-lag` to determine whether a replica is "in sync".

# [Redis cluster](https://redis.io/topics/cluster-tutorial)

Unlike the Sentinels in High Availability, where different nodes act as two distinct roles: sentinels and servers, in the Redis cluster all nodes act as servers. 

In the cluster, the data is shard by a simple version of consistent hashing: each leader has some hash slots, and only contains data with same hash mods. Each leader remembers other leaders' slots, so when a client asks for data to the wrong leader, the leader will return the right leader id and the client should try again. The leader doesn't act as a proxy. For efficiency, the client should cache the slot leader mappings.

Each node in the cluster has a unique node id that never change, this is how nodes remember each other instead of host:port. 

Because the slot algorithm is easy to understand, the operations of the cluster are also obvious. When we need to add a new leader to cluster, just move some slots from old leaders to it. When we need to remove a leader, just move all of its slots to others. Sadly no automatic rebalance now. We can manually incur failover to switch leader and slave, this way we can ensure no data loss. 

Why failover may incur data loss? Because the replication between leader and slave is asynchronous, when leader acknowledges a client write and crashes before sending write to the slave, this write will be lost. This scenario is very common in Redis docs when talking about data consistency.

Although no data automatic rebalance, the cluster has another useful feature: replicas migration. That's when a leader crashes and the only slave becomes the new leader, other leader's slaves can be automatically migrated to this new leader. No data rebalance, but has replicas rebalance.

# [Persistence](https://redis.io/topics/persistence)

We already see RDB file in the Replication when replicas need a full sync, what exactly is RDB?

Redis is a persistence database, it has two persistent file types, RDB (Redis database) and AOF (append only file).

RDB is a snapshot of the memory, created when condition(at least M writes in N seconds) is met. It's suitable for backup and failover.

AOF can be tuned to fsync for every command, or every second, or decided by kernel. Only fsync every command can ensure no data loss, but that's very expensive. The performance of fsync every second is good enough as RDB, fsync for every command is no doubt the slowest.

Due to the characteristic of AOF, the file has redundant and needs to be rewrite. The rewrite is not based on old AOF, but the dataset in memory. The doc says this is more robust, because there are rare bugs of AOF, though not ever reported in production.

So in production, use RDB + AOF, use cron job to back RDB, for example hourly snapshot for last 48 hours and daily snapshot for last 2 months.

# [Signals Handling](https://redis.io/topics/signals)

`SIGTERM` and `SIGINT` to gracefully shutdown the server. The actions include kill background RDB AOF rewrite jobs, fsync current AOF, save current RDB, remove pid file and socket file, exit 0.

# [High Availability](https://redis.io/topics/sentinel)

Redis servers don't elect a new leader by themselves when failover, but use a external service: sentinel(though actually they are the same binary executable file). Sentinel controls the failover, but this can be a single point of failure, so there must be a group of sentinels. So in general, to have HA, we must have a group of sentinels, a leader, and a group of replicas.

The configuration file is also the state of sentinel, this means sentinel will update the conf file and store states in it.

A quorum number of sentinels are needed to agree on the leader failure, but to elect the new leader, they need the votes of the majority of the total sentinel processes.

The doc contains many deployment examples, but after reading this I still don't know the best practice, maybe deploy each Redis instance with a sentinel together?

One problem the sentinel can't solve is when network partition and client writes to old leader, when network recover the new written data will be lost. We can only use the configs in replication section to minimize the time window, only allow server to accept write when majority replicas are available. For example, 5 server A-E, A is leader, then network partition happens, A-B is a group, C-E is a group with C is new leader. To minimize the data loss, we need to set `min-replicas-to-write 2`, because 2+1=3 is the majority, then old leader A will reject writes.

# [Benchmarks](https://redis.io/topics/benchmarks)

To benchmark correctly is very difficult, even when Redis provides a benchmark tool, there are still many factors to consider: same version with different configs, or different version with same configs, the characteristic of comparison opponent(DB or cache), NIC bandwidth, loopback or Unix socket, NUMA, client numbers, pipelining, VM or bare metal, persistent policy, log level, monitor tools...

> Finally, when very efficient servers are benchmarked (and stores like Redis or memcached definitely fall in this category), it may be difficult to saturate the server. Sometimes, the performance bottleneck is on client side, and not server-side. In that case, the client (i.e. the benchmark program itself) must be fixed, or perhaps scaled out, in order to reach the maximum throughput.

So the best guess of the performance should be testing Redis in the same way as the production usage.

# [Pipelining](https://redis.io/topics/pipelining)

We can use pipelining to speed up processing. It reduces RTT and socket I/O (system call).

But I don't quite understand what the appendix says. It says due to kernel thread scheduler, the server process in the same machine is only running when it is scheduled. Yes, but this condition also applies to the ruby benchmark above. One uses `10000.times` and costs 1.185238 seconds, one uses `FOR-ONE-SECOND`, what's the difference? Is the ruby benchmark also silly? 

# [Pub/Sub](https://redis.io/topics/pubsub)

Redis can be used as a publish/subscribe system. But I don't see anything about message persistent or rollback. So the messages are just like radio in the air, if we missed it, then it just gone? Yes.

# [Expires](https://redis.io/commands/expire)

Only full overwrite a key will reset its TTL, `INCR` or `RENAME` won't affect the TTL.

Although `EXPIRE` exists from version 1.0.0, it still gains new functions in newer versions. Redis 7.0 adds a set of options for atomic actions, so need to test and set now.

The expire accuracy is 1ms now.

An interesting part is how Redis handles expires. The passive way is when a key is accessed. The active way is randomly checking 20 keys with expire set every second, if more than 25% expires, check again in no time. 

So the expire accuracy doesn't means a key is deleted within 1ms when it is expired. It means the minimum time difference that the server can tell if the key is expired. When a key is expired, it can still be in the RAM.

# [Redis as an LRU cache](https://redis.io/topics/lru-cache)

Redis can act as an LRU cache, and has many policies when evict data. They are `noeviction`, `volatile-ttl`, and combination of `allkeys` `volatile` with `random` `lru` `lfu`.

To save the memory, the LRU algorithm is approximated. Redis samples a number of keys and selects the best one. The default number is 5. At first sight I thought this number is too small, but there is a good graph to show the comparison of different value, compared with theoretical LRU. The result is promising, and 5 is good enough.

For the LFU mode, it means least frequently used. For example we have 10 slots, each slot contains a key (A1 to A10), and each key is accessed for 100 times, these 10 keys are frequently used in the past, and we predict they will also be used in the future. Suddenly the client create 10 new keys(B1 to B10) now, in the LRU mode, those 10 old frequently used keys(A1 to A10) will be evicted, that's what we don't want to see. Then here LFU mode goes. In LFU mode, the new keys(B1 to B10) will be evicted and old keys are left. But what if the old keys won't come again, and the new keys are coming over and over again? After first round of B1 to B10, when B1 comes again, old B1 is already evicted, so no counter for B1 exists, then how the new key(B1 to B10) beats the old key(A1 to A10)? The answer is decay period, as time goes, counter for A1 to A10 decays. Brilliant!

Besides, the counter is not just a simple integer counter, it's [Morris counter](https://en.wikipedia.org/wiki/Approximate_counting_algorithm), an approximate counting algorithm, using probabilistic to count, and trading accuracy for for minimal memory usage.

# [Transactions](https://redis.io/topics/transactions)

Use `MULTI` and `EXEC` to start and end a transaction.

Use `WATCH` as a condition of transaction execution, to implement new CAS actions. The original CAS action is check the current value with the expected value, but `WATCH` checks if the value has changed.

Transaction can be replaced with Redis script, because Redis script is transactional by definition.

# [Client side caching](https://redis.io/topics/client-side-caching)

When using client side caching, Redis is more like a data store instead of a cache, or we can say the client side cache is like L3 cache, and Redis is like RAM.

One important feature of client side caching is invalidation message. There are two modes:

- Server remembers all clients' `GET` keys, and send to the client whose items are invalidated. More clients, more `GET` keys, the server consume more memory. To mitigate this, Redis uses a sized global table to store the requested keys, and evicts old ones when oversize.
- Client register specific prefixes of keys they want to receive invalidation message. This way server only needs to remember the client-prefixes pairs, doesn't affect memory, but the more prefixes, the more CPU overhead.

Normally when a client modify a key-value pair, it will also receive the invalidation message if it cached the key. In some cases we don't need the invalidation message because we can change the cache ourselves after write to Redis. So there is a very considerate option `NOLOOP`, nice!

Client side caching is suitable for keys that are requested often and changed low to medium rate.

# [Mass insertion](https://redis.io/topics/mass-insert)

Redis provide a way to populate a brand new Redis server with great amount of data as fast as possible: `cat data.txt | redis-cli --pipe`. But the data file contains not plain text key-value pair, or `SET` commands, but Redis protocol. A little awkward and unfriendly.

# [Memory optimization](https://redis.io/topics/memory-optimization)

Redis uses special encodings for small aggregate data types to improve memory usage without affecting functionality. When the number of elements or size of elements exceeds the limit, Redis will convert it to normal encoding.

A practical usage is when we need to save a bunch of values of a object, like a person's name, age and address, instead of using different key combinations of ids and field names: `SET 1:name foo` `SET 1:age 20` `SET 1:address bar` , use a hash is both more efficient and more logical: `HSET person:1 name foo` `HSET person:1 age 20` `HSET person:1 address bar`.

When we only has a plain key value dataset, we can still use this trick, by splitting the key to two parts, the first part is the hash name, the second part is the field name. For example: convert `SET object:1234 foo` to `HSET object:12 34 foo`. The downside is the code logic is not so straight forward.

 ---

Such a long journey! Redis already has so many features, the development is still very active (maybe too fast, we can see it from that [Elasticache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/SelectEngine.html) provides different version from 2.8 to 6.x). We can surely foresee more features to come! 