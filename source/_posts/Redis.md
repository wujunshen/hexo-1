---
title: Redis
excerpt: Redis相关
categories:
- 中间件
tags:
- 中间件
- Redis
comments: true
layout: post
index_img: /img/jetbrain/1920x1080-rubymine2022_1.png
abbrlink: 3135569683
date: 2022-05-21 21:23:30
sticky: 70
---
# 基础

`redis`数据类型

1. `string`
2. `hash`
3. `list`
4. `set`
5. `sorted set`

## string

最简单类型，普通的`set`和`get`，做简单的`KV`缓存。

``` log
set name wjs
```

## hash

类似`map`，可以将结构化的数据，比如一个对象（前提是这个对象没嵌套其他的对象）给缓存在`redis`里，然后每次读写缓存的时候，可以就操作`hash`里的某个字段。

``` log
hset person name wjs
hset person age 20
hset person id 1
hget person name
person = {
    "name": "wjs",
    "age": 20,
    "id": 1
}
```

## list
有序列表，通过`list`存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的东西

通过`lrange`命令，读取某个闭区间内的元素，基于`list`实现分页查询，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走。

``` log
# 0开始位置，-1结束位置，结束位置为-1时，表示列表的最后一个位置，即查看所有。
lrange mylist 0 -1
```

也可以搞个简单的消息队列，从`list`头放进去，从`list`尾巴里出来。

``` log
lpush mylist 1
lpush mylist 2
lpush mylist 3 4 5

# 1
rpop mylist
```

## set

无序集合，自动去重

直接基于`set`将系统里需要去重的数据扔进去，自动就给去重，如果需要对一些数据进行快速的全局去重，当然可以基于`HashSet`进行去重，但如果系统部署在多台机器上，得基于`redis`进行全局的`set`去重。

基于`set`玩交集、并集、差集的操作

``` log
#-------操作一个set-------
# 添加元素
sadd mySet 1

# 查看全部元素
smembers mySet

# 判断是否包含某个值
sismember mySet 3

# 删除某个/些元素
srem mySet 1
srem mySet 2 4

# 查看元素个数
scard mySet

# 随机删除一个元素
spop mySet

#-------操作多个set-------
# 将一个set的元素移动到另外一个set
smove yourSet mySet 2

# 求两set的交集
sinter yourSet mySet

# 求两set的并集
sunion yourSet mySet

# 求在yourSet中而不在mySet中的元素
sdiff yourSet mySet
```

## sorted set
排序`set`，去重且可以排序，写进去的时候给一个分数，自动根据分数排序。

``` log
zadd board 85 zhangsan
zadd board 72 lisi
zadd board 96 wangwu
zadd board 63 zhaoliu

# 获取排名前三的用户（默认是升序，所以需要 rev 改为降序）
zrevrange board 0 3

# 获取某用户的排名
zrank board zhaoliu
```

# 线程模型
`redis`内部使用文件事件处理器`file event handler`，它是单线程的，所以`redis`才叫做单线程的模型。

采用`IO`多路复用机制同时监听多个`socket`，将产生事件的`socket`压入内存队列中，事件分派器根据`socket`上的事件类型来选择对应的事件处理器进行处理

`file event handler`结构:

1. 多个`socket`
2. `IO`多路复用程序
3. 文件事件分派器
4. 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）

多个`socket`可能会并发产生不同的操作，每个操作对应不同的文件事件，但是`IO`多路复用程序会监听多个`socket`，会将产生事件的`socket`放入队列中排队，事件分派器每次从队列中取出一个`socket`，根据`socket`的事件类型交给相应的事件处理器进行处理。

# 客户端与redis交互过程

![客户端与redis交互过程](img/redis/5AE201D71C159926092D25629BCA84E0.jpg)

>前提：通信是通过`socket`来完成

1. `redis`服务端进程初始化的时候，会将`server socket`的`AE_READABLE`事件与连接应答处理器关联。
   客户端`socket01`向`redis`进程的`server socket`请求建立连接，此时`server socket`会产生一个`AE_READABLE`事件，`IO`多路复用程序监听到`server socket`产生的事件后，将该`socket`压入队列中。文件事件分派器从队列中获取`socket`，交给连接应答处理器。连接应答处理器会创建一个能与客户端通信的`socket01`，并将该`socket01`的`AE_READABLE`事件与命令请求处理器关联。
2. 客户端发送了一个`set key value`请求
   `redis`中的`socket01`会产生`AE_READABLE`事件，`IO`多路复用程序将`socket01`压入队列，此时事件分派器从队列中获取到`socket01`产生的`AE_READABLE`事件，由于前面`socket01`的`AE_READABLE`事件已经与命令请求处理器关联，因此事件分派器将事件交给命令请求处理器来处理。命令请求处理器读取`socket01`的`key value`并在自己内存中完成`key value`的设置。操作完成后，它会将`socket01`的`AE_WRITABLE`事件与命令回复处理器关联。
