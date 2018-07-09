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
命令之前，服务器应该对占用内存状况进行检查，因为这个命令可能会占用大量内存。
