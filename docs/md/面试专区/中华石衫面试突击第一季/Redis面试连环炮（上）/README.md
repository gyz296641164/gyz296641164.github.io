<h1 align="center">Redis面试连环炮(上)</h1>





# 1、redis和memcached有啥区别

**采取redis作者给出的几个比较**

- Redis支持服务器端的数据操作：

  **Redis**相比Memcached来说，**拥有更多的数据结构和并支持更丰富的数据操作**，通常在Memcached里，你需要将数据拿到客户端来进行类似的修改再set回去。这大大增加了网络IO的次数和数据体积。在Redis中，这些复杂的操作通常和一般的GET/SET一样高效。所以，如果需要缓存能够支持更复杂的结构和操作，那么Redis会是不错的选择。

- 集群模式：

  memcached没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据；但是redis目前是原生支持cluster模式的，redis官方就是支持redis cluster集群模式的，比memcached来说要更好

- 内存使用效率对比：

  ~~使用简单的key-value存储的话，Memcached的内存利用率更高，而如果Redis采用hash结构来做key-value存储，由于其组合式的压缩，其内存利用率会高于Memcached。~~

- 性能对比：

  ~~由于Redis只使用单核，而Memcached可以使用多核，所以平均每一个核上Redis在存储小数据时比Memcached性能更高。而在100k以上的数据中，Memcached性能要高于Redis，虽然Redis最近也在存储大数据的性能上进行优化，但是比起Memcached，还是稍有逊色。~~

***

# 2、Redis的线程模型

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549216.png" alt="image-20210809225201552"/>

## 2.1 文件事件处理器

- redis基于reactor模式开发了网络事件处理器，这个处理器叫做**文件事件处理器**，`file event handler`。这个`文件事件处理器`是单线程的，redis才叫做单线程的模型，采用IO多路复用机制同时监听多个socket，根据socket上的事件来选择对应的事件处理器来处理这个事件。

- 如果被监听的socket准备好执行accept、read、write、close等操作的时候，跟操作对应的文件事件就会产生，这个时候`文件事件处理器`就会调用之前关联好的事件处理器来处理这个事件。

- 文件事件处理器是单线程模式运行的，但是通过IO多路复用机制监听多个socket，可以实现高性能的网络通信模型，又可以跟内部其他单线程的模块进行对接，保证了redis内部的线程模型的简单性。

**文件事件处理器的结构包含4个部分**：

- 多个socket
- IO多路复用程序
- 文件事件分派器
- 事件处理器（命令请求处理器、命令回复处理器、连接应答处理器，等等）

多个socket可能并发的产生不同的操作，每个操作对应不同的文件事件，但是IO多路复用程序会监听多个socket，但是会将socket放入一个队列中排队，每次从队列中取出一个socket给`事件分派器`，`事件分派器`把`socket`给对应的`事件处理器`。

然后一个socket的事件处理完之后，IO多路复用程序才会将队列中的下一个socket给事件分派器。文件事件分派器会根据每个socket当前产生的事件，来选择对应的事件处理器来处理。



## 2.2 文件事件

- 当socket变得可读时（比如客户端对redis执行write操作，或者close操作），或者有新的可以应答的sccket出现时（客户端对redis执行connect操作），socket就会产生一个`AE_READABLE`事件。

- 当socket变得可写的时候（客户端对redis执行read操作），socket会产生一个`AE_WRITABLE`事件。
- IO多路复用程序可以同时监听`AE_REABLE`和`AE_WRITABLE`两种事件，要是一个socket同时产生了`AE_READABLE`和`AE_WRITABLE`两种事件，那么**文件事件分派器优先处理`AE_REABLE`事件，然后才是`AE_WRITABLE`事件**。

 

## 2.3 文件事件分派器

- 如果是客户端要连接redis，那么会为socket关联`连接应答处理器`
- 如果是客户端要写数据到redis，那么会为socket关联`命令请求处理器`
- 如果是客户端要从redis读数据，那么会为socket关联`命令回复处理器`



## 2.4 客户端与redis通信的一次流程

- 在redis启动初始化的时候，redis会将连接应答处理器跟AE_READABLE事件关联起来，接着如果一个客户端跟redis发起连接，此时会产生一个AE_READABLE事件，然后由连接应答处理器来处理跟客户端建立连接，创建客户端对应的socket，同时将这个socket的AE_READABLE事件跟命令请求处理器关联起来。
- 当客户端向redis发起请求的时候（不管是读请求还是写请求，都一样），首先就会在socket产生一个AE_READABLE事件，然后由对应的命令请求处理器来处理。这个命令请求处理器就会从socket中读取请求相关数据，然后进行执行和处理。
- 接着redis这边准备好了给客户端的响应数据之后，就会将socket的AE_WRITABLE事件跟命令回复处理器关联起来，当客户端这边准备好读取响应数据时，就会在socket上产生一个AE_WRITABLE事件，会由对应的命令回复处理器来处理，就是将准备好的响应数据写入socket，供客户端来读取。
- 命令回复处理器写完之后，就会删除这个socket的AE_WRITABLE事件和命令回复处理器的关联关系。



