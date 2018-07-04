Redis服务器是典型的一对多服务器程序，一个服务器可以与多个客户端建立网络连接，每个客户端可以向服务器发送命令请求，而服务器则接受
并处理客户端发送的命令请求，并向客户端返回命令回复。通过使用由I/O多路复用技术实现的文件事件处理器，Redis服务器使用单线程单进程的
方式来处理命令请求，并与多个客户端进行网络通信。

对于每个与服务器进行连接的客户端，服务器都为这些客户端建立了相应的redis.h/redisClient 结构（客户端状态），这个结构保存了客户端当前
的状态信息，以及执行相关功能时需要用到的数据结构。其中包括：客户端的套接字描述符；客户端的名字；客户端的标志值（flag）；指向客户端
正在使用的数据库的指针，以及该数据库的号码；客户端当前要执行的命令、命令的参数、命令参数的个数，以及指向命令实现函数的指针；客户端
的输入缓冲和输出缓冲；客户端复制状态信息，以及进行复制所需的数据结构；客户端执行BRPOP、BLPOP等列表阻塞命令时使用的数据结构；客户端
的事务状态，以及执行WATCH命令时用到的数据结构；客户端执行发布与订阅功能时用到的数据结构；客户端的身份验证标志；客户端的创建时间，和
服务器最后一次通信的时间，以及客户端的输出缓冲区大小超出软性限制（soft limit）的时间。

```
/* With multiplexing we need to take per-client state.
 * Clients are taken in a linked list. */
typedef struct client {
    uint64_t id;            /* Client incremental unique ID. */
    int fd;                 /* Client socket. */
    redisDb *db;            /* Pointer to currently SELECTed DB. */
    robj *name;             /* As set by CLIENT SETNAME. */
    sds querybuf;           /* Buffer we use to accumulate client queries. */
    sds pending_querybuf;   /* If this is a master, this buffer represents the
                               yet not applied replication stream that we
                               are receiving from the master. */
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size. */
    int argc;               /* Num of arguments of current command. */
    robj **argv;            /* Arguments of current command. */
    struct redisCommand *cmd, *lastcmd;  /* Last command executed. */
    int reqtype;            /* Request protocol type: PROTO_REQ_* */
    int multibulklen;       /* Number of multi bulk arguments left to read. */
    long bulklen;           /* Length of bulk argument in multi bulk request. */
    list *reply;            /* List of reply objects to send to the client. */
    unsigned long long reply_bytes; /* Tot bytes of objects in reply list. */
    size_t sentlen;         /* Amount of bytes already sent in the current
                               buffer or object being sent. */
    time_t ctime;           /* Client creation time. */
    time_t lastinteraction; /* Time of the last interaction, used for timeout */
    time_t obuf_soft_limit_reached_time;
    int flags;              /* Client flags: CLIENT_* macros. */
    int authenticated;      /* When requirepass is non-NULL. */
    int replstate;          /* Replication state if this is a slave. */
    int repl_put_online_on_ack; /* Install slave write handler on ACK. */
    int repldbfd;           /* Replication DB file descriptor. */
    off_t repldboff;        /* Replication DB file offset. */
    off_t repldbsize;       /* Replication DB file size. */
    sds replpreamble;       /* Replication DB preamble. */
    long long read_reploff; /* Read replication offset if this is a master. */
    long long reploff;      /* Applied replication offset if this is a master. */
    long long repl_ack_off; /* Replication ack offset, if this is a slave. */
    long long repl_ack_time;/* Replication ack time, if this is a slave. */
    long long psync_initial_offset; /* FULLRESYNC reply offset other slaves
                                       copying this slave output buffer
                                       should use. */
    char replid[CONFIG_RUN_ID_SIZE+1]; /* Master replication ID (if master). */
    int slave_listening_port; /* As configured with: SLAVECONF listening-port */
    char slave_ip[NET_IP_STR_LEN]; /* Optionally given by REPLCONF ip-address */
    int slave_capa;         /* Slave capabilities: SLAVE_CAPA_* bitwise OR. */
    multiState mstate;      /* MULTI/EXEC state */
    int btype;              /* Type of blocking op if CLIENT_BLOCKED. */
    blockingState bpop;     /* blocking state */
    long long woff;         /* Last write global replication offset. */
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
    dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */
    list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) */
    sds peerid;             /* Cached peer ID. */

    /* Response buffer */
    int bufpos;
    char buf[PROTO_REPLY_CHUNK_BYTES];
} client;
```


