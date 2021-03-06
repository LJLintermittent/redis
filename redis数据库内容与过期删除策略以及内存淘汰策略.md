redis给缓存数据设置过期时间，因为内存是有限的，且根据缓存的业务需求，有些内容需要定时更换，比如用户的登录验证码，一般在几分钟内是可用的，通过过期时间来实现验证码的过期失效功能

redis内部是通过一个过期字典来保存数据的过期时间，过期字典的键指向redis数据库中的某个key，过期时间的值是一个long类型的整数，这个整数保存了key所指向的数据库键的过期时间，毫秒精度的unix时间戳

那么对过期键的删除其实就是先通过给定的key在过期字典里找，看存不存在，存在话将过期字典中的键（指向键空间的某个键）与它的所对应的值（过期时间戳）之间的关系进行一个解绑

过期键的判定通过过期字典，首先检查给定键是否存在于过期字典，如果存在，那么取得键的过期时间，检查当前unix时间戳是否大于键的过期时间，如果是的话，那么键已经过期，否则的话，键未过期。

过期数据的删除策略：

1.定时删除：在设置键的过期时间的同时设置一个定时器，就相当于一个过期回调函数，只要过期了，立马进行过期

2.惰性删除：只会在取出key的时候才对数据进行过期检查，这样对cpu最好，但是可能会存在太多过期的key没有删除

3.定期删除：每隔一段时间抽取一批key执行删除过期key的操作，且redis底层会通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响

可以看出仅仅通过惰性删除和定期删除是没有办法完全解决内存中存留部分过期key的情况，所以在极端情况下，还会有oom的风险

定时删除策略是对内存最友好的，通过使用定时器，定时删除策略会保证过期的键一定不会存在在数据库中，但是缺点也很明显，那就是对CPU不友好，思考这样一种情景，如果我的redis服务器的内存资源很充足，但是CPU比较紧张，那么如果这时候有大并发流量进来后，我希望CPU能更多的先去处理客户端发来的读写请求，而不是将有限的CPU资源用在删除过期key上。除此以外，很重要的一点是创建定时器需要用到redis的时间事件，而时间事件的实现方式是无序链表，查找某一个时间事件的时间复杂复杂度是0N，所以说明并不能高效处理大量的时间事件。

惰性删除是对CPU最友好的策略，程序只会在取出键时进行判断，如果key已经过期了，那么执行删除操作，但是缺点是对内存不友好，并且可能会造成不被访问的过期key永远不能被删除，思考场景：秒杀服务一般会将秒杀商品的信息提前缓存在redis中，并在秒杀活动过期以后的半个小时或者一个小时删除秒杀商品信息key，前端一般都不会再展示已经过期的秒杀商品了，那么这时候如果使用惰性删除策略，秒杀key永远不会再被用户访问，那么即使设置了过期时间，依然会永久存在redis中，这是一个非常严重的事情，也就是内存泄露

定期删除是为了平衡CPU与内存之间的开销。定期删除每隔一段时间执行一次删除过期的key的操作，并通过限制删除操作执行的时长和频率来控制对CPU的影响

目前的redis服务器实际会采用惰性删除+定期删除策略

### 惰性删除+定期删除原理：

首先对于惰性删除的源码，在expireIfNeeded方法

~~~c
int expireIfNeeded(redisDb *db, robj *key) {
    if (!keyIsExpired(db,key)) return 0;
    if (server.masterhost != NULL) return 1;
    if (checkClientPauseTimeoutAndReturnIfPaused()) return 1;
    /* Delete the key */
    if (server.lazyfree_lazy_expire) {
        dbAsyncDelete(db,key);
    } else {
        dbSyncDelete(db,key);
    }
    server.stat_expiredkeys++;
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    signalModifiedKey(NULL,db,key);
    return 1;
}
~~~

所有读写数据库的命令，在真正执行实际命令前，都会走一遍expireIfNeeded函数的流程，就相当于是一个过滤器一样，这时候如果检查键过期了需要对键进行删除操作，另外如果键不存会直接返回null

定期删除策略源码：activeExpireCycle

每当redis的服务器周期性操作serverCron函数（整个服务的调度函数）的时候，activeExpireCycle就会被调用，然后它会在规定的时间里，分多次遍历服务器中的各个数据库，从数据库中的过期字典里随机找出部分过期键，并执行删除

activeExpireCycle的工作模式可以总结如下：

~~~wiki
1.函数每次运行时，都从一定数量的数据库中取出一定数量的随机键进行检查，并删除其中的过期键
2.全局变量current_db会记录当前activeExpireCycle函数检查的进度，并在下一次调用的时候就需要从这个点出发继续寻找
3.随着serverCron函数的定时执行，activeExpireCycle函数不断被调用，最终会把所有库都扫描一遍，然后current_db重置为0，继续下一轮
~~~



aof和rdb和主从复制对于过期键的处理：

在执行save命令或者bgsave命令（save是主线程去做同步磁盘的工作，当然同步的是rdb文件。bgsave是fork出一个子进程去做刷盘工作，父进程依然处理客户端发来的读写请求），程序会对数据库中的键进行检查，保证已过期的键不会加入到新的rdb文件中

同时在读取rdb文件的时候，如果里面的键有的已经过期了，是不会被载入到数据库中的，所以过期键对载入rdb文件的服务器没有任何影响，但是这是对于以主服务器模式运行的redis，如果对于以从服务器模式运行的redis，它会载入所有的键，不论是否过期，从服务器对于过期键的删除只有一种途径，就是主服务器发来del命令

aof文件写入策略，当过期键被惰性删除或者被定期删除后，程序会向aof文件追加一条del命令，来显示删除这个过期键

执行流程就是在数据库中删除以后，向aof文件加一条del命令

同时aof文件重写的时候也会检查键是否过期，如果过期了就不会被保存在aof文件中

过期键删除对主从复制架构的影响：

当集群以主从架构运行的时候，从服务器的过期删除动作由主服务器控制，主服务器在删除一个过期键以后，会显示地向所有从服务器发送del命令，告知从服务器删除这个过期key，从服务器如果碰到请求过期key，那么不会删除，而是返回这个key，直到接收到了来自主服务器发来的del命令，才做一个删除操作。

### redis设计与实现第九章总结

redis服务器所有数据都保存在redisserver.db数组中，而数组数量可以调整，默认是16

客户端通过修改目标数据库指针，来指向redisserver.db数组中不同的数据库，达到切换库的目的

数据库主要由两个字典构成，一个是dict（键空间字典），一个是expires字典，dict用于保存所有键值对，expires用来保存键和对应的过期时间

因为数据库是由字典建立的，所以对数据库的操作都是建立在字典操作上的。

数据库的键总是一个字符串对象，值对象可以是字符串对象，哈希对象，列表对象等任何一种

redis使用定期删除和惰性删除两种方法来删除过期key。

redis内存淘汰策略：

1.volatile-lru（least recently used）：从已设置过期时间的数据集中挑选最近最少使用的数据淘汰

2.volatile-ttl：从已设置过期时间的数据集中挑选将要过期的数据淘汰

3.volatile-random：从已设置过期时间的数据集中任意选择数据进行淘汰

4.volatile-lfu：从已设置过期时间的数据集中挑选最不经常使用的数据淘汰

5.allkeys-lru：当内存不足以写入新数据时，在键空间中，移除最近最少使用的key。这个是最常用的

6.allkeys-lfu：当内存不足以写入新数据时，在键空间中，移除最不经常使用的key

7.allkeys-random：从数据集中任意选择数据淘汰

8.no-eviction：禁止淘汰，当内存中不足以写入新数据的时候，直接报错



