## Redis缓存淘汰策略

将Redis用作缓存时, 如果内存空间用满, 就会自动驱逐老的数据, 默
认情况下memcached就是这种方式.
**LRU是Redis唯一支持的回收算法** 
maxmemory配置指令
maxmemory用于指定Redis能使用的最大内存. 可以在redis.conf文件中配置, 也可以在运行过程中动态修改.
例如, 要设置 100MB 的内存限制, 可以在redis.conf文件中配置:
maxmemory  100mb
将maxmemory设置为0, 表示不进行内存限制. 对一个32位的操作系统来
说有一个隐性的限制条件, 最多3GB.
当内存使用达到最大限制时, 如果需要存储新的数据, 根据配置的策略
不同, Redis可能直接返回错误信息, 或者删除部分旧数据.

1. 驱逐策略
达到内存的最大限制时(maxmemory), Redis根据maxmemory-policy配置的策略, 来决定具体的行为.
Redis支持的策略包括:
(1) noeviction: 不删除策略. 达到最大内存限制时, 如果需要更多的内存, 直接返回错误信息, 大多数写命令都会导致占用更多的内存.
(2) allkeys-lru: 所有key通用, 优先删除最近最少使用的key, 基于LRU算法.
(3) volatile-lru: 只限于设置了expire的部分, 优先使用删除LRU的key.
(4) allkeys-random: 所有key通用, 随机删除一部分key.
(5) volatile-random: 只限于设置了expire的部分, 随机删除一部分Key.
(6) volatile_ttl: 只限于设置了expire的部分, 优先删除剩余时间(TTL)短的key. 

如果没有设置expire的key, 不满足先决条件, 那么volatile-lru, 
volatile-random, volatile-ttl策略的行为, 和noeviction(不删除)
基本一致.

2. 驱逐策略的选择
    需要根据系统的特性, 来选择合适的驱逐策略.
    (1) 如果分为冷数据和热数据, 推荐使用allkeys-lru策略, 也就是, 其中一部分key经常被读写, 如果不确定具体的业务特征, 使用allkeys-lru.
    (2) 如果需要循环读写所有的key, 或者各个key的访问频率差不多, 可以使用allkeys-random策略, 即读写所有元素的频率差不多.
    (3) 假如要让redis根据TTL来删除需要删除的key, 则可以使用volatile-ttl策略.

3. 近似LRU算法
    Redis使用的并不完全是LRU算法, 自动驱逐的key, 并不一定是最满足LRU特征的那个, 而是通过近似的LRU算法, 抽取少量的key样本, 然后删除其中访问时间最久的那个key.驱逐算法, 从Redis3.0开始得到了巨大优化, 使用pool(池子)来作为候选, 这大大提升了算法的效率, 也更接近真实的LRU.在Redis的驱逐策略中, 可以通过设置样本(sample)的数量来调优算法. 

  通过以下指令配置:
  maxmemory-samples  5

4. 为什么不完全使用LRU实现??
   原因是为了节省内存, 但是Redis的行为和LRU基本等价, 在Redis3.0中, 使用10样本, 已经非常接近纯粹的LRU算法了. 与真正的LRU算法差别非常小, 但是将样本提升为10, 也额外消耗了更多的CPU资源.