服务器状态结构的clients属性是一个链表，这个链表保存了所有与服务器连接的客户端的状态结构，对客户端执行批量操作，或者查找某个指定的客户端，
都可以通过遍历clients链表来完成：

```
struct redisServer{
    //,.,
    // 一个链表，保存了所有客户端状态
    list * clients;
}

```

#### 客户端属性
客户端状态包含的属性可以分为两类：

+ 一类是比较通用的属性，这些属性很少与特定功能相关，无论客户端执行的是什么工作，他们都要用到这些属性。
+ 另外一类是和特定功能相关的属性，比如操作数据库时需要用到的db属性和dictid属性，执行事务时需要使用的mstate属性，以及执行WATCH命令时需要用到的watched_keys属性等等。

##### 套接字描述符
客户端状态的fd 属性记录了客户端正在使用的套接字描述符：

+ 伪客户端（fake client）的fd属性的值为-1,：伪客户端处理的命令请求来源于AOF文件或者lua脚本，而不是网络，所以这种客户端不需要套接字连接，自然也不需要记录套接字描述符。
+ 普通客户端的fd属性的值为大于-1的整数，普通客户端使用套接字来与服务器进行通信，所以服务器会用fd属性来记录客户端套接字的描述符。因为合法的套接字描述符不能是-1，所以
普通客户端的套接字描述符的值必然是大于-1的整数。

##### 名字
在默认情况下，一个连接到服务器的客户端是没有名字（name）的。
使用CLIENT setname 命令可以为客户端设置一个名字，让客户端的身份变得更清晰。

##### 标志
客户端的标志属性flags记录了客户端的角色（role），以及客户端目前所处的状态，多个标志可以进行二进制或运算：flags= <flag1> | <flag2> | ..

+ 在主从服务器进行复制操作时，主服务器会成为从服务器的客户端，而从服务器也会成为主服务器的客户端，REDIS_MASTER标志表示客户端代表的是一个主服务器，REDIS_SLAVE标志表示客户端
代表的是一个从服务器。
+ REDIS_PRE_PSYNC 标志表示客户端代表的是一个版本低于Redis2.8的从服务器，主服务器不能使用PSYNC命令（老版本复制同步命令）与这个从服务器进行同步，这个标志只能在REDIS_SLAVE标志处于打开时使用。
+ REDIS_LUA_CLIENT 标志表示客户端是专门用于处理Lua脚本里面包含的Redis命令的伪客户端

另外一部分标志则记录了客户端目前所处的状态：
+ REDIS_MONITOR 标志表示客户端正在执行MONITOR命令
+ REDIS_UNIX_SOCKET 标志表示服务器使用UNIX套接字来连接客户端
+ REDIS_BLOCKED 标志表示客户端正在被BRPOP、BLPOP等命令阻塞
+ REDIS_UNBLOCKED 标志表示客户端已经从REDIS_BLOCKED标志所表示的阻塞状态中脱离出来，不再阻塞。该标志只能在REDIS_BLOCKED标志已经打开的情况下使用。
+ REDIS_MULTI 标志表示客户端正在执行事务
+ REDIS_DIRTY_CAS 标志表示事务使用WATCH命令监视的数据库已经被修改，REDIS_DIRTY_EXEC标志表示事务在命令入队时出现了错误，以上两个标志都表示事务的安全性已经被破坏，只要这两个
标记中的任意一个被打开，EXEC命令必然会执行失败。这两个标志只能在客户端打开了REDIS_MULTI标志的情况下适用。
+ REDIS_CLOSE_ASAP 标志表示客户端的输出缓冲区大小超出了服务器允许的范围，服务器会在下一次执行serverCron函数时关闭这个客户端，以免服务器的稳定性受到这个客户端影响。积存在输
出缓冲区的所有内容直接被释放，不会返回给客户端。
+ REDIS_CLOSE_AFTER_REPLY 标志表示有用户对这个客户端执行了CLIENT KILL 命令，或者客户端发送给服务器的命令请求中包含了错误的协议内容，服务器会将客户端积存在输出缓冲区的所有内
荣返回给客户端，然后关闭客户端。
+ REDIS_ASKING 标志表示客户端向集群节点发送了ASKING命令
+ REDIS_FORCE_AOF 标志表示强制服务器将当前执行的命令写入到AOF文件里面，REDIS_FORCE_REPL 标志强制主服务器当前执行的命令复制给所有从服务器。执行 PUBSUB 命令会使客户端打开
REDIS_FORCE_AOF标志，执行SCRIPT LOAD 命令会使客户端打开REDIS_FORCE_AOF 标志和REDIS_FORCE_REPL标志
+ 在主从服务器进行命令传播期间，从服务器需要向主服务器发送REPLICATION ACK命令，在发送这个命令之前，从服务器必须打开主服务器对应的客户端的 REDIS_MASTER_FORCE_REPLY 标志，否
则发送操作会被拒绝执行。