***

# 3、为啥redis单线程模型也能效率这么高



1. 纯内存操作
2. 核心是基于非阻塞的IO多路复用机制
3. 单线程反而避免了多线程的频繁上下文切换问题



***

# 4、redis数据类型及使用场景

**string**

-  这是最基本的类型了，就是普通的set和get，做简单的kv缓存

**hash**

- 这个是类似map的一种结构，这个一般就是可以将结构化的数据，比如一个对象（前提是这个对象没嵌套其他的对象）给缓存在redis里，然后每次读写缓存的时候，可以就操作hash里的某个字段。

  ```
  key=150
  
  value={
    “id”: 150,
    “name”: “zhangsan”,
    “age”: 20
  }
  ```

- hash类的数据结构，主要是用来存放一些对象，把一些简单的对象给缓存起来，后续操作的时候，你可以直接仅仅修改这个对象中的某个字段的值

  ```
  value={
    “id”: 150,
    “name”: “zhangsan”,
    “age”: 21
  }
  ```

**list**

- 有序列表，微博，某个大v的粉丝，就可以以list的格式放在redis里去缓存

  ```
  key=某大v
  
  value=[zhangsan, lisi, wangwu]
  ```

- 比如可以通过`lrange`命令，就是从某个元素开始读取多少个元素，可以基于list实现分页查询，这个很棒的一个功能，基于redis实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走

- 比如可以搞个简单的消息队列，从list头怼进去，从list尾巴那里弄出来

**set**

- 无序集合，自动去重
- 直接基于set将系统里需要去重的数据扔进去，自动就给去重了，如果你需要对一些数据进行快速的全局去重，你当然也可以基于jvm内存里的HashSet进行去重，但是如果你的某个系统部署在多台机器上呢？得基于redis进行全局的set去重
- 可以基于set玩儿交集、并集、差集的操作，比如交集吧，可以把两个人的粉丝列表整一个交集，看看俩人的共同好友是谁，把两个大v的粉丝都放在两个set中，对两个set做交集

**sorted set**

- 排序的set，去重但是可以排序，写进去的时候给一个分数，自动根据分数排序，这个可以玩儿很多的花样，最大的特点是有个分数可以自定义排序规则

- 比如说你要是想根据时间对数据排序，那么可以写入进去的时候用某个时间作为分数，人家自动给你按照时间排序了

- 排行榜：将每个用户以及其对应的什么分数写入进去，zadd board score username，接着zrevrange board 0 99，就可以获取排名前100的用户；zrank board username，可以看到用户在排行榜里的排名

  ```
  zadd board 85 zhangsan
  zadd board 72 wangwu
  zadd board 96 lisi
  zadd board 62 zhaoliu
  
  96 lisi
  85 zhangsan
  72 wangwu
  62 zhaoliu
  
  zrevrange board 0 3
  
  获取排名前3的用户
  
  96 lisi
  85 zhangsan
  72 wangwu
  
  zrank board zhaoliu
  ```

  

***

# 5、redis的过期策略及LRU代码实现



## 5.1 往redis里写的数据怎么没了

- redis是缓存，不要当存储！

- 啥叫缓存？**用内存当缓存**。内存是无限的吗，**内存是很宝贵而且是有限的**，磁盘是廉价而且是大量的。可能一台机器就几十个G的内存，但是可以有几个T的硬盘空间。redis主要是`基于内存来进行高性能、高并发的读写操作的`。

- 既然内存是有限的，比如redis就只能用10个G，你要是往里面写了20个G的数据，会咋办？当然会干掉10个G的数据，然后就保留10个G的数据了。那干掉哪些数据？保留哪些数据？当然是`干掉不常用的数据`，`保留常用的数据了`。

- 所以说，这是缓存的一个最基本的概念，数据是会过期的，要么是你自己设置个过期时间，要么是redis自己给干掉。

  ```
  set key value 过期时间（1小时）
  set进去的key，1小时之后就没了，就失效了
  ```



## 5.2 数据明明都过期了，怎么还占用着内存啊

**问题**

- 如果你设置好了一个过期时间，为啥好多数据明明应该过期了，结果发现redis内存占用还是很高？那是因为你不知道`redis是怎么删除那些过期key的`。
- redis 内存一共是10g，你现在往里面写了5g的数据，结果这些数据明明你都设置了过期时间，要求这些数据1小时之后都会过期，结果1小时之后，你回来一看，redis机器，怎么内存占用还是50%呢？5g数据过期了，我从redis里查，是查不到了，结果过期的数据还占用着redis的内存。

**面试题剖析**

（1）**设置过期时间**

