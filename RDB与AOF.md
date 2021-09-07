### 持久化机制

redis有两种持久化方式：一种是基于快照的，叫RDB，另一种是只追加文件，AOF

RDB源码分析：rdb.c/rdbSave文件

~~~c
int rdbSave(char *filename, rdbSaveInfo *rsi) {
    char tmpfile[256];// 临时文件缓冲区
    char cwd[MAXPATHLEN]; /* Current working dir path for error messages. */
    FILE *fp = NULL;// 文件句柄
    rio rdb;// rdb文件
    int error = 0;
    // 创建一个临时的rdb文件。以w的方式创建.
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());
    fp = fopen(tmpfile,"w");
    // 如果打开失败，就把错误存储在cwd中
    if (!fp) {
        char *cwdp = getcwd(cwd,MAXPATHLEN);
        serverLog(LL_WARNING,
            "Failed opening the RDB file %s (in server root dir %s) "
            "for saving: %s",
            filename,
            cwdp ? cwdp : "unknown",
            strerror(errno));
        return C_ERR;
    }
    // 初始化IO
    rioInitWithFile(&rdb,fp);
    // 开始保存 RDBFLAGS_NONE = 0
    startSaving(RDBFLAGS_NONE);
    // 自动同步rdb REDIS_AUTOSYNC_BYTES (1024*1024*32)=32MB
    if (server.rdb_save_incremental_fsync)
        // 如果打开了增量同步开关，每32M进行一次flush操作，在后面的流程看到这个选项的作用
        rioSetAutoSync(&rdb,REDIS_AUTOSYNC_BYTES);
    // 写入文件，如果失败就直接返回错误error
    if (rdbSaveRio(&rdb,&error,RDBFLAGS_NONE,rsi) == C_ERR) {
        errno = error;
        goto werr;
    }

    /* Make sure data will not remain on the OS's output buffers */
    // 确保数据不会保留在操作系统的输出缓冲区上
    if (fflush(fp)) goto werr;
    if (fsync(fileno(fp))) goto werr;
    if (fclose(fp)) { fp = NULL; goto werr; }
    fp = NULL;
    
    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. */
     // 使用RENAME来确保只有在生成的DB文件ok的情况下才会自动更改DB文件
    if (rename(tmpfile,filename) == -1) {
        char *cwdp = getcwd(cwd,MAXPATHLEN);
        serverLog(LL_WARNING,
            "Error moving temp DB file %s on the final "
            "destination %s (in server root dir %s): %s",
            tmpfile,
            filename,
            cwdp ? cwdp : "unknown",
            strerror(errno));
        unlink(tmpfile);// 删除临时文件
        stopSaving(0);// 停止保存
        return C_ERR;// 返回异常信息
    }
    // 输出RDB保存成功日志
    serverLog(LL_NOTICE,"DB saved on disk");
    // 设置保存状态
    server.dirty = 0;// 数据库修改次数
    server.lastsave = time(NULL);// 上一次执行时间
    server.lastbgsave_status = C_OK;
    stopSaving(1);
    return C_OK;

werr:
    // 输出失败错误日志
    serverLog(LL_WARNING,"Write error saving DB on disk: %s", strerror(errno));
    if (fp) fclose(fp);
    unlink(tmpfile);
    stopSaving(0);
    return C_ERR;
}
~~~

redis可以通过创建快照的方式来获得存储在内存里面的数据在某个时间节点上的副本，redis创建快照后，可以对快照进行备份，可以将快照复制到其他服务器从而创建此redis服务器的副本，redis主从，还可以将redis留在原地以便重启服务器的时候使用

快照持久化是redis默认采用的持久化方式，在redis.conf配置文件中默认有如下配置：

save 900 1 ：在900s后有至少一个key发生变化，那么就需要触发bgsave命令创建快照

save 300 10 ：在300s后有至少10个key发生变化，那么就需要触发bgsave命令创建快照

save 60 10000：在60s后至少有10000个key发生变化，那么就需要触发bgsave命令创建快照

从快照这种方式可以 看出来，是存在丢失数据的风险的

主流的方式是AOF，aof持久化的实时性更好，因此已经成为更加主流的方式，appendonly yes开启AOF