##### 输入缓冲区
客户端状态的输入缓冲区（querybuf）用于保存客户端发送的命令请求：输入缓冲区的大小会根据输入内容动态的缩小或者扩大，但它的最大大小不能超过1GB，否则服务器将关闭这个客户端。

###### 命令与命令参数
在服务器将客户端发送的命令请求保存到客户端状态的queybuf属性之后，服务器将对命令请求的内容进行分析，并将得出的命令参数以及命令参数的个数分别保存到客户端状态的argv属性和
argc属性：

```
 int argc;               /* Num of arguments of current command. */
 robj **argv;            /* Arguments of current command. */
```
argv 属性是个数组，数组中的每一项都是字符串对象，其中 argv[0] 是要执行的命令，而之后的其他则是传给命令的参数。argc记录了argv数组的长度。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-client-0-20180704.png)

##### 命令的实现函数
当服务器从协议内容中分析得出argv 和 argc属性后，服务器会根据项argv[0]的值，在命令表中查找命令对应的命令实现函数。如图，命令表实际上是个字典，键是SDS，保存命令的名字；
值是redisCommand结构，这个结构保存了命令的实现函数、命令的标志、命令应该给定的参数个数、命令的总执行次数和总耗时。当程序在命令表中成功找到argv[0]所对应的redisCommand
结构时，它会将客户端状态的cmd指针指向这个结构。之后服务器可以使用cmd属性指向的redisCommand结构，以及argv、argc属性中的命令参数信息，调用命令实现函数，执行客户端指定的
命令。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-client-1-20180704.png)

##### 输出缓冲区
执行命令所得的命令回复会被保存在客户端状态的输出缓冲区里，每个客户端都有两个输出缓冲区，一个缓冲区的大小是固定的，另一个缓冲区的大小是可变的。固定大小的缓冲区用于保存那些
长度比较小的回复，比如OK、简短的字符串值、整数值、错误回复等。可变大小的缓冲区用于保存那些长度比较大的回复，比如一个非常长的字符串值，一个由很多项组成的列表，一个包含了很多
元素的集合等等。

+ 固定大小缓冲区由buf 和bufpos 两个属性组成，buf是一个大小为REDIS_REPLY_CHUNK_BYTES（默认16*1024,16KB）字节的字节数组，而bufpos属性则记录了buf数组目前已使用的字节数量。
+ 可变大小缓冲区由reply链表和一个或多个字符串对象组成，通过使用链表来连接多个字符串对象，服务器可以为客户端保存一个非常长的命令回复，而不必受到固定大小缓冲区16KB大小的限制。

##### 身份验证
客户端状态的authenticated 属性用于记录客户端是否通过了身份验证：如果为0，那么表示客户端未通过身份验证，如果值为1，那么表示客户端已经通过了身份验证。该属性只有在配置文件对
requirepass 选项打开的情况下使用。

