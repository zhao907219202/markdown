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