开启AOF后每执行一条会更改redis的命令就会将这个命令加入到aof缓存中，然后再根据appendfsync的配置来决定何时将其同步到磁盘中的aof文件中，三种redis的aof同步策略是：

```
appendfsync always    #每次有数据修改发生时都会写入AOF文件,这样会严重降低Redis的速度
appendfsync everysec  #每秒钟同步一次，显示地将多个写命令同步到硬盘
appendfsync no        #让操作系统决定何时进行同步
```

为了兼顾数据安全和写入性能，一般可以考虑appendfsync everysec参数，让redis每秒同步到磁盘一次aof，redis性能基本没有任何影响，而且这样即使出现系统崩溃，用户最多只会丢失一秒之内产生的数据

redis4.0开始支持RDB和AOF的混合持久化方式，默认是关闭的，可以通过配置项 `aof-use-rdb-preamble` 开启

RDB+AOF：可以在同一个实例中组合 AOF 和 RDB。请注意，在这种情况下，当redis重启时，aof文件将用于重建原始数据集，因为他保证是最完整的

RDB的优势：

1.rdb是redis数据的非常紧凑的单文件时间节点表示，rdb文件非常适合备份，可以在灾难恢复时恢复不同版本的数据

2.rdb最大程度的提高了redis的性能，因为redis父进程为了持久化需要做的唯一工作时派生一个将完成其余所有工作的子进程，父进程永远不会执行磁盘IO或类似动作，这是bgsave

3.与aof相比，rdb允许更快的启动大数据集

RDB的缺点：

1.如果需要在redis异常停止工作重启后最大限度的恢复数据，那么RDB并不好，虽然可以配置多个保存点，比如说x分钟至少有y个key发生了变化，但是通常这个时间会在几分钟以上，也就是说使用RDB需要做好损失数据的准备

2.RDB 经常需要 fork() 以便使用子进程在磁盘上持久化。如果数据集很大，Fork() 可能会很耗时，如果数据集很大且 CPU 性能不是很好，可能会导致 Redis 停止为客户端服务几毫秒甚至一秒钟。AOF 也需要 fork() 但你可以调整你想要重写日志的频率，而不会对持久性进行任何权衡

AOF的优势：

1.使用 AOF Redis 更持久：根本不做 fsync（交给操作系统），每秒 fsync，每次查询 fsync。使用 fsync 每秒写入性能的默认策略仍然很棒（fsync 是使用后台线程执行的，当没有 fsync 正在进行时，主线程将努力执行写入。）最多会丢失一秒的写数据

2.aof是仅附加日志，因此在断电时不会出现寻道或损坏的情况，即使日志由于某种原因（磁盘已满或者其他原因）以半写命令结束，redis-check-aof工具依然可以将它恢复

3.aof有易于理解和解析的格式包含所有操作的日志，可以轻松的导出AOF

AOF的缺点：

1.aof文件通常比相同数据集的等效RDB大

2.aof可能比rdb慢，具体取决于fsync策略，一般来说将fsync策略设置为每秒一次的时候性能是比较不错的，如果每次更新都fsync那么性能会下降很多

### 日志重写功能

随着更新操作的增多，aof会变得越来越大，比如我将一个值递增100次，最终在数据集中还是这一值，但是aof文件为了记录需要保存100个条目，为了重建状态可以直接使用一条命令达到最终的效果，所以redis加入了日志重写功能，能够在不中断对客户端的服务的连接的情况下在后台重建aof，bgrewriteaof命令可以重建aof，在mysql2.4版本已经能够自动触发日志重写

至于aof的刷盘策略，建议使用每秒一次的fsync。always策略足够安全但是非常慢，有组提交优化，如果有多个并行写入，redis会尝试执行单个fsync操作

aof断裂也是可以恢复数据的，比如正在写aof的过程中服务器宕机了，那么aof文件后面的部分命令可能会被截断，redis的最新版本无论如何都会加载aof，只需丢弃文件中最后一个格式不正确的命令就ok了

