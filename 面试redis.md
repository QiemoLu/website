Redis有几种类型
String、Hash、List、Set、SortedSet。

Pub/Sub。

如果有大量的key需要设置同一时间过期，一般需要注意什么？
如果大量的key过期时间设置的过于集中，到过期的那个时间点，Redis可能会出现短暂的卡顿现象。严重的话会出现缓存雪崩，我们一般需要在时间上加一个随机值，使得过期时间分散一些。

那你使用过Redis分布式锁么，它是什么回事？
先拿setnx来争抢锁，抢到之后，再用expire给锁加一个过期时间防止锁忘记了释放,SETEX key seconds value，成功返回1， 反之返回0. 释放锁 del key
利用的是redis的原子性，单线程串行访问。

这时候对方会告诉你说你回答得不错，然后接着问如果在setnx之后执行expire之前进程意外crash或者要重启维护了，那会怎么样？
这里主要是考锁得不到释放
这个应该是可以同时把setnx和expire合成一条指令来用的。

如果这个redis正在给线上的业务提供服务，那使用keys指令会有什么问题？
Redis的单线程的。keys指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。这个时候可以使用scan指令，scan指令可以无阻塞的提取出指定模式的key列表，但是会有一定的重复概率，在客户端做一次去重就可以了，但是整体所花费的时间会比直接用keys指令长
使用过Redis做异步队列么，你是怎么用的
一般使用list结构作为队列，rpush生产消息，lpop消费消息。当lpop没有消息的时候，要适当sleep一会再重试。 list还有个指令叫blpop，在没有消息的时候，它会阻塞住直到消息到来。
使用pub/sub主题订阅者模式，可以实现 1:N 的消息队列

如果对方究极TM追问Redis如何实现延时队列？

这一套连招下来，我估计现在你很想把面试官一棒打死（面试官自己都想打死自己了怎么问了这么多自己都不知道的），如果你手上有一根棒球棍的话，但是你很克制。平复一下激动的内心，然后神态自若的回答道：使用sortedset，拿时间戳作为score，消息内容作为key调用zadd来生产消息，消费者用zrangebyscore指令获取N秒之前的数据轮询进行处理。
添加数据
127.0.0.1:6379> ZADD testSet1 5 a
(integer) 1
127.0.0.1:6379> ZADD testSet1 1 b 8 c 7 d
(integer) 3
读取
127.0.0.1:6379> ZRANGEBYSCORE testSet1 0 3
1) "b"
127.0.0.1:6379> ZRANGEBYSCORE testSet1 0 5
1) "b"
2) "a"
处理失败最好再次塞回去

Redis是怎么持久化的？服务主从数据怎么交互的？

RDB做镜像全量持久化，AOF做增量持久化。因为RDB会耗费较长时间，不够实时，在停机的时候会导致大量丢失数据，所以需要AOF来配合使用。在redis实例重启时，会使用RDB持久化文件重新构建内存，再使用AOF重放近期的操作指令来实现完整恢复重启之前的状态。

 RDB对Redis的性能影响非常小，是因为在同步数据的时候他只是fork了一个子进程去做持久化的，而且他在数据恢复的时候速度比AOF来的快。RDB都是快照文件，都是默认五分钟甚至更久的时间才会生成一次
AOF是一秒一次去通过一个后台的线程fsync操作，那最多丢这一秒的数据。
一样的数据，AOF文件比RDB还要大。
AOF开启后，Redis支持写的QPS会比RDB支持写的要低，他不是每秒都要去异步刷新一次日志嘛fsync，当然即使这样性能还是很高

对方追问那如果突然机器掉电会怎样？

取决于AOF日志sync属性的配置，如果不要求性能，在每条写指令时都sync一下磁盘，就不会丢失数据。但是在高性能的要求下每次都sync是不现实的，一般都使用定时sync，比如1s1次，这个时候最多就会丢失1s的数据。
分布式锁的缺陷
一、客户端1得到了锁，因为网络问题或者GC等原因导致长时间阻塞，然后业务程序还没执行完锁就过期了，这时候客户端2也能正常拿到锁，可能会导致线程安全的问题。

