# Design-Cache-Redis
Design a distributed key value caching system, like Memcached or Redis.

![image](https://github.com/Mugdha-Hazra/Design-Cache-Redis/assets/63424869/bf195677-617e-466e-8054-436d017422ee)


---

Redis is an open-source, in-memory data structure store, used as a database, cache, and message broker. Redis provides data structures such as strings, hashes, lists, sets, sorted sets with range queries, bitmaps, hyperloglogs, geospatial indexes, and streams. Redis has built-in replication, Lua scripting, LRU eviction, transactions, and different levels of on-disk persistence, and provides high availability via Redis Sentinel and automatic partitioning with Redis Cluster.

# So Redis can be used as a traditional monolithic and can be used as distributed system as a cluster of nodes with sharding.

# Before talking about Redis I would explain some concepts and why we need these terms and techniques in our systems.

A Cache is like short-term memory. It is typically faster than the original data source. Accessing data from memory is faster than from a hard disk. Caching means saving frequently accessed data in-memory(short-term memory) so the value that is added by caching is retrieving data fastly and reduce calling the original data source it might be SQL DB because the complexity time of reading data will be o(1) as a direct access operation by key in memory like hashtables.

# So Redis is a system that offers to us a caching system in both environments monolithic and distributed.

All Redis data resides in the server’s main memory, in contrast to databases such as PostgreSQL, SQL Server, and others that store most data on disk. In comparison to traditional disk-based databases where most operations require a roundtrip to disk, in-memory data stores such as Redis don’t suffer the same penalty. They can therefore support an order of magnitude more operations and faster response times. The result is — blazing fast performance with average read or writes operations taking less than a millisecond and support for millions of operations per second.

# So as Redis is faster than traditional DB can we consider it the first source of truth?

The answer is No we can not consider Redis as the first source of truth it always comes as second support to improve the performance of the system because from the CAP theorem perspective Redis is neither highly available nor consistent. to understand why let's explain how Redis sync the data from memory to disk as the disk can consider consistency.

As we explained before Redis data lives in memory, which makes it is very fast to write to and read from, but in case of server crashes you lose all that’s in the memory, for some applications, it’s ok to lose these data in case of a crash, but for other apps, it’s important to be able to reload Redis data after server restarts.

# So Redis provides a different range of persistence options:

RDB (Redis Database): The RDB persistence performs point-in-time snapshots of your dataset at specified intervals.
AOF (Append Only File): The AOF persistence logs every write operation received by the server, that will be played again at server startup, reconstructing the original dataset. Commands are logged using the same format as the Redis protocol itself, in an append-only fashion. Redis is able to rewrite the log in the background when it gets too big.
No persistence: If you wish, you can disable persistence completely, if you want your data to just exist as long as the server is running.
RDB + AOF: It is possible to combine both AOF and RDB in the same instance. Notice that, in this case, when Redis restarts the AOF file will be used to reconstruct the original dataset since it is guaranteed to be the most complete.
So the most important part there is understanding the trade-offs between the RDB and AOF persistence as the No persistence is very clear that no level of consistency there even strong or bad consistency.

# RDB advantages :

RDB is a very compact single-file point-in-time representation of your Redis data. RDB files are perfect for backups. For instance, you may want to archive your RDB files every hour for the latest 24 hours and to save an RDB snapshot every day for 30 days. This allows you to easily restore different versions of the data set in case of disasters.
RDB is very good for disaster recovery, being a single compact file that can be transferred to far data centers.
RDB allows faster restarts with big datasets compared to AOF
# RDB disadvantages:

RDB is NOT good if you need to minimize the chance of data loss in case Redis stops working (for example after a power outage). You can configure different save points where an RDB is produced (for instance after at least five minutes and 100 writes against the data set, but you can have multiple save points). However you’ll usually create an RDB snapshot every five minutes or more, so in case of Redis stopping working without a correct shutdown for any reason, you should be prepared to lose the latest minutes of data.
# AOF advantages:

Using AOF Redis is much more durable: you can have different fsync policies: no fsync at all, fsync every second, fsync at every query. With the default policy of fsync every second write performances are still great (fsync is performed using a background

 thread and the main thread will try hard to perform writes when no fsync is in progress.) but you can only lose one second worth of writes.
The AOF log is an append-only log, so there are no seeks, nor corruption problems if there is a power outage. Even if the log ends with a half-written command for some reason (disk full or other reasons) the Redis-check-of tool is able to fix it easily.
Redis is able to automatically rewrite the AOF in the background when it gets too big. The rewrite is completely safe as while Redis continues appending to the old file, a completely new one is produced with the minimal set of operations needed to create the current data set, and once this second file is ready Redis switches the two and starts appending to the new one.
AOF contains a log of all the operations one after the other in an easy-to-understand and parsed format. You can even easily export an AOF file. For instance, even if you’ve accidentally flushed everything using the FLUSHALL command, as long as no rewrite of the log was performed in the meantime, you can still save your data set just by stopping the server, removing the latest command, and restarting Redis again.
# AOF disadvantages:

AOF files are usually bigger than the equivalent RDB files for the same dataset.
AOF can be slower than RDB depending on the exact fsync policy.
Finally, AOF can improve the data consistency but does not guarantee so likely you can lose your data but less than RDB mode considering the RDB is faster.

# what should I use?

It depends as usual in any system design but The general indication is that you should use both persistence methods if you want a degree of data safety comparable to what PostgreSQL can provide you. If you care a lot about your data, but still can live with a few minutes of data loss in case of disasters, you can simply use RDB alone.

After we explain the mechanism of data store in Redis let’s explain two important persistence models.

# Snapshotting:

By default Redis saves snapshots of the dataset on disk, in a binary file called dump.rdb. You can configure Redis to have it save the dataset every N seconds if there are at least M changes in the dataset, or you can manually call the SAVE or BGSAVE commands.

# How it works:

Redis forks. We now have a child and a parent process.
The child starts to write the dataset to a temporary RDB file.
When the child is done writing the new RDB file, it replaces the old one.
So Redis stores snapshots of your data to disk in the following conditions:

Every minute if 1000 keys were changed
Every 5 minutes if 10 keys were changed
Every 15 minutes if 1 key was changed
So if you’re doing heavy work and changing lots of keys, then a snapshot per minute will be generated for you, in case your changes are not that much then a snapshot every 5 minutes, if it’s really not that much then every 15 minutes a snapshot will be taken.

# Append-only file:

Snapshotting is not very durable. If your computer running Redis stops, your power line fails, or you accidentally kill -9 your instance, the latest data written on Redis will get lost. While this may not be a big deal for some applications, there are use cases for full durability, and in these cases, Redis was not a viable option. The append-only file is an alternative, fully-durable strategy for Redis. It became available in version 1.1. You can turn on the AOF in your configuration file:

appendonly yes
# How durable is the append-only file?

As we explained in AOF section we have the following options for durability levels

appendfsync always: fsync every time new commands are appended to the AOF. Very very slow, very safe. Note that the commands are appended to the AOF after a batch of commands from multiple clients or a pipeline are executed, so it means a single write and a single fsync (before sending the replies).
appendfsync everysec: fsync every second. Fast enough (in 2.4 likely to be as fast as snapshotting), and you can lose 1 second of data if there is a disaster.
appendfsync no: Never fsync, just put your data in the hands of the Operating System. The faster and less safe method. Normally Linux will flush data every 30 seconds with this configuration, but it's up to the kernel exact tuning.
# How it works:

Redis forks, so now we have a child and a parent process.
The child starts writing the new AOF in a temporary file.
The parent accumulates all the new changes in an in-memory buffer (but at the same time it writes the new changes in the old append-only file, so if the rewriting fails, we are safe).
When the child is done rewriting the file, the parent gets a signal and appends the in-memory buffer at the end of the file generated by the child.
Profit! Now Redis atomically renames the old file into the new one.
The great thing is that even if Redis stops working in the middle of the rewrite, we can still use the old file with the same correctness guarantees. So it's very safe to use the append-only file, especially with fsync every second, as you can lose at most 1 second worth of writes if a disaster happens.

# Ok but what’s the difference?

In short, Redis will append to the AOF all the write operations that will modify the dataset, that is also true for commands that are modifying the dataset by consuming other commands like SUNIONSTORE, or ZUNIONSTORE, but they can be represented as a sequence of simple command. However, read-only commands are not appended to the AOF.

# RDB and AOF both have their advantages and disadvantages, and which one to choose depends on your specific use case and requirements for data consistency, durability, and performance.

In summary, Redis is a powerful in-memory data structure store that can be used as a database, cache, or message broker. It provides various data structures and persistence options, allowing developers to optimize performance and durability according to their specific needs. However, choosing the right persistence model (RDB or AOF) requires careful consideration of factors such as data safety, recovery time objective, and performance trade-offs.