3. 客户端准备好接收返回结果
   `redis`中的`socket01`会产生一个`AE_WRITABLE`事件，同样压入队列中，事件分派器找到相关联的命令回复处理器，由命令回复处理器对`socket01`输入本次操作的一个结果，比如`ok`，之后解除`socket01`的`AE_WRITABLE`事件与命令回复处理器的关联。

## `redis`单线程模型为啥效率这么高？

1. 纯内存操作。
2. 核心是基于非阻塞的`IO`多路复用机制。
3. C语言实现，一般来说，C语言实现的程序“距离”操作系统更近，执行速度相对会更快。
4. 单线程反而避免了多线程的频繁上下文切换问题，预防了多线程可能产生的竞争问题。

# 过期策略

定期删除+惰性删除

定期删除，指的是`redis`默认每隔`100ms`随机抽取一些设置了过期时间的`key`，检查其是否过期，如果过期就删除。

问题是定期删除可能会导致很多过期`key`到了时间并没有被删除掉，所以还要惰性删除。获取`key`时，如果此时`key`已经过期，就删除，不会返回任何东西。

但是还有问题，如果定期删除漏掉了很多过期`key`，也没及时去查，也没走惰性删除，此时大量过期`key`堆积在内存里，会导致`redis`内存块耗尽。如何解决？

答案是：内存淘汰机制。

## 内存淘汰机制
`redis`内存淘汰机制如下:

* `noeviction`: 当内存不足以容纳新写入数据时，新写入操作会报错
* `allkeys-lru`：当内存不足以容纳新写入数据时，在键空间中，移除**最近最少使用**的`key`（这个是最常用的）
* `allkeys-random`：当内存不足以容纳新写入数据时，在键空间中，**随机移除**某个`key`
* `volatile-lru`：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除**最近最少使用**的`key`
* `volatile-random`：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，**随机移除**某个`key`
* `volatile-ttl`：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有**更早过期时间的`key`优先移除**

# 持久化

两种方式

* `RDB`
  对`redis`中的数据执行周期性的持久化
* `AOF`
  对每条写入命令作为日志，以`append-only`的模式写入一个日志文件中，在`redis`重启的时候，可以通过回放`AOF`日志中的写入指令来重新构建整个数据集

使用`mac`电脑的同学都知道mac用来做硬盘数据备份的”时间机器“`app`，`rdb`就好比我们经常用云空间或移动硬盘开启时间机器`app`进行硬盘数据定期备份。而`aof`就好比我们执行每个操作时候同步进行的备份操作。当然`mac`电脑并没有提供类似`aof`这样的备份机制。

如果`redis`挂了，服务器上的内存和磁盘上的数据都丢了，可从云服务上拷贝之前的数据，放到指定目录，然后重新启动`redis`，`redis`就会自动根据持久化数据文件中的数据，去恢复内存中的数据，继续对外提供服务。

如果同时使用`RDB`和`AOF`两种持久化机制，那么在`redis`重启的时候，会使用`AOF`来重新构建数据，因为`AOF`中的数据更加完整。

## `RDB`优缺点

`RDB`会生成多个数据文件，每个数据文件都代表了某一个时刻`redis`数据，这种多个数据文件的方式，非常适合做冷备，可将这种完整的数据文件发送到一些远程的安全存储上去，以预定好的备份策略来定期备份`redis`数据。

`RDB`对`redis`对外提供的读写服务，影响非常小，可以让`redis`**保持高性能**，因为`redis`主进程只需要`fork`一个子进程，让子进程执行磁盘`IO`操作来进行`RDB`持久化即可。

相对于`AOF`持久化机制来说，直接基于`RDB`数据文件来重启和恢复`redis`进程，更加快速。

如果想要尽可能少的丢数据，那么`RDB`没有`AOF`好。`RDB`数据快照文件，都是每隔5分钟，或者更长时间生成一次，这个时候就得接受一旦`redis`进程挂机，那么最近5分钟的数据会丢失。

`RDB`每次在`fork`子进程来执行RDB快照数据文件生成的时候，如果数据文件特别大，可能会导致对客户端提供的服务暂停数毫秒，或者甚至数秒。

## `AOF`优缺点

`AOF`会每隔1秒，通过一个后台线程执行一次`fsync`操作，最多丢失1秒钟的数据。

`AOF`日志文件以`append-only`模式写入，没有任何磁盘寻址的开销，写入性能非常高，而且文件不容易破损，即使文件尾部破损，也很容易修复。

`AOF`日志文件即使过大的时候，出现后台重写操作，也不会影响客户端的读写。因为在`rewrite log`的时候，会对其中的指令进行压缩，创建出一份需要恢复数据的最小日志出来。在创建新日志文件的时候，老的日志文件还是照常写入。当新的`merge`后的日志文件`ready`的时候，再交换新老日志文件即可。