- 我们set key的时候，都可以给一个expire time，就是过期时间，指定这个key比如说只能存活1个小时？10分钟？这个很有用，我们自己可以指定缓存到期就失效。
- 如果假设你设置一个一批key只能存活1个小时，那么接下来1小时后，redis是怎么对这批key进行删除的？答案是：**定期删除** + **惰性删除**

- **所谓定期删除**
  - 指的是redis默认是`每隔100ms`就`随机抽取一些设置了过期时间的key`，检查其是否过期，如果过期就删除。假设redis里放了10万个key，都设置了过期时间，你每隔几百毫秒，就检查10万个key，那redis基本上就死了，cpu负载会很高的，消耗在你的检查过期key上了。**注意**，这里可不是每隔100ms就遍历所有的设置过期时间的key，那样就是一场性能上的灾难。实际上redis是每隔100ms随机抽取一些key来检查和删除的。
  - 但是问题是，定期删除可能会导致很多过期key到了时间并没有被删除掉，那咋整呢？所以就是惰性删除了。

- **惰性删除**
  - 这就是说，在你获取某个key的时候，redis会检查一下 ，这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除，不会给你返回任何东西。
  - 并不是key到时间就被删除掉，而是你查询这个key的时候，redis再懒惰的检查一下！

- 通过上述两种手段结合起来，保证过期的key一定会被干掉。

- 很简单，就是说，你的过期key，靠定期删除没有被删除掉，还停留在内存里，占用着你的内存呢，除非你的系统去查一下那个key，才会被redis给删除掉。

- 但是实际上这还是有问题的，如果定期删除漏掉了很多过期key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期key堆积在内存里，导致redis内存块耗尽了，咋整？

  答案是：**走内存淘汰机制**。

（2）**内存淘汰**

如果redis的内存占用过多的时候，此时会进行内存淘汰，有如下一些策略：

```
redis 10个key，现在已经满了，redis需要删除掉5个key

1个key，最近1分钟被查询了100次

1个key，最近10分钟被查询了50次

1个key，最近1个小时被查询了1次
```

- `noeviction`：当内存不足以容纳新写入数据时，新写入操作会报错，这个一般没人用吧，实在是太恶心了
- `allkeys-lru`：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key（这个是最常用的）
- `allkeys-random`：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key，这个一般没人用吧，为啥要随机，肯定是把最近最少使用的key给干掉啊
- `volatile-lru`：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key（这个一般不太合适）
- `volatile-random`：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key
- `volatile-ttl`：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除

很简单，你写的数据太多，内存满了，或者触发了什么条件，`redis lru`自动给你清理掉了一些最近很少使用的数据。