~~~wiki
RDB持久化：
rdb持久化可以手动执行，也可以根据服务器配置项save 900 1 这样子按条件定期执行，该功能会将某一个时间点上的数据库保存在磁盘里面，避免数据意外丢失
有两个命令可以直接用于生成rdb文件，一个是save，一个是bgsave。
save命令会阻塞redis服务器进程，因为主进程会去做生成RDB文件的工作，而bgsave会创建一个子进程，然后由子进程执行RDB文件的创建，服务器进程会继续处理命令请求。创建RDB的函数由rdb.c的RDBsave函数完成，save和bgsave命令会以不同的方式来调用这个函数
RDB文件在服务器重启以后自动载入，另外注意，由于aof文件的更新频率通常来说比rdb文件更高，所以如果开启了aof持久化，那么就会优先使用aof的文件来载入
载入函数由rdbload完成，
save命令会阻塞所有来自客户端的读写请求，主要当服务进程执行完rdb的创建后，其他请求才可以被处理
bgsave命令会派生个子进程，redis服务器仍然可以执行客户端的命令请求，但是在bgsave执行期间：
客户端发来的save请求会被服务器拒绝，服务器禁止save和bgsave同时运行是为了避免父子进程同时执行两个rdbsave调用，防止产生竞争
其次在bgsave执行的时候另一个bgsave也是被拒绝的，因为同时执行两个bgsave也会产生竞争
最后bgsave和bgrewriteaof也不能一块执行，这俩都是由子进程执行，所以两个命令在操作方面并没有冲突，禁止只是为了性能考虑-并发两个子进程同时执行大量的磁盘读写工作，是很吃性能的。

另外rdb在载入的时候，服务器整体会stw
~~~

~~~wiki
自动间隔性保存
因为bgsave不会阻塞redis服务，所以在配置文件中redis提供了配置bgsave的执行条件，比如save 900 1 实际上就会在满足条件的情况下调用bgsave，这个条件的意义是在900秒以内至少有一个key发生了改变
如果用户没有主动配置save，那么默认是
save 900 1 
save 300 10 
save 60 10000
接着服务器会根据save选项配置的条件设置服务器状态redisserver中的saveparms属性
saveparms是一个保存了save条件的一个数组，saveparms结构体里面不出意外的定义了秒数和修改次数，从而对应了配置文件的参数
除了saveparms数组以外，服务器状态redisserver中还维护了一个dirty计数器，以及一个lastsave属性
只有记录了这两个属性，才能跟saveparms里面的配置进行比较，然后符合条件后执行相应的保存逻辑
dirty计数器记录了距离上一次成功执行save或bgsave命令后，服务器对数据库中的数据进行了多少次修改
lastsave是一个时间戳，记录了上一次成功执行save或bgsave命令的时间。
当服务器成功执行一个数据库修改命令后，就会对dirty属性值进行更新
redis服务器的周期性函数servercron默认每隔100毫秒执行一次，这个函数对于正在运行的redis进行维护，它的其中一项工作就是检查配置中的save条件是否满足，如果满足，就执行bgsave。
程序会遍历saveparms数组中的所有保存条件，只要有一个满足，就会开始bgsave
这就是redis服务器根据save配置来进行自动bgsave的原理
~~~

~~~wiki
RDB文件结构：
REDIS - db_version - databases - EOF - check_sum
可以看出RDB文件跟class字节码文件一样，开头使用一个魔数来标识这是一个xx文件，RDB文件开头是REDIS，占用五个字节，程序在载入文件时，快速检查载入的文件是否为RDB文件（rdb文件为二进制文件，查看需要使用od -c xxx.rdb来将其转为asii码格式）
db_version长度为4字节，记录了rdb文件的版本号，比如0006就代表rdb文件的版本为第六版
databases：
如果这个服务器的数据库为空，那么这个部分也为空，否则为非空，这个部分保存了数据库保存的所有键值对和类型等信息
EOF代表rdb正文部分的结束，到了eof，表示最大的一部分区域，也就是所有键值对都已经包括在里面了
最后一个是check_sum
校验和是一个无符号整数，是通过上面四个参数来决定的，主要是为了检查rdb文件是否有出错或者损坏的情况出现
~~~