`AOF`日志文件的命令通过非常可读的方式进行记录，这个特性非常适合做**灾难性的误删除的紧急恢复**。比如不小心用`flushall`命令清空了所有数据，只要这个时候后台`rewrite`还没有发生，那么就可以立即拷贝`AOF`文件，将最后一条`flushall`命令删了，然后再将该`AOF`文件放回去，就可通过恢复机制，自动恢复所有数据。

对于同一份数据来说，`AOF`日志文件通常比`RDB`数据快照文件更大。

`AOF`开启后，支持的写`QPS`会比`RDB`支持的写`QPS`低，因为`AOF`一般会配置成每秒`fsync`一次日志文件，当然，每秒一次`fsync`，性能也还是很高的。（如果实时写入，那么`QPS`会大降，`redis`性能会大大降低）

类似`AOF`这种较为复杂的基于命令日志`/merge/`回放的方式，比基于`RDB`每次持久化一份完整的数据快照文件的方式，更加脆弱一些，容易有`bug`。不过`AOF`为了避免rewrite过程导致的`bug`，每次`rewrite`并不是基于旧的指令日志进行`merge`的，而是**基于当时内存中的数据进行指令的重新构建**，这样健壮性会好一点。

## 该选哪种来持久化？

1. 不要仅仅使用`RDB`，因为这样会导致你丢失很多数据
2. 也不要仅仅使用`AOF`，因为这样有两个问题
    * 通过`AOF`做冷备，没有`RDB`做冷备来的恢复速度更快
    * `RDB`每次简单粗暴生成数据快照，更加健壮，可避免`AOF`这种复杂的备份和恢复机制的`bug`
3. `redis`支持同时开启开启两种持久化方式，可综合使用`AOF`和`RDB`，用`AOF`来保证数据不丢失，作为数据恢复的第一选择; 用`RDB`来做不同程度的冷备，在`AOF`文件都丢失或损坏不可用的时候，还可使用`RDB`来进行快速的数据恢复。

# 架构

主从(`master-slave`)架构

一主多从，主负责写，并且将数据复制到其它的`slave`节点，从节点负责读。所有的读请求全部走从节点。这样可以轻松实现水平扩容，支撑**读高并发**

![master-slave架构](img/redis/551823AEC2EF71D93A52C09647CA57BB.jpg)

## `redis replication`

* 采用异步方式复制数据到`slave`节点，从`redis2.8`开始，`slave`会周期性地确认自己每次复制的数据量
* 一个`master`可配置多个`slave`
* `slave`也可连接其他的`slave`
* `slave`复制时，不会`block master`的正常工作
* `slave`在做复制时，也不会`block`对自己的查询操作，它会用旧的数据集来提供服务；但是复制完成的时候，需要删除旧数据集，加载新数据集，此时就会暂停对外服务了
* `slave`主要用来进行横向扩容，读写分离，扩容`slave`提高读的吞吐量

采用主从架构，必须开启`master`持久化。因为如果关掉`master`持久化，在`master`挂机重启时，数据是空的，然后可能一经过复制，`slave`的数据也丢了。

另外，需要对`master`做各种备份方案。万一本地所有文件丢失，从备份中挑选一份`rdb`去恢复`master`，这样才能确保启动时有数据，即使采用了高可用机制，`slave`可以自动接管`master`，也可能还没检测到`master failure`，`master`就自动重启了，还是可能导致所有的`slave`数据被清空。

## 主从复制核心原理

启动一个`slave`时，它会发送一个`PSYNC`命令给`master`。

如果这是`slave`第一次连接到`master`，就会触发一次`full` `resynchronization`（全量复制）。
此时`master`会启动一个后台线程，开始生成一份`RDB`快照文件，同时还会将从客户端`client`新收到的所有写命令缓存在内存中。
`RDB`文件生成完毕后，`master`会将这个`RDB`发送给`slave`，`slave`会先写入本地磁盘，然后再从本地磁盘加载到内存中，
接着`master`会将内存中缓存的写命令发送到`slave`，`slave`也会同步这些数据。
`slave`如果跟`master`有网络故障，断开连接，会自动重连，连接之后`master`仅会复制给`slave`部分缺少的数据

![主从复制核心原理](img/redis/E3C461B9F65E1C64C14B7784108F55D3.jpg)

几个小问题需要厘清

### 主从复制的断点续传

从`redis2.8`开始，就支持主从复制的断点续传，如果主从复制过程中，网络断了，可以接着上次复制的地方，继续复制。

`master`会在内存中维护一个`backlog`，`master`和`slave`都会保存一个`replica offset`还有一个`master run id`，`offset`就是保存在`backlog`中的。如果`master`和`slave`网络断了，`slave`会让`master`从上次`replica offset`开始继续复制，如果没有找到对应的`offset`，那么就会执行一次全量复制

>如果根据`host+ip`定位`master`是不靠谱的，如果`master`重启或数据出现了变化，`slave`应该根据不同的`run id`区分

### 无磁盘化复制

`master`在内存中直接创建`RDB`，然后发送给`slave`，不会在自己本地落盘。只需要在配置文件中开启
``` log
repl-diskless-sync yes
```