## 5.3 手写一个LRU算法

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    
private final int CACHE_SIZE;

    // 这里就是传递进来最多能缓存多少数据
    public LRUCache(int cacheSize) {
    // 这块就是设置一个hashmap的初始大小，同时最后一个true指的是让linkedhashmap按照访问顺序来进行排序，最近访问的放在头，最老访问的就在尾
        super((int) Math.ceil(cacheSize / 0.75) + 1, 0.75f, true); 
        CACHE_SIZE = cacheSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > CACHE_SIZE; // 这个意思就是说当map中的数据量大于指定的缓存个数的时候，就自动删除最老的数据
    }
}    
```



***

# 6、怎么保证redis是高并发以及高可用的



## 6.1 redis如何通过读写分离来承载读请求QPS超过10万+

**redis高并发跟整个系统的高并发之间的关系**

- redis，你要搞高并发的话，不可避免，要把底层的缓存搞得很好
- mysql，高并发，做到了，那么也是通过一系列复杂的分库分表，订单系统，事务要求的，QPS到几万，比较高了
- 要做一些电商的商品详情页，真正的超高并发，QPS上十万，甚至是百万，一秒钟百万的请求量，光是redis是不够的，但是redis是整个大型的缓存架构中，支撑高并发的架构里面，非常重要的一个环节
- 首先，你的底层的缓存中间件，缓存系统，必须能够支撑的起我们说的那种高并发，其次，再经过良好的整体的缓存架构的设计（多级缓存架构、热点缓存），支撑真正的上十万，甚至上百万的高并发

**redis不能支撑高并发的瓶颈在哪里**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549217.png" alt="redis单机的瓶颈" />

- 因为单机的Redis，QPS只能在上万左右，成为了支撑高并发的瓶颈。

**如果redis要支撑超过10万+的并发，那应该怎么做**

- 单机的redis几乎不太可能说QPS超过10万+，除非一些特殊情况，比如你的机器性能特别好，配置特别高，物理机，维护做的特别好，而且你的整体的操作不是太复杂，单机在几万

- **读写分离**

  - 一般来说，对缓存，一般都是用来支撑读高并发的，写的请求是比较少的，可能写请求也就一秒钟几千，一两千大量的请求都是读，一秒钟二十万次读
  - 主从架构 -> 读写分离 -> 支撑10万+读QPS的架构

- <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549218.png" alt="redis主从实现读写分离支撑10万+的高并发" />

- 架构做成主从架构，一主多从，主服务器负责写，并且将数据同步到其它的slave节点，从节点负责读，所有的读请求全部走节点。

- 同时这样的架构，支持碎片扩容，就是说如果QPS在增加，也很简单，只需要增加 Redis Slave节点即可。

  

## 6.2 redis replication以及master持久化对主从架构的安全意义

第一：

- 如果采用了主从架构，那么建议必须开启master node的持久化！
- 不建议用slave node作为master node的数据热备，因为那样的话，如果你关掉master的持久化，可能在master宕机重启的时候数据是空的，然后可能一经过复制，salve node数据也丢了
- master -> RDB和AOF都关闭了 -> 全部在内存中
- master宕机，重启，是没有本地数据可以恢复的，然后就会直接认为自己IDE数据是空的
- master就会将空的数据集同步到slave上去，所有slave的数据全部清空，100%的数据丢失
- master节点，必须要使用持久化机制

第二：

- master的各种备份方案，要不要做，万一说本地的所有文件丢失了; 从备份中挑选一份rdb去恢复master; 这样才能确保master启动的时候，是有数据的
- 即使采用了后续讲解的高可用机制，slave node可以自动接管master node，但是也可能sentinal还没有检测到`master failure`，`master node`就自动重启了，还是可能导致上面的所有slave node数据清空故障



## 6.3 redis主从复制原理、断点续传、无磁盘化复制、过期key处理

**主从复制原理**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549219.png" alt="redis主从复制的原理" />

- 当启动一个slave node的时候，它会发送一个PSYNC命令给master node
- 如果这是slave node重新连接master node，那么master node仅仅会复制给slave部分缺少的数据; 否则如果是slave node第一次连接master node，那么会触发一次full resynchronization
- 开始full resynchronization的时候，master会启动一个后台线程，开始生成一份RDB快照文件，同时还会将从客户端收到的所有写命令缓存在内存中。RDB文件生成完毕之后，master会将这个RDB发送给slave，slave会先写入本地磁盘，然后再从本地磁盘加载到内存中。然后master会将内存中缓存的写命令发送给slave，slave也会同步这些数据。
- slave node如果跟master node有网络故障，断开了连接，会自动重连。master如果发现有多个slave node都来重新连接，仅仅会启动一个rdb save操作，用一份数据服务所有slave node。

**断点续传**

- 从redis 2.8开始，就支持主从复制的断点续传，如果主从复制过程中，网络连接断掉了，那么可以接着上次复制的地方，继续复制下去，而不是从头开始复制一份
- `master node`会在内存中常见一个`backlog`，master和slave都会保存一个`replica offset`还有一个`master id`，`offset`就是保存在backlog中的。如果master和slave网络连接断掉了，slave会让master从上次的replica offset开始继续复制
- 但是如果没有找到对应的offset，那么就会执行一次`resynchronization`

**无磁盘化复制**

- master在内存中直接创建rdb，然后发送给slave，不会在自己本地落地磁盘了

- ```
  repl-diskless-sync
  repl-diskless-sync-delay，等待一定时长再开始复制，因为要等更多slave重新连接过来
  ```

**过期key处理**

- slave不会过期key，只会等待master过期key。如果master过期了一个key，或者通过LRU淘汰了一个key，那么会模拟一条del命令发送给slave。



## 6.4 redis replication的完整流运行程和原理的再次深入剖析

**Redis主从复制的完整流程**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549220.png" alt="复制的完整的基本流程" />

- `slave node`启动，仅仅保存`master node`的信息，包括`master node`的`host`和`ip`，但是复制流程没开始，master host 和 ip是从哪来的？`redis.conf`里面的`slaveof`配置的
- `slave node`内部有个定时任务，每秒检查是否有新的master node要连接和复制，如果发现，就跟master node建立socket网络连接
- slave node发送ping命令给master node
- 口令认证，如果master设置了requirepass，那么salve node必须发送masterauth的口令过去进行认证
- master node第一次执行全量复制，将所有数据发给slave node
- master node后续持续将写命令，异步复制给slave node

**数据同步相关的核心机制**

指的就是第一次slave连接msater的时候，执行的全量复制，那个过程里面你的一些细节的机制：

1. master和slave都会维护一个offset

   - master会在自身不断累加offset，slave也会在自身不断累加offset。slave每秒都会上报自己的offset给master，同时master也会保存每个slave的offset。

   - 这个倒不是说特定就用在全量复制的，主要是master和slave都要知道各自的数据的offset，才能知道互相之间的数据不一致的情况

2. backlog

   - master node有一个backlog，默认是1MB大小
   - master node给slave node复制数据时，也会将数据在backlog中同步写一份
   - backlog主要是用来做全量复制中断候的增量复制的

3. master run id

   - `info server`，可以看到`master run id`

   - 如果根据`host+ip`定位master node，是不靠谱的，如果master node重启或者数据出现了变化，那么slave node应该根据不同的run id区分，run id不同就做全量复制如果需要不更改run id重启redis，可以使用`redis-cli debug reload`命令

   - `master run id` 的作用

     <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549221.png" alt="maste run id的作用" />

4. psync

   - 从节点使用psync从master node进行复制，`psync runid offset`
   - master node会根据自身的情况返回响应信息，可能是`FULLRESYNC runid offset`触发全量复制，可能是`CONTINUE`触发增量复制

**全量复制**

- master执行bgsave，在本地生成一份rdb快照文件
- master node将rdb快照文件发送给salve node，如果rdb复制时间超过60秒（repl-timeout），那么slave node就会认为复制失败，可以适当调节大这个参数
- 对于千兆网卡的机器，一般每秒传输100MB，6G文件，很可能超过60s
- master node在生成rdb时，会将所有新的写命令缓存在内存中，在salve node保存了rdb之后，再将新的写命令复制给salve node
- client-output-buffer-limit slave 256MB 64MB 60，如果在复制期间，内存缓冲区持续消耗超过64MB，或者一次性超过256MB，那么停止复制，复制失败
- slave node接收到rdb之后，清空自己的旧数据，然后重新加载rdb到自己的内存中，同时基于旧的数据版本对外提供服务（）
- 如果slave node开启了AOF，那么会立即执行`BGREWRITEAOF`，`重写AOF`

rdb生成、rdb通过网络拷贝、slave旧数据的清理、slave aof rewrite，很耗费时间。

如果复制的数据量在4G~6G之间，那么很可能全量复制时间消耗到1分半到2分钟。

**增量复制**

- 如果全量复制过程中，master-slave网络连接断掉，那么salve重新连接master时，会触发增量复制
- master直接从自己的backlog中获取部分丢失的数据，发送给slave node，默认backlog就是1MB
- msater就是根据slave发送的psync中的offset来从backlog中获取数据的

**heartbeat（心跳机制）**

- 主从节点互相都会发送heartbeat信息
- master默认每隔10秒发送一次heartbeat，salve node每隔1秒发送一个heartbeat

**异步复制**

- master每次接收到写命令之后，先在内部写入数据，然后异步发送给slave node



## 6.5 redis主从架构下如何才能做到99.99%的高可用性

> **什么是99.99%高可用？**

架构上，99.99%的高可用性！就可称为高可用

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549222.png" alt="什么是99.99%高可用性"/>

> **系统处于不可用是什么意思？**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549223.png" alt="系统处于不可用是什么意思" />

> **redis不可用是什么？**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549224.png" alt="redis的不可用" />

> **redis高可用的方案？**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549225.png" alt="redis基于哨兵的高可用性" />

>  **总结**

- Redis实现高并发：一主多从，一般来说，很多项目其实就足够了，单主用来写数据，单机几万QPS，多从用来查询数据，多个从实例可以提供每秒10万QPS
- Redis高并发的同时，还需要容纳大量的数据：一主多从，每个实例都容纳了完整的数据，比如Redis主就10G内存量，其实你就可以对只能容纳10G的数据量。如果你的缓存要容纳的数据量很大，达到了几十G，甚至几百G，那就需要使用到Redis集群，而且用Redis集群之后，提供可能每秒几十万的读写并发
- Redis高可用：如果用主从架构部署，在加上哨兵就可以实现任何一个实例宕机，就会自动进行主备切换



## 6.6 redis哨兵架构的相关基础知识

> **哨兵的介绍**

`sentinal`，中文名是哨兵

哨兵是redis集群架构中非常重要的一个组件，主要功能如下：

- `集群监控`，负责监控redis master和slave进程是否正常工作
- `消息通知`，如果某个redis实例有故障，那么哨兵负责发送消息作为报警通知给管理员
- `故障转移`，如果master node挂掉了，会自动转移到slave node上
- `配置中心`，如果故障转移发生了，通知client客户端新的master地址

哨兵本身也是分布式的，作为一个哨兵集群去运行，互相协同工作

- 故障转移时，判断一个master node是宕机了，需要大部分的哨兵都同意才行，涉及到了分布式选举的问题
- 即使部分哨兵节点挂掉了，哨兵集群还是能正常工作的，因为如果一个作为高可用机制重要组成部分的故障转移系统本身是单点的，那就很坑了



> **哨兵的核心知识**

- 哨兵**至少需要3个实例**，来保证自己的健壮性

- **哨兵 + redis主从的部署架构，是不会保证数据零丢失的，只能保证redis集群的高可用性**

- 对于哨兵 + redis主从这种复杂的部署架构，尽量在测试环境和生产环境，都进行充足的测试和演练



> **为什么redis哨兵集群只有2个节点无法正常工作**

哨兵集群必须部署2个以上节点。如果哨兵集群仅仅部署了个2个哨兵实例，`quorum=1`

```
+----+         +----+
| M1 |---------| R1 |
| S1 |         | S2 |
+----+         +----+
```

`Configuration: quorum = 1`

`quorum的解释如下：`

1. `至少多少个哨兵要一致同意，master进程挂掉了，或者slave进程挂掉了，或者要启动一个故障转移操作 `
2. `quorum是用来识别故障的，真正执行故障转移的时候，还是要在哨兵集群执行选举，选举一个哨兵进程出来执行故障转移操作 `
3. `假设有5个哨兵，quorum设置了2，那么如果5个哨兵中的2个都认为master挂掉了; 2个哨兵中的一个就会做一个选举，选举一个哨兵出来，执行故障转移; 如果5个哨兵中有3个哨兵都是运行的，那么故障转移就会被允许执行`

master宕机，s1和s2中只要有1个哨兵认为master宕机就可以还行切换，同时s1和s2中会选举出一个哨兵来执行故障转移

同时这个时候，需要`majority`，也就是大多数哨兵都是运行的，`2个哨兵的majority就是2`（2的majority=2，3的majority=2，5的majority=3，4的majority=2），2个哨兵都运行着，就可以允许执行故障转移

但是如果整个M1和S1运行的机器宕机了，那么哨兵只有1个了，此时就没有majority来允许执行故障转移，虽然另外一台机器还有一个R1，但是故障转移不会执行



> **经典的3节点哨兵集群**

```
       +----+
       | M1 |
       | S1 |
       +----+
          |
