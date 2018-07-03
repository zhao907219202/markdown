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