### 过期`key`处理
`slave`不会过期`key`，只会等待`master`过期`key`。如果`master`过期了一个`key`，或通过`LRU`淘汰了一个`key`，那么会模拟一条`del`命令发送给`slave`

## 主从复制完整流程

![主从复制完整流程](img/redis/E4236BBC31F337EED9790BB06F7E1FC3.jpg)

`slave`启动时，会在自己本地保存`master`的信息，包括`master`的`host`和`ip`，但是复制流程没开始。

`slave`内部有个定时任务，每秒检查是否有新的`master`要连接和复制，如果发现，就跟`master`建立`socket`网络连接。然后`slave`发送`ping`命令给`master`。如果`master`设置了`requirepass`，那么`slave`必须发送`masterauth`的口令过去进行认证。`master`第一次执行全量复制，将所有数据发给`slave`。而在后续，`master`持续将写命令，异步复制给`slave`

### 全量复制

* `master`执行`bgsave` ，在本地生成一份`rdb`快照文件。
* `master`将`rdb`快照文件发送给`slave`，如果`rdb`复制时间超过60秒（`repl-timeout`），那么`slave`就会认为复制失败，可以适当调大这个参数
* `master`在生成`rdb`时，会将所有新的写命令缓存在内存中，在`slave`保存了`rdb`之后，再将新的写命令复制给`slave`
* 如果在复制期间，内存缓冲区持续消耗超过64MB，或者一次性超过256MB，那么停止复制，复制失败
  `client-output-buffer-limit slave 256MB 64MB 60`

* `slave`接收到`rdb`之后，清空自己的旧数据，然后重新加载`rdb`到自己的内存中，同时基于旧的数据版本对外提供服务
* 如果`slave`开启了`AOF`，那么会立即执行`BGREWRITEAOF`，重写`AOF`

### 增量复制

* 全量复制过程中，如果网络断了，那么`slave`重新连接`master`时，会触发增量复制。
* `master`直接从自己的`backlog`中获取部分丢失的数据，发送给`slave`，默认`backlog`就是1MB
* `master`根据`slave`发送的`psync`中的`offset`来从`backlog`中获取数据

### 心跳（`heartbeat`）

* 主从节点互相发送`heartbeat`信息
* `master`默认每隔10秒发送一次`heartbeat`，`slave`每隔1秒发送一个`heartbeat`

### 异步复制
`master`每次接收到写命令之后，先在内部写入数据，然后异步发送给`slave`

# 如何实现`Redis`高可用？

`redis`的高可用架构，叫做`failover`故障转移，也可叫做主备切换

`master`在故障时，自动检测，并将某个`slave`自动切换为`master`的过程，就叫做主备切换

`redis3.x`版本开始引入自身的`cluster`集群机制，已经不需要使用哨兵模式来实现自身的高可用

`redis cluster`，主要是针对**海量数据+高并发+高可用**的场景。`redis cluster`支撑N个`redis`的`master`，每个`master`都可以挂载多个`slave`。整个`redis`就可横向扩容。如果要支撑更大数据量的缓存，那就横向扩容更多的`master`节点，每个`master`就能存放更多的数据。

## `redis cluster`介绍

* 自动将数据进行分片，每个`master`上放一部分数据
* 提供内置的高可用支持，部分`master`不可用时，还是可以继续工作的

在`redis cluster`架构下，每个`redis`要开放两个端口，比如一个是`6379`，另外一个就是加1w的端口号，比如`16379`。（其实都可以自行在配置文件中指定不同的端口号，只要端口号没被系统占用就行）

`16379`端口号是用来进行节点间通信的，也就是`cluster bus`，`cluster bus`通信是用来进行故障检测、配置更新、故障转移授权。它使用了另外一种二进制的协议，`gossip`协议，用于节点间进行高效的数据交换，占用更少的网络带宽和处理时间。

### 内部通信机制

集群元数据的维护有两种方式：集中式、`Gossip`协议。

`redis cluster`节点间采用`gossip`协议进行通信

>`gossip`协议:
>所有节点都持有一份元数据，不同的节点如果出现了元数据的变更，就不断将元数据发送给其它的节点，让其它节点也进行元数据的变更。

![gossip协议](img/redis/6FDCB17AF939679EF409B494C4073847.jpg)

`gossip`优点

元数据的更新比较分散，不是集中在一个地方，更新请求会陆陆续续打到所有节点上去更新，降低了压力

`gossip`缺点

元数据的更新有延时，可能导致集群中的一些操作会有一些滞后

每个节点都有一个专门用于节点间通信的端口，比如之前说的`6379`，那么用于节点间通信的就是`16379`端口。每个节点每隔一段时间都会往另外几个节点发送`ping`消息，同时其它几个节点接收到`ping`之后返回`pong`

交换的信息包括故障信息，节点的增加和删除，`hash slot`信息等等。

`gossip`协议包含多种消息，诸如`ping`,`pong`,`meet`,`fail`等等。