二、redis服务器时钟漂移问题
如果redis服务器的机器时钟发生了向前跳跃，就会导致这个key过早超时失效，比如说客户端1拿到锁后，key的过期时间是12:02分，但redis服务器本身的时钟比客户端快了2分钟，导致key在12:00的时候就失效了，这时候，如果客户端1还没有释放锁的话，就可能导致多个客户端同时持有同一把锁的问题。
三、单点实例安全问题
如果redis是单master模式的，当这台机宕机的时候，那么所有的客户端都获取不到锁了，为了提高可用性，可能就会给这个master加一个slave，但是因为redis的主从同步是异步进行的，可能会出现客户端1设置完锁后，master挂掉，slave提升为master，因为异步复制的特性，客户端1设置的锁丢失了，这时候客户端2设置锁也能够成功，导致客户端1和客户端2同时拥有锁。
为了解决Redis单点问题，redis的作者提出了RedLock算法。
1、获取当前时间戳（ms）；
2、先设定key的有效时长（TTL），超出这个时间就会自动释放，然后client（客户端）尝试使用相同的key和value对所有redis实例进行设置，每次链接redis实例时设置一个比TTL短很多的超时时间，这是为了不要过长时间等待已经关闭的redis服务。并且试着获取下一个redis实例。
比如：TTL（也就是过期时间）为5s，那获取锁的超时时间就可以设置成50ms，所以如果50ms内无法获取锁，就放弃获取这个锁，从而尝试获取下个锁；
3、client通过获取所有能获取的锁后的时间减去第一步的时间，还有redis服务器的时钟漂移误差，然后这个时间差要小于TTL时间并且成功设置锁的实例数>= N/2 + 1（N为Redis实例的数量），那么加锁成功
比如TTL是5s，连接redis获取所有锁用了2s，然后再减去时钟漂移（假设误差是1s左右），那么锁的真正有效时长就只有2s了；
4、如果客户端由于某些原因获取锁失败，便会开始解锁所有redis实例。Redis的同步机制了解么？



Redis的同步机制了解么？

Redis可以使用主从同步，从从同步。第一次同步时，主节点做一次bgsave，并同时将后续修改操作记录到内存buffer，待完成后将RDB文件全量同步到复制节点，复制节点接受完成后将RDB镜像加载到内存。加载完成后，再通知主节点将期间修改的操作记录同步到复制节点进行重放就完成了同步过程。后续的增量数据通过AOF日志同步即可，有点类似数据库的binlog。
Redis为什么这么快？
采用的是基于内存的采用的是单进程单线程模型的 KV 数据库，由C语言编写，官方提供的数据是可以达到100000+的QPS（每秒内查询次数）

* 完全基于内存，绝大部分请求是纯粹的内存操作，非常快速。它的，数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；
* 数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的；
* 采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗；
* 使用多路I/O复用模型，非阻塞IO；
* 使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；

高可用，Redis还有其他保证集群高可用的方式么？
还有哨兵集群sentinel。
哨兵必须用三个实例去保证自己的健壮性的，哨兵+主从并不能保证数据不丢失，但是可以保证集群的高可用。

* 集群监控：负责监控 Redis master 和 slave 进程是否正常工作。
* 消息通知：如果某个 Redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
* 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。
* 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。

Redis过期淘汰机制
Redis的过期策略，是有定期删除+惰性删除两种。
定期好理解，默认100ms就随机抽一些设置了过期时间的key，去检查是否过期，过期了就删了。
如果一直没随机到很多key，里面不就存在大量的无效key了？
好问题，惰性删除，见名知意，惰性嘛，我不主动删，我懒，我等你来查询了我看看你过期没，过期就删了还不给你返回，没过期该怎么样就怎么样。

内存淘汰策略
一般的剔除策略有 FIFO 淘汰最早数据、LRU 剔除最近最少使用、和 LFU 剔除最近使用频率最低的数据几种策略。
1. noeviction：当内存使用超过配置的时候会返回错误，不会驱逐任何键

2. allkeys-lru：加入键的时候，如果过限，首先通过LRU算法驱逐最久没有使用的键

3. volatile-lru：加入键的时候如果过限，首先从设置了过期时间的键集合中驱逐最久没有使用的键

4. allkeys-random：加入键的时候如果过限，从所有key随机删除

5. volatile-random：加入键的时候如果过限，从过期键的集合中随机驱逐

6. volatile-ttl：从配置了过期时间的键中驱逐马上就要过期的键

7. volatile-lfu：从所有配置了过期时间的键中驱逐使用频率最少的键

8. allkeys-lfu：从所有键中驱逐使用频率最少的键

