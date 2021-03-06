redis作为一个内存服务器，他需要处理很多来自外部的网络请求，他使用IO多路复用机制同时监听多个文件描述符的可读和可写状态，一旦收到网络请求就会在内存中快速处理，由于绝大多数操作都是内存操作，所以处理速度会非常快。在redis4.0以后，在部分命令上使用了多线程机制，例如 `UNLINK`、`FLUSHALL ASYNC`、`FLUSHDB ASYNC` 等非阻塞的删除操作，这些操作会使用非主线程之外的线程去处理

无论是使用单线程模型还是多线程模型，两个设计上的决定都是为了更好地提升redis的开发效率，执行效率。

redis一开始选择单线程模型来处理来自客户端的绝大多数网络请求，这种考虑是多方面的，比如 ：
1.使用单线程模型带来更好的可维护性，方便开发和调试

2.使用单线程模型也能并发的处理客户端的请求

3.redis服务的绝大多数操作的性能瓶颈不是CPU，而是内存和网络IO

redis官方对于redis老版本使用单线程的解释是：redis的瓶颈不是CPU，而往往是网络带宽和内存大小，另外单线程没有线程的上下文切换，不会存在这个方面的消耗，另外如果CPU成了redis的瓶颈，官方也说了可以使用redis cluster，因为redis是key value数据库，而不是关系型数据库，数据之间没有任何约束，只要客户端搞清楚哪个key该放在哪个redis进程中，redis-cluster可以用多进程的方式让你的多核CPU利用起来

redis可以处理高并发请求吗？

这个是可以的，并发性io流，意味着能够让一个计算单元处理来个多个客户端的请求，并行性意味着服务器能够同时处理多个事情，具有多个计算单元，如果使用单线程无法发挥多核CPU的能力，可以在单机开多个redis实例，这里一直强调的单线程指的是，在处理网络请求的时候只有一个线程来处理，一个正式的redis server运行的时候肯定不止一个线程

redis在处理客户端请求的时候，包括获取socket（读），解析，执行，内容返回（socket写）等都由一个串行化执行的主线程处理，这就是所谓的单线程，到了redis4.0以后，除了主线程以外，他也有其他后台线程来处理一些较为缓慢的操作，例如清理脏数据，无用连接的释放，大key的删除等操作

在redis6.0中都没有默认启用多线程，还只是启用单线程，如需开启需要修改redis.conf配置文件：io-threads-do-reads yes

另外在开启多线程后还需要设置线程数，否则是不生效的，关于线程数的设置，官方建议4核心的机器设置为2到3个线程，8核建议6个线程，线程数一定小于机器核数，官方认为线程数超过8后基本没什么意义

如果开启redis的多线程，至少要4核心的机器，且redis实例已经占用相当大的CPU耗时的时候才建议采用，否则使用多线程是没有意义的