+----+    |    +----+
| R2 |----+----| R3 |
| S2 |         | S3 |
+----+         +----+
```

- `Configuration: quorum = 2，majority`

- 如果M1所在机器宕机了，那么三个哨兵还剩下2个，S2和S3可以一致认为master宕机，然后选举出一个来执行故障转移

- 同时3个哨兵的majority是2，所以还剩下的2个哨兵运行着，就可以允许执行故障转移



## 6.7 redis哨兵主备切换的数据丢失问题：异步复制、集群脑裂

> **两种数据丢失的情况**

主备切换的过程，可能会导致数据丢失

- 异步复制导致的数据丢失

  <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549226.png" alt="异步复制导致的数据丢失问题" />

  - 因为master -> slave的复制是异步的，所以可能有部分数据还没复制到slave，master就宕机了，此时这些部分数据就丢失

- 脑裂导致的数据丢失

  <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549227.png" alt="集群脑裂导致的数据丢失问题" />

  - 脑裂，也就是说，某个master所在机器突然脱离了正常的网络，跟其他slave机器不能连接，但是实际上master还运行着。此时哨兵可能就会认为master宕机了，然后开启选举，将其他slave切换成了master。这个时候，集群里就会有两个master，也就是所谓的脑裂
  - 此时虽然某个slave被切换成了master，但是可能client还没来得及切换到新的master，还继续写向旧master的数据可能也丢失了。因此旧master再次恢复的时候，会被作为一个slave挂到新的master上去，自己的数据会清空，重新从新的master复制数据

> **解决异步复制和脑裂导致的数据丢失**

```
min-slaves-to-write 1
min-slaves-max-lag 10
```

- 要求至少有1个slave，数据复制和同步的延迟不能超过10秒

- 如果说一旦所有的slave，数据复制和同步的延迟都超过了10秒钟，那么这个时候，master就不会再接收任何请求了

- 上面两个配置可以减少异步复制和脑裂导致的数据丢失

  1. 减少异步复制的数据丢失

     有了min-slaves-max-lag这个配置，就可以确保说，一旦slave复制数据和ack延时太长，就认为可能master宕机后损失的数据太多了，那么就拒绝写请求，这样可以把master宕机时由于部分数据未同步到slave导致的数据丢失降低的可控范围内

     <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549228.png" alt="异步复制导致数据丢失如何降低损失" />

  2. 减少脑裂的数据丢失

     如果一个master出现了脑裂，跟其他slave丢了连接，那么上面两个配置可以确保说，如果不能继续给指定数量的slave发送数据，而且slave超过10秒没有给自己ack消息，那么就直接拒绝客户端的写请求这样脑裂后的旧master就不会接受client的新数据，也就避免了数据丢失；

     上面的配置就确保了，如果跟任何一个slave丢了连接，在10秒后发现没有slave给自己ack，那么就拒绝新的写请求。因此在脑裂场景下，最多就丢失10秒的数据。

     <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549229.png" alt="脑裂导致数据丢失的问题如何降低损失" />



## 6.8 redis哨兵的多个核心底层原理的深入解析（包含slave选举算法）



> **sdown和odown转换机制**

- sdown和odown两种失败状态

- sdown是主观宕机，就一个哨兵如果自己觉得一个master宕机了，那么就是主观宕机

- odown是客观宕机，如果quorum数量的哨兵都觉得一个master宕机了，那么就是客观宕机

- sdown达成的条件很简单，如果一个哨兵ping一个master，超过了`is-master-down-after-milliseconds`指定的毫秒数之后，就主观认为master宕机

- sdown到odown转换的条件很简单，如果一个哨兵在指定时间内，收到了quorum指定数量的其他哨兵也认为那个master是sdown了，那么就认为是odown了，客观认为master宕机



> **哨兵集群的自动发现机制**

- 哨兵互相之间的发现，是**通过redis的pub/sub系统实现**的，每个哨兵都会往__sentinel__:hello这个channel里发送一个消息，这时候所有其他哨兵都可以消费到这个消息，并感知到其他的哨兵的存在

- 每隔两秒钟，每个哨兵都会往自己监控的某个master+slaves对应的__sentinel__:hello channel里发送一个消息，内容是自己的host、ip和runid还有对这个master的监控配置

- 每个哨兵也会去监听自己监控的每个master+slaves对应的__sentinel__:hello channel，然后去感知到同样在监听这个master+slaves的其他哨兵的存在

- 每个哨兵还会跟其他哨兵交换对master的监控配置，互相进行监控配置的同步



> **slave配置的自动纠正**

- 哨兵会负责自动纠正slave的一些配置，比如slave如果要成为潜在的master候选人，哨兵会确保slave在复制现有master的数据; 如果slave连接到了一个错误的master上，比如故障转移之后，那么哨兵会确保它们连接到正确的master上



> **slave->master选举算法**

- 如果一个master被认为odown了，而且majority哨兵都允许了主备切换，那么某个哨兵就会执行主备切换操作，此时首先要选举一个slave来

- 会考虑slave的一些信息

  - 跟master断开连接的时长
  - slave优先级
  - 复制offset
  - run id

- 如果一个slave跟master断开连接已经超过了`down-after-milliseconds`的10倍，外加master宕机的时长，那么slave就被认为不适合选举为master

  ```
  (down-after-milliseconds * 10) + milliseconds_since_master_is_in_SDOWN_state
  ```

- 接下来会对slave进行排序

  - 按照slave优先级进行排序，slave priority越低，优先级就越高
  - 如果slave priority相同，那么看replica offset，哪个slave复制了越多的数据，offset越靠后，优先级就越高
  - 如果上面两个条件都相同，那么选择一个run id比较小的那个slave



> **quorum和majority**

- 每次一个哨兵要做主备切换，首先需要quorum数量的哨兵认为odown，然后选举出一个哨兵来做切换，这个哨兵还得得到majority哨兵的授权，才能正式执行切换
- 如果quorum < majority，比如5个哨兵，majority就是3，quorum设置为2，那么就3个哨兵授权就可以执行切换
- 但是如果quorum >= majority，那么必须quorum数量的哨兵都授权，比如5个哨兵，quorum是5，那么必须5个哨兵都同意授权，才能执行切换



> **configuration epoch**

- 哨兵会对一套redis master+slave进行监控，有相应的监控的配置
- 执行切换的那个哨兵，会从要切换到的新master（salve->master）那里得到一个configuration epoch，这就是一个version号，每次切换的version号都必须是唯一的
- 如果第一个选举出的哨兵切换失败了，那么其他哨兵，会等待failover-timeout时间，然后接替继续执行切换，此时会重新获取一个新的configuration epoch，作为新的version号



> **configuraiton传播**

- 哨兵完成切换之后，会在自己本地更新生成最新的master配置，然后同步给其他的哨兵，就是通过之前说的pub/sub消息机制
- 这里之前的version号就很重要了，因为各种消息都是通过一个channel去发布和监听的，所以一个哨兵完成一次新的切换之后，新的master配置是跟着新的version号的，其他的哨兵都是根据版本号的大小来更新自己的master配置的



***

# 7、怎么保证redis挂掉之后再重启数据可以进行恢复



## 7.1 redis持久化机对于生产环境中的灾难恢复的意义

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549230.png" alt="image-20210811235828883" />

- Redis持久化的意义，在于故障恢复，也属于高可用的一个环节。例如：当存放在内存中数据，会因为Redis的突然挂掉，而导致数据丢失。

- Redis的持久化，就是将内存中的数据，持久化到磁盘上中，然后将磁盘上的数据放到阿里云ODPS中

- 通过持久化将数据存储在磁盘中，然后定期比如说同步和备份到一些云存储服务上去。



## 7.2 redis的RDB和AOF两种持久化机制的工作原理

> **RDB和AOF两种持久化机制的介绍**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549231.png" alt="RDB和AOF的介绍" />

- RDB持久化机制，对redis中的数据执行周期性的持久化
- AOF机制对每条写入命令作为日志，以append-only的模式写入一个日志文件中，在redis重启的时候，可以通过回放AOF日志中的写入指令来重新构建整个数据集
- 如果我们想要redis仅仅作为纯内存的缓存来用，那么可以禁止RDB和AOF所有的持久化机制
- 通过RDB或AOF，都可以将redis内存中的数据给持久化到磁盘上面来，然后可以将这些数据备份到别的地方去，比如说阿里云，云服务
- 如果redis挂了，服务器上的内存和磁盘上的数据都丢了，可以从云服务上拷贝回来之前的数据，放到指定的目录中，然后重新启动redis，redis就会自动根据持久化数据文件中的数据，去恢复内存中的数据，继续对外提供服务
- 如果同时使用RDB和AOF两种持久化机制，那么在redis重启的时候，会使用AOF来重新构建数据，因为AOF中的数据更加完整



> **RDB持久化机制的优点**

- RDB会生成多个数据文件，每个数据文件都代表了某一个时刻中redis的数据，这种多个数据文件的方式，非常适合做冷备，可以将这种完整的数据文件发送到一些远程的安全存储上去，比如说Amazon的S3云服务上去，在国内可以是阿里云的ODPS分布式存储上，以预定好的备份策略来定期备份redis中的数据
- RDB对redis对外提供的读写服务，影响非常小，可以让redis保持高性能，因为redis主进程只需要fork一个子进程，让子进程执行磁盘IO操作来进行RDB持久化即可
- 相对于AOF持久化机制来说，直接基于RDB数据文件来重启和恢复redis进程，更加快速



> **RDB持久化机制的缺点**

- 如果想要在redis故障时，尽可能少的丢失数据，那么RDB没有AOF好。一般来说，RDB数据快照文件，都是每隔5分钟，或者更长时间生成一次，这个时候就得接受一旦redis进程宕机，那么会丢失最近5分钟的数据

- RDB每次在fork子进程来执行RDB快照数据文件生成的时候，如果数据文件特别大，可能会导致对客户端提供的服务暂停数毫秒，或者甚至数秒



> **AOF持久化机制的优点**

- AOF可以更好的保护数据不丢失，一般AOF会每隔1秒，通过一个后台线程执行一次fsync操作，最多丢失1秒钟的数据
- AOF日志文件以append-only模式写入，所以没有任何磁盘寻址的开销，写入性能非常高，而且文件不容易破损，即使文件尾部破损，也很容易修复
- AOF日志文件即使过大的时候，出现后台重写操作，也不会影响客户端的读写。因为在rewrite log的时候，会对其中的指导进行压缩，创建出一份需要恢复数据的最小日志出来。再创建新日志文件的时候，老的日志文件还是照常写入。当新的merge后的日志文件ready的时候，再交换新老日志文件即可。
- AOF日志文件的命令通过非常可读的方式进行记录，这个特性非常适合做灾难性的误删除的紧急恢复。比如某人不小心用flushall命令清空了所有数据，只要这个时候后台rewrite还没有发生，那么就可以立即拷贝AOF文件，将最后一条flushall命令给删了，然后再将该AOF文件放回去，就可以通过恢复机制，自动恢复所有数据



> **AOF Rewrite 原理剖析**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/audition/202211081549232.png" alt="AOF rewrite原理剖析" />



> **AOF持久化机制的缺点**

- 对于同一份数据来说，AOF日志文件通常比RDB数据快照文件更大

- AOF开启后，支持的写QPS会比RDB支持的写QPS低，因为AOF一般会配置成每秒fsync一次日志文件，当然，每秒一次fsync，性能也还是很高的

- 以前AOF发生过bug，就是通过AOF记录的日志，进行数据恢复的时候，没有恢复一模一样的数据出来。所以说，类似AOF这种较为复杂的基于命令日志/merge/回放的方式，比基于RDB每次持久化一份完整的数据快照文件的方式，更加脆弱一些，容易有bug。不过AOF就是为了避免rewrite过程导致的bug，因此每次rewrite并不是基于旧的指令日志进行merge的，而是基于当时内存中的数据进行指令的重新构建，这样健壮性会好很多。



> **RDB和AOF到底该如何选择**

- 不要仅仅使用RDB，因为那样会导致你丢失很多数据

- 也不要仅仅使用AOF，因为那样有两个问题，第一，你通过AOF做冷备，没有RDB做冷备，来的恢复速度更快; 第二，RDB每次简单粗暴生成数据快照，更加健壮，可以避免AOF这种复杂的备份和恢复机制的bug

- 综合使用AOF和RDB两种持久化机制，用AOF来保证数据不丢失，作为数据恢复的第一选择; 用RDB来做不同程度的冷备，在AOF文件都丢失或损坏不可用的时候，还可以使用RDB来进行快速的数据恢复