* `meet`: 某个节点发送`meet`给新加入的节点，让新节点加入集群中，然后新节点就会开始与其它节点进行通信
* 内部发送了一个`gossip meet`消息给新加入的节点，通知那个节点去加入我们的集群
* `ping`: 每个节点都会频繁给其它节点发送`ping`，其中包含自己的状态还有自己维护的集群元数据，互相通过`ping`交换元数据
* `pong`: 返回`ping`和`meet`，包含自己的状态和其它信息，也用于信息广播和更新
* `fail`: 某个节点判断另一个节点`fail`之后，就发送`fail`给其它节点，通知其它节点说，某个节点宕机了

### `hash slot`算法

`redis cluster`有固定的`16384`（2的14次方）个`hash slot`，对每个`key`计算`CRC16`值，然后对`16384`取模，可以获取`key`对应的`hash slot`

`redis cluster`中每个`master`都会持有部分`slot`，比如有3个`master`，那么可能每个`master`持有5000多个`hash slot`。`hash slot`让`node`的增加和移除很简单，增加一个`master`，就将其他`master`的`hash slot`移动部分过去，减少一个`master`，就将它的`hash slot`移动到其他`master`上去。移动`hash slot`的成本是非常低的。客户端的`api`，可以对指定的数据，让他们走同一个`hash slot`，通过`hash tag`来实现。

任何一台机器挂机，另外两个节点不影响。因为`key`找的是`hash slot`，不是机器。

![hash slot](img/redis/41022A3C81D77421B84BFB25167EC3C8.jpg)

## `redis cluster`实现高可用

### 判断节点挂机

如果一个节点认为另外一个节点挂机，那么就是`pfail`: **主观挂机**。如果多个节点都认为另外一个节点挂机了，那么就是`fail`: **客观挂机**

在`cluster-node-timeout`内，某个节点一直没有返回`pong`，那么就被认为`pfail`。

如果一个节点认为某个节点`pfail`了，那么会在`gossip ping`消息中，`ping`给其他节点，如果超过半数的节点都认为`pfail`了，那么就会变成`fail`。

### 从节点过滤
对挂机的`master`，从其所有的`slave`中，选择一个切换成`master`。

检查每个`slave`与`master`断开连接的时间，如果超过了`cluster-node-timeout` ,那么就没有资格切换成`master`。

### 从节点选举

每个`slave`都根据自己对`master`复制数据的`offset`，来设置一个选举时间，`offset`越大（复制数据越多）的`slave`，选举时间越靠前，优先进行选举

所有的`master`开始`slave`选举投票，给要进行选举的`slave`进行投票，如果大部分`master`（`N/2 + 1`）都投票给了某个`slave` ，那么选举通过，这个`slave`就可变成`master`。

然后，`slave`开始执行主备切换，切换成`master`。

# `Redis`缓存

目前大多数公司对`redis`的应用在于构建分布式缓存方面，因此对于`redis`缓存使用，面试官会问下列一些问题

## 缓存雪崩

指缓存在某一时刻数据全部失效，所有对数据查询请求全部跑到`DB`，`DB`瞬时因为请求压力太大，产生挂机或耗时处理。严重的，甚至会让`DB`也挂掉。导致整体系统处于不可用状态。

缓存数据全部失效有两种可能:

1. 缓存服务器挂掉了
2. 缓存数据中部分数据的缓存有效时间在某一个时刻全部集体失效，虽然缓存服务器还是可用状态，但是其中大部分缓存数据已不存在，对数据进行查询的请求只能走`DB`

解决方案:

* 保证`redis`高可用，使用`redis cluster`避免全盘崩溃
* 本地`ehcache`缓存+`hystrix`限流&降级，避免`DB`挂机
* `redis`数据做持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据
* 在原有的缓存有效时间上增加一个随机值，比如1-5分钟随机，或使用`java`的`random`方法。保证每个缓存数据有效时间的重复率降低，就不太会引发集体失效的事情

## 缓存穿透

黑客发起的恶意攻击或是人为向缓存数据发起了一个缓存和`db`都不存在的数据查询请求。这个不存在的数据每次请求发现缓存中没有就会去`DB`查询，这样就失去了缓存存在意义。请求流量大时，可导致`DB`挂机不可用。

解决方案:

* 将不存在的数据缓存起来，并设置一个过期时间，下次有相同的不存在数据查询请求过来，在缓存失效之前，都可直接从缓存中取出这个数据
* 在缓存之前，设置布隆过滤器，实际上是一个`bitMap`结构。当一个元素被加入时，将这个元素映射成一个位数组中的K个点，把它们置为1。检索时，只要看看这些点是不是都是1就（大约）知道集合中有没有它了。如果这些点有任何一个0，则被检元素一定不存在；如果都是1，则被检元素很可能存在。不存在就直接返回（针对黑客攻击都采用这一方案）

## 缓存击穿

某个`key`非常热点，访问非常频繁，处于集中式高并发访问的情况，当这个`key`在失效瞬间，大量的请求就击穿了缓存，直接请求数据库，就像是在一道屏障上凿开了一个洞。又被称之为缓存并发