~~~wiki
databases是一个比较重要的部分
它的结构是 selectdb， db_number key_value_pairs
selectdb毫无疑问，一个rdb文件保存了那么多库，当然得告诉程序后面的命令该在哪个库执行，所以selectdb+db_number构成了选择数据库的逻辑，当读完了db_number后，程序会调用select来执行选库的操作。
key_value_pairs就是保存了所有的键值对，如果键值对带有过期时间，是会一块被保存的，根据键值对的数量类型和内容以及是否有过期时间等，这部分长度不等
key_value_pairs部分：
这部分保存了这个库中所有的键值对。不带过期时间的键值对在rbd文件中以type，key，value三部分组成
type记录了value的类型，比如string，list，hash_ziplist,set_intset等
其实这个type就是redisobject中的encoding属性，记录了当前对象的底层数据结构实现是什么
服务器根据type来判断该以什么样的方式来读入和解释这些value数据
key总是一个字符串对象，value的话会根据type类型来决定，
带过期时间的key会额外多出两个表示过期时间的参数，一个是告诉程序接下来需要读取的是一个以毫秒为单位的过期时间的标志性参数
另一个是真正的过期时间，用毫秒表示，unix时间戳
~~~

~~~wiki
以列表对象为例解释一下 key_value_pairs中的value里面是什么样子的：
对于列表对象，如果type为list，那么表示使用的是linkedlist实现的，如果是list_ziplist，那么底层就是ziplist实现的
对于list编码（linkedlist实现）的value，它的结构是一个list_length 后面跟item，每一个item表示第一个字符串对象。列表元素就被以一个一个item存储
总结一下rdb文件格式，每一个rdb文件格式以redis开头，并且后面跟rdb文件版本号，如果数据库为空，那么dataabases部分为空，如果有数据，那么databases部分包含了selectdb db_number 和key_value_pairs
前两个用来选择正确的库，后面key_value_pairs里面记录的键和值以及过期时间
它的完整组成是三部分，分别是type，key，value，type毫无疑问，由于redis键的多态实现，每一种对象都对应至少两种以上的底层实现方式，所以type比如说是HASH，那么意思就是哈希对象使用字典或者叫hashtable来实现，而不是ziplist来实现的hash对象。每一种不同的对象在databases的key_value_pairs的value部分的实现都是不一样的，学会分析的文件的方式最重要，具体实现细节没必要记住
数据量最大的databases部分的格式大致就是这样，databases完了以后有一个EOF，代表rdb文件的正文部分结束，最终来一个校验和，作用是检查文件是否有损坏等情况
~~~

### AOF

RDB保存数据库状态的方式是将数据库中的键值对记录了一下，就相当于生成了一个数据库的副本一样，而aof持久化数据库状态是将客户端发来的更新命令进行追加，被写入aof中的命令都是以redis的命令请求协议格式进行保存的，在整个aof文件中，其实除了我们发送的修改命令外，只有用于一个指定数据库的select命令是服务器自己添加的，其他都是我们之前通过客户端发送的。

aof做持久化的实现从大体上分为三步：命令追加，文件写入和文件同步（fsync）

命令追加：
当aof持久化功能打开时，每当我们发送一条更新命令，这个命令就会以纯文本的格式发送给服务器中的aof_buf中，aof缓冲区是在redisserver中定义的

文件写入和文件同步：
redis的服务进程就是一个事件循环，其中整个事件循环中的文件事件用于处理接收来自客户端的请求，以及向客户端发送回复，而时间事件负责执行像serverCron函数这样的需要定时运行的函数

因为服务器在执行文件事件来处理请求命令时，避免不了出现写命令，那么aof机制就会把一些内容加入到aof缓冲区，所以在每一次事件循环结束后，需要执行flushaof函数，考虑是否将aof缓冲区中内容写入到aof文件缓存并且同步到文件中

flushaof函数的执行逻辑由服务器配置文件的参数appendfsync选项的值来决定：
1.always：每次事件循环中的文件事件执行完毕后，都要将aof缓冲区中的记录写入并同步到aof文件中

2.everysec：当aof_buf中的所有内容都写入到文件缓存后，如果上次fsync距离现在有了一秒了，那就执行fsync，真正落盘，

这种情况下最极端情况会丢失一秒的数据

3.no，将aof缓冲区中的内容会写到文件缓存，但是刷不刷盘由操作系统决定。

