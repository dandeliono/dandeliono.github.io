#  Redis key 过期
## [\*](#keys-with-an-expire)Keys with an expire 有过期的 key

Normally Redis keys are created without an associated time to live. The key will simply live forever, unless it is removed by the user in an explicit way, for instance using the [DEL](https://redis.io/commands/del) command.

通常，Redis 键是在没有相关生存时间的情况下创建的。密钥将永远存在，除非用户以显式的方式删除它，例如使用 DEL 命令。

The [EXPIRE](https://redis.io/commands/expire) family of commands is able to associate an expire to a given key, at the cost of some additional memory used by the key. When a key has an expire set, Redis will make sure to remove the key when the specified amount of time elapsed.

终止命令系列能够将终止期限关联到给定的密钥，代价是该密钥所使用的一些额外内存。如果密钥设置了过期设置，Redis 将确保在指定的时间过期时删除密钥。

The key time to live can be updated or entirely removed using the [EXPIRE](https://redis.io/commands/expire) and [PERSIST](https://redis.io/commands/persist) command (or other strictly related commands).

可以使用 EXPIRE 和 PERSIST 命令 (或其他严格相关的命令) 更新或完全删除关键存活时间。

## [\*](#expire-accuracy)Expire accuracy 过期精度

In Redis 2.4 the expire might not be pin-point accurate, and it could be between zero to one seconds out.

在 Redis 2.4 中，过期值可能不精确，而且可能在 0 到 1 秒之间。

Since Redis 2.6 the expire error is from 0 to 1 milliseconds.

由于 Redis 2.6，过期错误从 0 毫秒到 1 毫秒。

## [\*](#expires-and-persistence)Expires and persistence 过期和持久性

Keys expiring information is stored as absolute Unix timestamps (in milliseconds in case of Redis version 2.6 or greater). This means that the time is flowing even when the Redis instance is not active.

到期的键信息以绝对 Unix 时间戳的形式存储 (在 Redis 2.6 或更大版本中，以毫秒为单位)。这意味着即使 Redis 实例不活动，时间也是流动的。

For expires to work well, the computer time must be taken stable. If you move an RDB file from two computers with a big desync in their clocks, funny things may happen (like all the keys loaded to be expired at loading time).

为了使计算机正常工作，计算机时间必须稳定。如果您将 RDB 文件从两台计算机的时钟中带有大的 desync 的计算机中移动，可能会发生有趣的事情 (比如加载时加载的所有密钥都过期了)。

Even running instances will always check the computer clock, so for instance if you set a key with a time to live of 1000 seconds, and then set your computer time 2000 seconds in the future, the key will be expired immediately, instead of lasting for 1000 seconds.

即使是正在运行的实例也总是会检查计算机时钟，所以举例来说，如果你设置一个时间为 1000 秒的密钥，然后在未来设置你的计算机时间为 2000 秒，这个密钥将立即过期，而不是持续 1000 秒。

## [\*](#how-redis-expires-keys)How Redis expires keys 如何 Redis 过期键

Redis keys are expired in two ways: a passive way, and an active way.

Redis 密钥有两种过期方式: 被动过期和主动过期。

A key is passively expired simply when some client tries to access it, and the key is found to be timed out.

当某个客户端试图访问密钥时，密钥被动地过期，并且发现密钥已超时。

Of course this is not enough as there are expired keys that will never be accessed again. These keys should be expired anyway, so periodically Redis tests a few keys at random among keys with an expire set. All the keys that are already expired are deleted from the keyspace.

当然，这是不够的，因为有过期的密钥将永远不会再次访问。这些密钥无论如何都应该过期，所以 Redis 会定期在具有过期设置的密钥之间随机测试一些密钥。所有已过期的密钥都将从密钥空间中删除。

Specifically this is what Redis does 10 times per second:

具体来说，这就是 Redis 每秒钟做的 10 次:

1.  Test 20 random keys from the set of keys with an associated expire. 测试来自带有关联过期值的键集合的 20 个随机键
2.  Delete all the keys found expired. 删除所有过期的密钥
3.  If more than 25% of keys were expired, start again from step 1. 如果超过 25% 的密钥过期，从第一步开始重新启动

This is a trivial probabilistic algorithm, basically the assumption is that our sample is representative of the whole key space, and we continue to expire until the percentage of keys that are likely to be expired is under 25%

这是一个简单的概率算法，基本上假设我们的样本代表了整个键空间，我们继续过期，直到可能过期的键的百分比低于 25%

This means that at any given moment the maximum amount of keys already expired that are using memory is at max equal to max amount of write operations per second divided by 4.

这意味着在任何给定时刻，正在使用内存的已过期键的最大数量等于每秒写操作的最大数量除以 4。

## [\*](#how-expires-are-handled-in-the-replication-link-and-aof-file)How expires are handled in the replication link and AOF file 在复制链接和 AOF 文件中如何处理过期

In order to obtain a correct behavior without sacrificing consistency, when a key expires, a [DEL](https://redis.io/commands/del) operation is synthesized in both the AOF file and gains all the attached replicas nodes. This way the expiration process is centralized in the master instance, and there is no chance of consistency errors.

为了在不牺牲一致性的情况下获得正确的行为，当一个密钥失效时，在 AOF 文件中合成一个 DEL 操作，并获得所有附加的副本节点。通过这种方式，过期过程集中在主实例中，不会出现一致性错误。

However while the replicas connected to a master will not expire keys independently (but will wait for the [DEL](https://redis.io/commands/del) coming from the master), they'll still take the full state of the expires existing in the dataset, so when a replica is elected to master it will be able to expire the keys independently, fully acting as a master.

然而，虽然连接到主控制器的副本不会独立地过期密钥 (但是会等待主控制器发出 DEL) ，但是它们仍然会接受数据集中已经存在的过期的完整状态，因此当一个副本被选择为主控制器时，它将能够独立地过期密钥，充当主控制器。 
 [https://redis.io/commands/expire#how-redis-expires-keys](https://redis.io/commands/expire#how-redis-expires-keys)
