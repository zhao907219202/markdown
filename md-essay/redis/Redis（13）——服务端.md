Redis服务器负责与多个客户端建立网络连接，处理客户端发送的命令请求，在数据库中保存客户端执行命令所产生的数据，并通过资源管理来维持服务器自身的运转。
#### 命令请求的执行过程
##### 发送命令请求
Redis服务器的命令请求来自Redis客户端，当用户在客户端中键入一个命令请求时，客户端会将这个命令请求转换成协议格式，然后通过连接到这个服务器的套接字，
将协议格式的命令请求发送给服务器。
##### 读取命令请求
当客户端与服务器之间的连接套接字因为客户端的写入而变得可读时，服务器将调用命令请求处理器来执行一下操作：

1. 读取套接字中协议格式的命令请求，并将其保存到客户端状态的输入缓冲区里面
2. 对输入缓冲去的命令请求进行分析，提取出命令请求中包含的命令参数，以及命令参数的个数，然后分别将参数和参数个数保存到客户端状态的argv和argc属性里面。
3. 调用命令执行器，执行客户端指定的命令。

##### 命令执行器（1）：查找命令实现
命令执行器要做的第一件事就是根据客户端状态argv[0]参数，在命令表（command table）中查找参数所指定的命令，并将找到的命令保存到客户端状态的cmd属性里面。命令表
是一个字典，字典的键是一个个命令名字，字典的值是一个个redisCommand结构，每个redisCommand结构记录了一个Redis命令的实现信息。

```
struct redisCommand {
    char *name;
    redisCommandProc *proc;
    int arity;
    char *sflags; /* Flags as string representation, one char per flag. */
    int flags;    /* The actual flags, obtained from the 'sflags' field. */
    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    redisGetKeysProc *getkeys_proc;
    /* What keys should be loaded in background when calling this command? */
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */
    long long microseconds, calls;
};
```

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-server-0-20180709.png)

sflags 属性可以使用的标识值，以及这些标识的意义。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-server-1-20180709.png)

举个例子，SET命令的name是“set”，实现函数为setCommand；命令的参数个数为-3，表示命令接受三个或以上数量的参数；命令的标识为“wm”，表示SET命令是一个写入命令，并且在执行这个
命令之前，服务器应该对占用内存状况进行检查，因为这个命令可能会占用大量内存。命令表中redisCommand的状态如下图所示：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-server-2-20180709.png)

##### 命令执行器（2）：执行预备操作
到目前为止，服务器已经将执行命令所需的命令实现函数（保存在客户端状态的cmd属性）、参数（保存在客户端状态的argv属性）、参数个数（argc属性）都收集齐了，但是在真正执行命令之前
程序还需进行一些预备操作，从而确保命令可以正确、顺利地被执行，这些操作包括：

+ 检查客户单状态的cmd指针是否指向null，如果是，则找不到该命令，返回错误
+ 根据客户端cmd属性指向的redisCommand结构的arity属性，检查命令请求所给定的参数个数是否正确，当参数个数不正确时，返回错误
+ 检查客户端是否已经通过了身份验证，未通过身份验证的客户端执行执行AUTH命令，如果未通过身份验证的客户端试图执行其他命令，返回错误
+ 如果服务器打开了maxmemory功能，那么在执行命令之前，先检查服务器的内存占用情况，并在有需要时进行内存回收，从而使得接下来的命令可以顺利执行。如果内存回收失败，返回错误
+ 如果服务器上一次执行BGSAVE命令出错，并且服务器打开了stop-writes-on-bgsave-error功能，而且服务器即将要执行的是一个写命令，那么服务器将返回错误
+ 如果客户端当前正在用SUBSCRIBE命令订阅频道，或者正在用PSUBSRIBE命令订阅模式，那么服务器只会执行客户端发来的SUBSCRIBE、PSUBSCRIBE、UNSUBSCRIBE、PUNSUBSCRIBE命令，其他返回错误
+ 如果服务器正在执行数据载入，那么客户端发送的命令必须带有l标识（比如INFO、SHUTDOWN、PUBLISH）才会被执行，其他命令都会被服务器拒绝
+ 如果服务器因为执行Lua脚本而超时并进入阻塞状态，那么服务器只会执行客户端发来的SHUTDOWN nosave命令和SCRIPT KILL 命令，其他命令拒绝执行
+ 如果客户端正在执行事务，那么服务器只会执行客户端发来的EXEC、DISCARD、MULTI、WATCH四个命令，其他命令都会被放进事务队列中
+ 如果服务器打开了监视器功能，那么服务器会将要执行的命令和参数等信息发送给监视器。

