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

~~~