如果没有显示的指定appendfsync的值，那么默认的everysec。

~~~wiki
文件写入和文件同步（fsync）
之所以一个文件写入磁盘的操作还分为两步，主要是为了提高了文件的写入效率，当用户调用write函数后，将一些数据写到磁盘文件时，os通常会将一些内容暂时保存在内存缓冲区，如果缓冲区被填满，或者到了指定的时间，才会真正fsync，这种做法虽然提高了效率，但是如果os carsh了，那么保存在文件系统缓存中的数据都会丢失，为此操作系统除了提供write函数外，还有fsync函数，来让应用程序自己决定是否调用，看应用程序是否能接受不安全的情况
~~~

~~~wiki
服务器配置的appendfsync的值直接决定了系统数据的安全性
当为always，服务器在每个事件循环前都要将aof_buf中的内容写入到aof中，并且同步aof文件，所以always效率最慢，性能最好，但是总体来说是最安全的一个，但是也不是绝对安全，最极端情况是丢失一个事件循环中所产生的的命令
everysec：服务器在每个事件循环都会将缓冲区的内容写到文件缓存，每隔一秒对文件进行一次fsync，从效率上来讲，不是很慢，最多会丢失一秒的数据
no：服务器在每个事件循环都会将缓冲区写入文件缓存， 至于刷盘完全交给os，虽然它的写入时间最快，但是os缓存中积累的内容过多，最终你还是要fsync，它的单次同步时长是最长的，所以平摊下来的话与everysec差不多，只不过安全性太差，最坏情况会丢失上一次aof同步后到crash时刻的所有内容
~~~

AOF的文件载入与数据还原：
aof里面包含了重建数据裤状态所需要的所有命令，所以只需要将这些命令做一个执行，就可以在crash后恢复。

详细步骤：

1.创建一个不带网络连接的伪客户端，因为redis命令只能在客户端上下文执行，而载入aof文件的过程中需要的命令都在aof中保存好了，所以这个过程不需要网络连接，但是要客户端上下文，所以redis使用一个不带网络连接的伪客户端来执行aof中的命令，从而达到redis服务器执行带网络连接的客户端一样的效果

2.从aof中取出语句

3.使用这个伪客户端执行命令

4.重复2，3步，直到aof文件到达末尾。

并且一定注意aof文件中的第一个命令肯定是select选库

### AOF重写

因为aof是不停的记录客户端的写命令，随着服务器运行时间的流逝，aof文件的内容会越来越多，为此aof重写机制出现，来解决aof文件过大的问题。

为了解决aof文件膨胀的问题，redis提供了aof重写，整体上来说就是通过建立一个新的aof文件来代替原来的aof文件，新旧两个aof对数据库状态的恢复效果相同，新aof不会出现冗余命令

AOF文件重写原理：
aof文件重写虽然名字叫重写，其实新aof跟旧aof一点互动都没有（其实后面原子改名也是一种互动。。），为什么说没有互动，是因为新aof的创建是通过读取当前数据库的状态来实现的

比如sadd 1 ，sadd 2，sadd3，这三条命令在旧的aof中，那么新aof直接读取当前数据库状态，一条命令搞定，sadd 1,2,3

这就是aof重写的基本原理

整个过程的伪代码（思路）：

~~~wiki
创建aof文件
遍历数据库
忽略空数据库
写入select命令，选择库
遍历数据库中的所有键
忽略过期键
根据键的类型进行重写
写入完毕
~~~

这种方式的aof重写只会在aof文件包含还原到当前数据库状态的必要命令，所以aof文件的大小在一个可以接收的范围之内

上面提到使用一条sadd来添加集合中所有的元素，如果这个集合中的元素数量过大，阈值是64个元素，如果超过64个元素，那么需要使用不止一个命令，比如超过了64小于128就用两个sadd

这种做法是为了避免执行命令过大造成伪客户端的输入缓冲区溢出

### AOF后台重写（bgrewriteaof实现原理）

思考这样一种问题，就是aof的重写需要写入大量的命令，所以这个aof重写函数的线程需要长时间阻塞，因为redis服务器使用单线程来处理请求，那么在aof重写期间，会阻塞其他所有客户端发来的读写请求，这显然是无法接收的