Redis 底层结构
键k1是一个字符串，底层实现是保存着字符串k1的SDS，值v1也是一个字符串，底层实现是保存着字符串v1的SDS
string
struct sdshdr {
    //记录buf数组中已使用字节的数量，相当于保存的字符串的长度
    int len;  
    //记录buf数组中未使用的字节数量
    int free;
    //字节数组，用于保存字符串
    char buf[]
};
List
struct listNode{
    //前置节点
    struct listNode *prev;
    //后置节点
    struct listNode *next;
    //节点的值
    void *value;     
}listNode;
不聋过滤器
原理是给加入集合的键做标记，1说明存在， 0说明不存在，
简单来说是过滤请求，把数据库和缓存都没有数据的键值为0，避免缓存击穿和无效请求。

bloom filter之所以能做到在时间和空间上的效率比较高，是因为牺牲了判断的准确率、删除的便利性
* 存在误判，可能要查到的元素并没有在容器中，但是hash之后得到的k个位置上值都是1。如果bloom filter中存储的是黑名单，那么可以通过建立一个白名单来存储可能会误判的元素。
* 删除困难。一个放入容器的元素映射到bit数组的k个位置上是1，删除的时候不能简单的直接置为0，可能会影响其他元素的判断。可以采用Counting Bloom Filter

缓存有哪些类型？
缓存是高并发场景下提高热点数据访问性能的一个有效手段，在开发项目时会经常使用到。
缓存的类型分为：本地缓存、分布式缓存和多级缓存。

本地缓存：
本地缓存就是在进程的内存中进行缓存，比如我们的 JVM 堆中，可以用 LRUMap 来实现，也可以使用 Ehcache 这样的工具来实现。
本地缓存是内存访问，没有远程交互开销，性能最好，但是受限于单机容量，一般缓存较小且无法扩展。
分布式缓存：
分布式缓存可以很好得解决这个问题。
分布式缓存一般都具有良好的水平扩展能力，对较大数据量的场景也能应付自如。缺点就是需要进行远程请求，性能不如本地缓存。
多级缓存：
为了平衡这种情况，实际业务中一般采用多级缓存，本地缓存只保存访问频率最高的部分热点数据，其他的热点数据放在分布式缓存中。
在目前的一线大厂中，这也是最常用的缓存方案，单考单一的缓存方案往往难以撑住很多高并发的场景。

计数器和排行榜
incr命令 计数器 给值加1， 排行榜 sorted set
事务：
最后一个功能是事务，但 Redis 提供的不是严格的事务，Redis 只保证串行执行命令，并且能保证全部执行，但是执行命令失败时并不会回滚，而是会继续执行下去。
数据不一致
先更新数据库，再更新缓存 确保数据库正确，
缓存更新失败
如果服务对耗时不是特别敏感可以增加重试；如果服务对耗时敏感可以通过异步补偿任务来处理失败的更新，或者短期的数据不一致不会影响业务，那么只要下次更新时可以成功，能保证最终一致性就可以。

缓存穿透
缓存穿透。产生这个问题的原因可能是外部的恶意攻击，例如，对用户信息进行了缓存，但恶意攻击者使用不存在的用户id频繁请求接口，导致查询缓存不命中，然后穿透 DB 查询依然不命中。这时会有大量请求穿透缓存访问到 DB。
解决的办法如下。
1. 对不存在的用户，在缓存中保存一个空对象进行标记，防止相同 ID 再次访问 DB。不过有时这个方法并不能很好解决问题，可能导致缓存中存储大量无用数据。
2. 使用 BloomFilter 过滤器，BloomFilter 的特点是存在性检测，如果 BloomFilter 中不存在，那么数据一定不存在；如果 BloomFilter 中存在，实际数据也有可能会不存在。非常适合解决这类的问题。
缓存击穿
缓存击穿，就是某个热点数据失效时，大量针对这个数据的请求会穿透到数据源。
解决这个问题有如下办法。
1. 可以使用互斥锁更新，保证同一个进程中针对同一个数据不会并发请求到 DB，减小 DB 压力。
2. 使用随机退避方式，失效时随机 sleep 一个很短的时间，再次查询，如果失败再执行更新。
 缓存雪崩
针对多个热点 key 同时失效的问题，可以在缓存时使用固定时间加上一个小的随机数，避免大量热点 key 同一时刻失效。
提高集群的可用性

redis库存和mysql库存<br>
支付前是预扣，是扣redis库存，是锁定库存的过程 支付后是真正扣，扣mysql库存，保证库存最终一致<br>
仍然是sku:id为键，但是直接设置为库存余量N。当秒杀时，执行DECR 即可，当DECR返回值小于0时，即代表库存卖完了。
优点： 节约内存
补偿机制