（以上是单机模式下的预备操作，复制或者集群模式下，预备操作更多）

##### 命令执行器（3）：调用命令的实现函数
当服务器决定要执行命令时，它只要执行一下语句就可以了：
```
// client 是指向客户端状态的指针
client->cmd->proc(client);
```
被调用的命令实现函数会执行执行的操作，并产生相应的命令回复，这些回复会被保存在客户端状态的输出缓冲区里（buf属性和reply属性），之后实现函数还会为客户端的套接字关联命令回复
处理器，这个处理器负责将命令回复返回给客户端。

##### 命令执行器（4）：执行后续工作
服务器还需要执行一些后续工作：

+ 如果服务器开启了慢查询日志功能，那么慢查询日志模块会检查是否需要为刚刚执行完的命令请求添加一条新的慢查询日志
+ 根据刚刚执行命令所耗费的时长，更新被执行命令的redisCommand结构的milliseconds属性，并将命令的redisCommand结构的calls计数器增一
+ 如果服务器开启了AOF持久化功能，那么AOF持久化模块会将刚刚执行的命令请求写入到AOF缓冲区里面
+ 如果有其他服务器正在复制当前这个服务器，那么服务器会将刚刚执行的命令传播给所有从服务器

##### 将命令回复发送给客户端
命令实现函数会将命令回复保存到客户端的输出缓冲区，并关联命令回复处理器，当客户端套接字变为可写状态时，服务器就会执行命令回复处理器，将保存在客户端输出缓冲区中的命令
回复发送给客户端。当命令回复发送完毕之后，回复处理器会清空客户端状态的输出缓冲去，为处理下一个命令请求做好准备。

##### 客户端接受并打印命令回复
当客户端接收到协议格式的命令回复之后，它会将这些回复转换成人类可读的格式，并打印给用户观看

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-server-3-20180709.png)

#### serverCron函数
Redis服务器中的serverCron函数默认每隔100毫秒执行一次，这个函数负责管理服务器的资源，并保持服务器自身的良好运转。
##### 更新服务器时间缓存
Redis服务器中有不少功能需要获取系统的当前时间，而每次获取系统的当前时间都需要执行一次系统调用，为了减少系统调用的执行次数，服务器状态中的unixtime属性和mstime属性被
用作当前时间的缓存：
```
struct redisServer{
    time_t unixtime;   // 秒级时间戳
    long long mstime;  // 毫秒级时间戳
}
```
serverCron函数默认每100毫秒一次的频率更新unixtime属性和mstime属性，所以这两个属性记录的时间的精准度并不高。因此只会在打印日志、更新服务器的LRU时钟，决定是否执行持久化任务、
计算服务器上线时间这些时间精准度要求不高的功能上使用上面两个缓存，对于为键设置过期时间、添加慢查询日志这种需要高精度时间的功能，服务器会再次执行系统调用。

##### 更新LRU时钟
服务器状态中的lrulock属性保存了服务器的lru时钟，这个属性和上面介绍的unixtime属性、mstime属性一样，都是服务器时间缓存的一种：
```
struct redisServer {
    // 默认每10秒更新一次的时钟缓存
    // 用于计算键的空转时长
    unsigned lruclock;
}
```
每个Redis对象都会有一个lru属性，这个lru属性保存了对象最后一次被访问的时间：
typedef struct redisObject{
    unisigned lru;
}robj;
当数据库要计算一个数据库键的空转时间（也就是数据库键对应的值对象的空转时间），程序会用服务器的lruclock属性记录的时间减去对象的lru属性记录的时间，得出计算结果就是这个
对象的空转时间。
serverCron函数默认会以每100毫秒一次的频率更新lruclock属性的值，因为这个时钟不是实时的，所以根据计算出来的LRU时间实际上只是一个模糊的估算值。