所以Redis的做法是使用子进程的方式执行aof重写程序，以此达到两个目的：
1.子进程进行aof重写期间，不会影响服务器进程，也就是父进程的处理请求

2.子进程带有父进程的数据副本，使用子进程而不使用线程的好处是在不使用锁的情况下也不会出现数据不安全的情况

不过子进程虽然很好的解决了redis服务器进程的处理命令的阻塞问题，但是又带来一个新问题，由于子进程与父进程同时在执行，那么可能会出现新的命令被执行，然后子进程在做aof重写，无法感知到新来的命令，造成重写后的aof与当前数据库的数据不一致的情况，这是一个非常重要的问题

为了解决重写后的aof文件与数据库状态不一致的问题，redis服务器又设置了一个aof重写缓冲区，那么这时候在子进程执行aof重写的时候，就需要多做一些事情了

1.主进程不仅要执行客户端发来的命令

2.并且照例将写命令加到aof缓冲区

3.并且多了一步，将写命令加入到aof重写缓冲区

这样子的步骤，对于aof缓冲区内容会被写入到aof文件中，aof文件这块没有影响，然后对于aof重写缓冲区中的内容，在子进程完成对原来的aof文件重写操作后，需要给主进程发送一个信号，主进程收到以后，调用一个处理函数：

1.将aof重写缓冲区中的内容写到新的aof文件中，这时候重写后的aof，也就是新aof，它的内容就跟数据库中的状态完全一致了

2.新aof到此一切就绪，然后对新aof进行改名，原子覆盖旧aof，完成对新旧aof的交替

这就是bgrewriteaof的实现原理

关于AOF的总结：

~~~wiki
aof通过保存所有修改数据库的命令来记录数据库的最新状态，从而达到数据持久化以及crash-safe的目的
aof文件所有命令都以纯文本的形式，或者叫redis请求协议格式来保存
命令请求先到aof缓冲区，然后被写到文件，并且按配置同步刷盘
服务器只需要载入aof文件并重新执行保存在aof中的命令，就可以还原数据库的状态
aof重写会产生一个新的aof文件，体积更小，但是整个过程跟旧aof没有关系，只是最后新旧交替会覆盖旧aof
其实aof重写这个名字有一些误导，因为他是根据当前数据库状态来实现的，程序无需对现有aof进行任何的读取操作
bgrewriteaof通过在子进程重写aof的同时，主进程不断的向aof重写缓冲区中写入最新内容，当子进程执行完毕后给主进程发送信号，主进程将aof重做缓冲区的内容写入到新aof中，然后改名并原子覆盖，从而达到aof重写的效果，并且不会让父进程阻塞，并且数据一定是一致的
~~~

AOF补充：
通常情况下，redis只会那些对数据库做了修改的命令写入到aof中，并复制到各个从服务器，如果一个命令没有对数据库进行过任何修改，那么redis会认为这是一个只读命令，这个命令不会被写入到aof文件，当然也不会被复制到服务器中

这个规则适用于大部分redis的命令，但是pubsub命令和script load命令是例外，pubsub虽然没有修改redis数据库，但是他会向频道的所有的订阅者发送消息，为了弥补这个行为产生的副作用，所以redis会这个pubsub加上redis_force_aof标志，强制将这个命令也写入到aof文件中，这样将来在载入aof文件时，就会再次执行这个命令，来达到恢复前的一致性效果

同样的，对于script load，也没有修改数据库，但是修改了服务器状态，那么也会使用redis_force_aof标志强行写入aof，另外，为了让主从服务器都可以正确的载入script load命令所指定的脚本，服务器还需要使用redis_force_repl标志，强制将script load命令复制给其他所有从服务器

redis_force_repl：强制主服务器将当前执行的命令复制给所有的从服务器（这两个标志都是在redisclient中的flag属性）

aofclient是一个不带网络连接的伪客户端，另外需要注意，lua_client也是一个伪客户端，并且luaclient会在服务器初始化的时候就会创建，并且这个伪客户端会一直存在，而aofclient仅在需要载入aof文件的时候进行创建，并且载入完成以后，redis服务器会关闭这个伪客户端