解决方案:
使用分布式锁，保证对于每个`key`同时只有一个线程去查询，其他线程没有获得分布式锁，因此只能等待。这种方式将高并发的压力转移到了分布式锁，因此对分布式锁的考验很大。下面会详细讲`redis`分布式锁实现。

至于某些资料说将数据设置为”永不过期“，个人认为是治标不治本的方法。容易引起缓存和`DB`数据不一致问题。

## 缓存和`DB`数据双写一致

`DB`数据随着时间可能会发生变化，而缓存数据不及时同步更新就会不一致。所以需要做到最终一致。

目前做法是

1. 先写`DB`，再删除缓存（`Cache Aside`模式）
2. 先删除缓存，再写`DB`

其实都有问题
如果1，在删除缓存前，写`DB`的线程挂了，没有删除掉缓存，则会出现数据不一致
如果2，删除了缓存，还没有来得及写`DB`，另一个线程就来读取，发现缓存为空，则去DB中读取数据写入缓存，此时缓存中就为脏数据。
本质是因为写和读是并发的，没法保证顺序,所以会出现数据不一致问题。

解决方案:

* 情况1，异步更新缓存(基于订阅`binlog`的同步机制)
  `binlog`增量订阅消费+消息队列+增量数据更新到缓存
  步骤
    1. 写`DB`
    2. 数据更新日志写入`binlog`中
    3. 读取`binlog`后，解析数据，利用消息队列推送到各节点更新缓存数据
    4. 尝试删除缓存操作
    5. 删除失败，将需要删除的缓存`Key`发送到消息队列中
    6. 从队列中拿到要删除的缓存`key`，再次尝试删除缓存，如果再次删除失败，可重发消息多次尝试;

    >总的来说就是提供一个"重试保障机制"，如果删除缓存失败，可将删除失败的key发送到消息队列，再进行重试删除操作

* 情况2，延时双删策略+缓存超时设置
  写`DB`前后都进行`redis.del`(`key`操作，并设定合理的有效时间)。
  步骤
    1. 先删缓存
    2. 再写`DB`
    3. 休眠500毫秒
       那么，这个500毫秒怎么确定的，具体休眠多久呢？

       评估自己项目的读数据业务逻辑的耗时。目的是确保读请求结束，写请求可以删除读请求造成的缓存脏数据

       还要考虑`redis`和`DB`主从同步的耗时。最后写数据的休眠时间应该在读数据业务逻辑的耗时基础上，加几百ms

    4. 再删缓存
       给缓存设置有效时间，用来保证最终一致性。所有的写操作以`DB`为准，只要缓存有效时间到了，则后面的读请求自然会从`DB`中读取新值然后回填缓存

    >结合延时双删策略+缓存超时设置，最差情况是在有效时间内数据存在不一致，且增加了写请求的耗时

# `Redis`分布式锁

## 单`Redis`节点

使用`setnx`命令创建一`key`，这样就算加锁。

``` log
SET resource_name my_random_value NX PX 30000
```

* `my_random_value`是由客户端生成的一个随机字符串，它要保证在足够长的一段时间内在所有客户端的所有获取锁的请求中都是唯一的
* `NX`表示只有`key`不存在的时候才会设置成功。（如果此时存在这个`key`，那么设置失败，返回`nil`）
* `PX 30000`意思是30s后锁自动释放。别人创建的时候如果发现已经有了就不能加锁了

释放锁就是删除`key`，一般可以用`lua`脚本删除，判断`value`一样才删除