##### 时间
客户端还有几个和时间有关的属性，ctime属性记录了创建客户端的时间，用来记录客户端与服务端已经连接了多少秒；lastinteraction 属性记录了客户端与服务器最后一次进行互动的时间，这里的
互动可以是客户端向服务端发送命令请求，也可以是服务端向客户端发送命令回复，这个属性可以用来计算客户端的空转（idle）时间，CLIENT list 命令的idle域记录了这个空转秒数。
obuf_soft_limit_reached_time属性记录了输出缓冲区第一次到达软性限制（soft limit）的时间，下面说这个属性的用途。

#### 客户端的创建和关闭
##### 创建普通客户端
如果客户端是通过网络连接与服务器进行连接的普通客户端，那么在客户端使用connect函数连接到服务器时，服务器就会调用连接事件处理器，为客户端创建相应的客户端状态，并将这个新的客户端
状态添加到服务器状态结构clients链表的末尾。

###### 关闭普通客户端
一个普通客户端可以因为多种原因而被关闭：

+ 如果客户端进程退出或者被杀死，那么客户端与服务器之间的网络连接将被关闭，从而造成客户端被关闭。
+ 如果客户端向服务器发送了带有不符合协议格式的命令请求，那么这个客户端也会被服务器关闭。
+ 如果客户端成为了CLIENT KILL 命令的目标，那么它也会被关闭。
+ 如果用户为服务器设置了timeout配置选项，那么当客户端的空转时间超过timeout选项设置的值时，客户端将被关闭。不过timeout选项有一些例外情况：如果客户端是主服务器（打开REDIS_MASTER
标志），从服务器（打开REDIS_SLAVE标志），正在被BLPOP等命令阻塞（打开了REDIS_BLOCKED标志），或者正在执行SUBSCRIBE、PSUBSCRIBE等订阅命令，那么即使客户端的空转时间超过了timeout
选项的值，客户端也不会被关闭。
+ 如果客户端发送的命令请求大小超过了输入缓冲区的大小限制（默认1GB），那么这个客户端会被服务器关闭。
+ 如果要发送给客户端的命令回复的大小超过了输出缓冲区的限制大小，那么这个客户端会被服务器关闭。

客户端输出缓冲区可变大小缓冲区是一个链表和任意多个字符串对象组成，理论上说，这个缓冲区可以保存任意长的命令回复。但是为了避免客户端的回复过大，占用过多的服务器资源，服务器会
时刻检查客户端的缓冲区大小，并在缓冲区的大小超出范围时，执行相应的限制操作。

服务器使用两种模式来限制客户端输出缓冲区的大小：

+ 硬性限制（hard limit）：如果输出缓冲区的大小超过了硬性限制所设置的大小，那么服务器立即关闭客户端。
+ 软性限制（soft limit）：如果输出缓冲区的大小超过了软性限制所设置的大小，但还没有超过硬性限制，那么服务器将使用服务器状态结构的 obuf_soft_limit_reached_time 属性记录下客户端
到达软性限制的起始时间，之后服务器会继续监视客户端，如果输出缓冲区的大小一直超出软性限制，并且持续时间超过服务器设定的时长，那么服务器就会关闭客户端，相反地，如果客户端在指
定时间内不再超出软性限制，那么客户端就不会被关闭，并且obuf_soft_limit_reached_time也会被清零。

使用client-output-buffer-limit 可以为普通客户端、从服务器客户端、执行发布与订阅功能的客户端分别设置不同的软性限制和硬性限制，格式为：

```
client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
# 例如：
client-output-buffer-limit normal 0 0 0  #不限制输出缓冲区大小
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60 #执行发布和订阅功能的客户端硬性限制为32mb，软性限制为8mb，软性限制时长为60秒
```

##### Lua脚本的伪客户端
服务器会在初始化时创建负责执行Lua脚本中包含的Redis命令的伪客户端，并将这个伪客户端关联在服务器状态结构的lua_client属性中。lua_client伪客户端在服务器运行的整个生命期中一直存在，
只有服务器被关闭时，这个客户端才会被关闭。

##### AOF文件的伪客户端
服务器在载入AOF文件时，会创建用于执行AOF文件包含的Redis命令的伪客户端，并在载入完成之后，关闭这个伪客户端。