##### 更新服务器每秒执行命令次数
serverCron函数中的trackOperationsPerSecond函数会以每100毫秒一次的频率执行，这个函数的功能是以抽样计算的方式，估算并记录服务器在最近一秒钟处理的命令请求数量，这个值可以
通过INFO status命令的 instantaneous_ops_per_sec 域查看。trackOperationsPerSecond 函数和服务器状态的四个属性有关。
```
 /* The following two are used to track instantaneous metrics, like
     * number of operations per second, network traffic. */
    struct {
        long long last_sample_time; /* Timestamp of last sample in ms */
        long long last_sample_count;/* Count in last sample */
        // 数组中每个项都记录了一次抽样结果（环形数组）
        long long samples[STATS_METRIC_SAMPLES];
        // 每次抽样后自增1，等于16重置为0
        int idx;
    } inst_metric[STATS_METRIC_COUNT];
```
trackOperationsPerSecond 函数每次运行，都会根据 last_sample_time 记录的上一次抽样时间和服务器当前时间，以及 last_sample_count 记录的上一次抽样已执行命令数量和服务器当前
已执行命令数量，计算出两次 trackOperationsPerSecond 调用之间，服务器平均每一毫秒处理了多少个命令请求，然后乘1000，这就得到了服务器在一秒钟内能处理多少个命令请求的估计值，
这个估计值会被作为一个新的数组项放进samples环形数组里面。当客户端执行INFO命令，服务器会调用getOperationPerSecond函数，这个函数会将环形数组的所有值加和再求平均值，然后返回
出去，并且就是instantaneous_ops_per_sec的值。

##### 更新服务器内存峰值记录
serverCron函数执行，程序会查看服务器当前使用的内存数量，并与redisServer中的stat_peak_memory属性值比较，如果比该值大，则存入。INFO memory命令的 used_memory_peak 和 used_memory_peak_human
两个域记录了服务器的内存峰值。

##### 处理SIGTERM信号
服务器启动会为服务器进程的SIGTERM信号关联处理器sigtermHandler函数，这个信号处理器负责在服务器接到SIGTERM信号时，打开服务器状态的shutdown_asap标识，每次serverCron函数运行时，
程序都会对服务器状态的shutdown_asap属性进行检查，并根据属性的值决定是否关闭服务器（服务器在关闭之前会进行RDB持久化操作）。

##### 管理客户端资源
serverCron函数每次执行都会调用clientsCron函数，clientsCron函数会对一定数量的客户端进行一下两个检查：

+ 如果客户端与服务器之间的连接已经超时，那么释放这个客户端
+ 如果客户端在上一次执行命令请求之后，输入缓冲区的大小超过了一定的长度，那么程序会释放客户端当前的输入缓冲区，并重新创建一个默认大小的输入缓冲区，从而防止客户端的输入缓冲
耗费过多的内存

##### 管理数据库资源
serverCron函数每次执行都会调用databasesCron函数，这个函数会对服务器中的一部分数据库进行检查，删除其中的过期键，并在有需要时，对字典进行收缩操作。

##### 执行被延迟的BGREWRITEAOF
在服务器执行BGSAVE命令期间，如果客户端向服务器发来 BGREWRITEAOF 命令，那么服务器会将 BGREWRITEAOF 延迟到 BGSAVE 命令执行完毕之后。redisServer中的 aof_rewrite_scheduled 属性记录是否有
BGREWRITEAOF 被延迟，每次serverCron执行，检查到BGSAVE、BGREWRITEAOF 都没有在执行，并且aof_rewrite_scheduled为1，那么服务器就会执行 BGREWRITEAOF。

##### 检查持久化操作的运行状态
服务器状态使用 rdb_child_pid 属性和 aof_child_pid 属性记录执行 BGSAVE 命令和 BGREWRITEAOF 命令的子进程ID，这两个属性也可以用于检查BGSAVE命令或者BGREWRITEAOF命令是否正在执行。
每次serverCron函数执行时，程序都会检查 rdb_child_pid 和 aof_child_pid 两个属性的值，只要其中一个不为-1，程序就会执行一次wait3函数，检查子进程是否有信号发来服务器进程：

+ 如果有信号到达，那么表示新的RDB文件已经生成完毕（对于BGSAVE命令来说），或者AOF文件已经重写完毕（对于 BGREWRITEAOF 命令来说），服务器需要进行相应命令的后续操作，比如用新的RDB文件
替换现有的RDB文件，或者用重写后的AOF文件替换现有的AOF文件。
+ 如果没有信号到达，那么表示持久化操作未完成，程序不做操作。

另一方面，如果两个属性的值都是-1，那么表示服务器没有在进行持久化操作，这时，程序执行以下三个检查:

+ 查看是否有 BGREWRITEAOF 被延迟了，如果有，开始执行 BGREWRITEAOF
+ 检查服务器的自动保存条件是否已经满足，如果条件已经满足，并且服务器没有在执行其他持久化操作，那么服务器开始一次新的BGSAVE操作（因为条件1可能执行 BGREWRITEAOF，所以这里再次确认服务器
没有在执行持久化操作）
+ 检查服务器的AOF重写条件是否满足，如果满足，并且服务器没有在执行持久化操作，那么服务器将开始一次新的 BGREWRITEAOF 操作。