``` nginx
-- 删除锁的时候，找到 key 对应的 value，跟自己传过去的 value 做比较，如果是一样的才删除。
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

这段`Lua`脚本在执行的时候要把前面的`my_random_value`作为`ARGV[1]`的值传进去，把`resource_name`作为`KEYS[1]`的值传进去。

### 问题

1. 锁必须要设置一个过期时间。否则当一个客户端获取锁成功之后，假如它崩溃了，或者网络不可用导致它再也无法和`Redis`节点通信了，那么它就会一直持有这个锁，而其它客户端永远无法获得锁。这个过期时间被称为锁的有效时间`lock validity time`。获得锁的客户端必须在这个时间之内完成对共享资源的访问
2. 获取锁操作不应该写成
    ``` log
    SETNX resource_name my_random_value
    EXPIRE resource_name 30
    ```

   虽然这两个命令和前面描述中的`SET`命令执行效果相同，但却不是原子的。如果客户端在执行完`SETNX`后崩溃了，那么就没有机会执行`EXPIRE`了，导致它一直持有这个锁
3. 设置一个随机字符串`my_random_value`保证了一个客户端释放的锁必须是自己持有的那个锁。假设获取锁时`SET`的不是随机字符串，而是固定值，那么可能会发生下面的执行序列
    * 客户端1获取锁成功。
    * 客户端1在某个操作上阻塞了很长时间。
    * 过期时间一到，锁自动释放。
    * 客户端2获取到了对应同一个资源的锁。
    * 客户端1从阻塞中恢复过来，释放掉了客户端2持有的锁。

   之后，客户端2在访问共享资源的时候，就没有锁为它提供保护了
4. 释放锁的操作必须使用Lua脚本来实现。释放锁其实包含三步操作：`GET`、判断和`DEL`，用`Lua`脚本来实现能保证这三步的原子性。否则，如果把这三步操作放到客户端逻辑中去执行的话，就有可能发生与前面问题3类似的执行序列
    * 客户端1获取锁成功。
    * 客户端1访问共享资源。
    * 客户端1为了释放锁，先执行`GET`操作获取随机字符串的值。
    * 客户端1判断随机字符串的值，与预期的值相等。
    * 客户端1由于某个原因阻塞住了很长时间。
    * 过期时间到了，锁自动释放了。
    * 客户端2获取到了对应同一个资源的锁。
    * 客户端1从阻塞中恢复过来，执行`DEL`操作，释放掉了客户端2持有的锁。
5. 假如`Redis`节点挂机了，那么所有客户端就都无法获得锁，服务变得不可用。为了提高可用性，给这个`Redis`节点挂一个`Slave`，当`Master`不可用时，系统自动切到`Slave`上（`failover`）。但由于`Redis`的主从复制（`replication`）是异步的，这可能导致`failover`过程中丧失锁了的安全性。考虑下面的执行序列:
    * 客户端1从`Master`获取了锁
    * `Master`挂机了，存储锁的`key`还没有来得及同步到`Slave`上。
    * `Slave`升级为`Master`
    * 客户端2从新`Master`获取到了对应同一个资源的锁

   于是，客户端1和客户端2同时持有了同一个资源的锁。锁的安全性被打破

基于单`Redis`节点的分布式锁无法解决这个问题5。而正是这个问题催生了`RedLock`的出现

## Redission

### 原理图

![Redission原理图](img/redis/E8FE04BB912A5E2A30210D36DE4785E1.jpg)

### 加锁

`Lua`脚本

``` nginx
if (redis.call('exists', KEYS[1]) == 0) then 
        redis.call('hset', KEYS[1], ARGV[2], 1);
         redis.call('pexpire', KEYS[1], ARGV[1]); 
         return nil;
          end;
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
        redis.call('hincrby', KEYS[1], ARGV[2], 1);
        redis.call('pexpire', KEYS[1], ARGV[1]); 
        return nil;
        end;
return redis.call('pttl', KEYS[1]);
```

为什么要用`lua`?
通过封装在`lua`脚本中发送给`redis`，保证执行的原子性

解释

`KEYS[1]`:表示你加锁的那个`key`，比如说
``` nginx
RLock lock = redisson.getLock(“myLock”);
```
这里你自己设置了加锁的那个锁`key`就是`myLock`。

`ARGV[1]`:表示锁的有效期，默认30s
`ARGV[2]`:表示表示加锁的客户端ID,类似于下面这样
`8743c9c0-0795-4907-87fd-6c719a6b4586:1`

用`exists myLock`命令判断，如果要加锁的那个锁`key`不存在就进行加锁,接着执行`pexpire myLock 30000`命令，设置`myLock`这个锁`key`的生存时间是30秒(默认)

### 锁互斥

如果客户端2尝试加锁，执行了同样的一段`lua`脚本，第一个`if`判断会执行`exists myLock`，发现`myLock`这个锁`key`已经存在了。

接着第二个`if`判断，判断一下，`myLock`锁`key`的`hash`数据结构中，是否包含客户端2的`ID`，但明显不是，包含的是客户端1的`ID`。

所以，客户端2会获取到`pttl myLock`返回的一个数字，这个数字代表了`myLock`这个锁`key`的剩余生存时间。比如还剩15000毫秒的生存时间。

此时客户端2会进入一个`while`循环，不停的尝试加锁。

### `watch dog`自动延期机制

客户端1加锁的锁`key`默认生存时间才30秒，如果超过了30秒，客户端1还想一直持有这把锁，怎么办？

只要客户端1一旦加锁成功，就会启动一个`watch dog`看门狗，这是一个后台线程，会每隔10秒检查一下，如果客户端1还持有锁`key`，那就会不断的延长锁`key`的生存时间。续命操作

### 可重入加锁机制
如果客户端1已经持有这把锁，可重入的加锁会如何

``` nginx
#重入加锁
RLock lock = redisson.getLock("myLock")
lock.lock();
//业务代码
lock.lock();
//业务代码
lock.unlock();
lock.unlock();
```

分析上面`lua`代码
第一个`if`判断不成立，`exists myLock`会显示锁`key`已经存在了

第二个`if`会成立，因为`myLock`的`hash`数据结构中包含的客户端1的`ID`

此时就会执行可重入加锁的逻辑，用`incrby`这个命令，对客户端1的加锁次数，累加1

### 解锁
``` nginx
if (redis.call('exists', KEYS[1]) == 0) then
       redis.call('publish', KEYS[2], ARGV[1]);
        return 1; 
        end;
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then 
     return nil;
     end;
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); 
if (counter > 0) then
     redis.call('pexpire', KEYS[1], ARGV[2]); 
     return 0; 
else redis.call('del', KEYS[1]); 
     redis.call('publish', KEYS[2], ARGV[1]); 
     return 1;
     end;
return nil;
```

执行`lock.unlock()`，释放分布式锁，每次对`myLock`数据结构中的那个加锁次数减1。如果发现加锁次数是0了，说明客户端1已不再持有锁，此时就会用：`del myLock`命令，从`redis`里删除这个`key`。
然后另外的客户端2就可尝试加锁。

### 问题

见[单Redis节点](3135569683.html#单Redis节点)问题5，`redis`主从异步复制导致`redis`分布式锁的最大缺陷:
在`redis master`实例挂掉时，可能导致多个客户端同时完成加锁。所以还是要实现`RedLock`

## 集群`Redis`节点（`RedLock`）

运行`RedLock`算法的客户端依次执行下面各个步骤，来完成获取锁的操作

1. 获取当前时间（毫秒数）
2. 按顺序依次向N个`Redis`节点执行获取锁的操作。这个获取操作跟单`Redis`节点获取锁的过程相同，包含随机字符串`my_random_value`，也包含过期时间(比如`PX 30000`，即锁的有效时间)。为保证在某个`Redis`节点不可用的时候算法能够继续运行，这个获取锁的操作还有一个超时时间(`time out`)，它要远小于锁的有效时间（几十毫秒量级）。客户端在向某个`Redis`节点获取锁失败以后，应该立即尝试下一个`Redis`节点。这里的失败，应该包含任何类型的失败，比如该`Redis`节点不可用，或者该`Redis`节点上的锁已经被其它客户端持有
3. 计算整个获取锁的过程总共消耗了多长时间，计算方法是用当前时间减去第1步记录的时间。如果客户端从大多数`Redis`节点（`>= N/2+1`）成功获取到了锁，并且获取锁总共消耗的时间没有超过锁的有效时间(`lock validity time`)，那么这时客户端才认为最终获取锁成功；否则，认为最终获取锁失败。
4. 如果最终获取锁成功了，那么这个锁的有效时间应该重新计算，它等于最初的锁的有效时间减去第3步计算出来的获取锁消耗的时间。
5. 如果最终获取锁失败了（可能由于获取到锁的Redis节点个数少于`N/2+1`，或整个获取锁的过程消耗的时间超过了锁的最初有效时间），那么客户端应立即向所有`Redis`节点发起释放锁的操作（即前面介绍的`Lua`脚本）

当然，上面描述的只是获取锁的过程，而释放锁的过程比较简单: 客户端向所有`Redis`节点发起释放锁的操作，不管这些节点当时在获取锁的时候成功与否。

### 问题
1. 如果有节点发生崩溃重启，还是会对锁的安全性有影响的。具体的影响程度跟`Redis`对数据的持久化程度有关。
   假设一共有5个`Redis`节点：A, B, C, D, E。设想发生了如下的事件序列:
    * 客户端1成功锁住了A, B, C，获取锁成功（但D和E没有锁住）。
    * 节点C崩溃重启了，但客户端1在C上加的锁没有持久化下来，丢失了。
    * 节点C重启后，客户端2锁住了C, D, E，获取锁成功。

   这样，客户端1和客户端2同时获得了锁（针对同一资源）,需要延迟重启解决。也就是说，一个节点崩溃后，先不立即重启，等待一段时间再重启，这段时间应该大于锁的有效时间(`lock validity time`)。这样这个节点在重启前所参与的锁都会过期，它在重启后就不会对现有的锁造成影响。

2. 释放锁的时候，客户端应该向所有`Redis`节点发起释放锁的操作。即使当时向某个节点获取锁没有成功，在释放锁的时候也不应该漏掉这个节点。why？设想这样一种情况，客户端发给某个`Redis`节点的获取锁请求成功到达了该`Redis`节点，这个节点也成功执行了`SET`操作，但是它返回给客户端的响应包丢失了。在客户端看来，获取锁的请求由于超时而失败，但在`Redis`这边看来，加锁已经成功了。因此，释放锁的时候，客户端也应该对当时获取锁失败的那些`Redis`节点同样发起请求

# 参考资料

1. [中华石杉--互联网Java进阶面试训练营](https://gitee.com/shishan100/Java-Interview-Advanced)

2. [基于Redis的分布式锁到底安全吗？（上）](https://mp.weixin.qq.com/s/JTsJCDuasgIJ0j95K8Ay8w)

3. [基于Redis的分布式锁到底安全吗？（下）](https://mp.weixin.qq.com/s/4CUe7OpM6y1kQRK8TOC_qQ)

4. [2020最全Redis分布式锁全集](https://www.bilibili.com/video/BV1d741127ZA?p=5)

# [推荐书单](/Redis)
