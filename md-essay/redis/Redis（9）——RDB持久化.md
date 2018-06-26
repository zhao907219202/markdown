Redis是一个键值对数据库服务器，服务器中包含着任意个非空数据库，而每个非空数据库中又可以包含任意个键值对，为了方便，我们把服务器中的非空数据库以及它们的键值对统称为数据库状态。

因为redis是内存数据库，如果不想办法将存储在内存中的数据库状态保存到磁盘里面，那么一旦服务器进程退出，服务器中的数据库状态也会消失不见。为了解决这个问题，Redis提供了RDB持久化功能，这个功能可以将Redis在内存中的数据库状态保存到磁盘里面，避免数据意外丢失。RDB持久化可以手动执行，也可以根据服务器配置选项定期执行，该功能可以将某个时间点上的数据库状态保存到一个RDB文件中。

#### RDB文件的创建和载入
有两个Redis命令可以用于生成RDB文件，一个是SAVE，另一个是BGSAVE。
SAVE命令会阻塞Redis服务器进程，知道RDB文件创建完毕为止，在服务器进程阻塞期间，服务器不会处理任何命令请求。和SAVE命令直接阻塞服务器进程不同，BGSAVE命令会派生出一个子进程，然后由子进程负责创建RDB文件，服务器进程（父进程）继续处理命令请求。
```
redis-> SAVE             //等待直到RDB文件创建完毕
OK
redis-> BGSAVE           //派生子进程，并由子进程创建RDB文件
Background saving started
```
创建RDB文件的实际工作由rdb.c/rdbSave函数完成，SAVE 和 BGSAVE会以不同的方式调用这个函数。

RDB文件的载入工作是在服务器启动时自动执行的，所以Redis并没有专门用于载入RDB文件的命令，只要Redis服务器在启动时检测到RDB文件存在，它就会自动载入RDB文件。

需要注意的是，因为AOF文件的更新频率通常比RDB文件的更新频率高，所以如果服务器开启了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态；只有AOF功能关闭的时候，服务器才会使用RDB文件来还原数据库状态。载入RDB由rdb.c/rdbLoad函数完成。

数据库文件的载入图示如下：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-rdb-0-20180608.png)

##### SAVE/BGSAVE 命令执行时的服务器状态
SAVE命令执行时，服务器会被阻塞，客户端所有命令请求都会被拒绝。

BGSAVE命令执行期间，子进程创建RDB文件，Redis服务器仍然可以继续处理客户端的命令请求，但是服务器处理SAVE、BGSAVE、BGREWRITEAOF三个命令的方式会和平时有所不同。首先，在BGSAVE命令执行期间，客户端发送的SAVE命令会被服务器拒绝，服务器禁止SAVE命令和BGSAVE命令同时执行是为了避免父进程（服务器进程）和子进程同时执行两个rdbSave调用，防止产生竞争条件。其次，在BGSAVE命令执行期间，客户端发送的BGSAVE命令会被服务器拒绝，因为同时执行两个BGSAVE命令也会产生竞争条件。最后，BGREWRITEAOF 和 BGSAVE另个命令不能同时执行，

+ 如果BGSAVE 命令正在执行，那么客户端发送的BGREWRITEAOF 命令会被延迟到BGSAVE命令执行完毕之后执行
+ 如果BGREWRITEAOF 命令正在执行，那么客户端发送的BGSAVE 命令会被服务器拒绝。

因为BGSAVE 和 BGREWRITEAOF 两个命令的实际工作都由子进程执行，所以这两个命令在操作方面并没有冲突的地方，不能同时执行只是一个性能方面的考虑，并发出两个进程，并且都执行大量的磁盘写入操作，对性能影响比较大。

##### RDB 文件载入时的服务器状态
服务器载入 RDB 文件时，会一直处于阻塞状态，直到载入工作完成为止。

#### 自动间隔性保存
因为BGSAVE命令可以在不阻塞服务器进程的情况下执行，所以Redis允许用户通过设置服务器配置的save选项，让服务器每隔一段时间自动执行一次BGSAVE命令。用户可以通过save选项设置多个保存条件，但只要其中任意一个被满足，服务器就会执行BGSAVE命令。
```
save 900 1
save 300 10
save 60 10000
```
如果像上面这样配置，那么只要满足下面三个条件的任意一个，BGSAVE就会被执行：

1. 服务器在900秒之内，对数据库进行了1次修改
2. 服务器在300秒之内，对数据库进行了10次修改
3, 服务器在60秒之内，对数据库进行了至少10000次修改

##### 设置保存条件
当redis服务器启动时，用户可以通过指定配置文件或者传入启动参数的方式设置save选项，如果用户没有主动设置save选项，那么服务器会为save选项设置默认条件。

接着，服务器程序会根据save选项所设置的保存条件，设置服务器状态rediServer结构的saveparams属性：
```
struct redisServer {
    /* RDB persistence */
    long long dirty;                /* Changes to DB from the last save */
    long long dirty_before_bgsave;  /* Used to restore dirty on failed BGSAVE */
    pid_t rdb_child_pid;            /* PID of RDB saving child */
    struct saveparam *saveparams;   /* Save points array for RDB */
    int saveparamslen;              /* Number of saving points */
    char *rdb_filename;             /* Name of RDB file */
    int rdb_compression;            /* Use compression in RDB? */
    int rdb_checksum;               /* Use RDB checksum? */
    time_t lastsave;                /* Unix time of last successful save */
    time_t lastbgsave_try;          /* Unix time of last attempted bgsave */
    time_t rdb_save_time_last;      /* Time used by last RDB save run. */
    time_t rdb_save_time_start;     /* Current RDB save start time. */
    int rdb_bgsave_scheduled;       /* BGSAVE when possible if true. */
    int rdb_child_type;             /* Type of save by active child. */
    int lastbgsave_status;          /* C_OK or C_ERR */
    int stop_writes_on_bgsave_err;  /* Don't allow writes if can't BGSAVE */
    int rdb_pipe_write_result_to_parent; /* RDB pipes used to return the state */
    int rdb_pipe_read_result_from_child; /* of each slave in diskless SYNC. */
};
```
该属性是一个数组，数组中的每个元素都是一个saveparam结构，每个saveparam结构都保存了save选项设置的保存条件：
```
struct saveparam {
    time_t seconds;
    int changes;
}
```
##### dirty计数器和lastsave属性
除了saveparam数组之外，服务器状态还维持着一个dirty计数器，以及一个lastsave属性：

+ dirty计数器记录距离上一次成功执行SAVE命令或者BGSAVE命令之后，服务器对数据库状态（服务器中所有数据库）进行了多少次修改（包括写入、删除、更新等操作）。
+ lastsave属性是一个unix时间戳，记录了服务器上一次成功执行SAVE命令或者BGSAVE命令的时间。

当程序成功执行一个数据库修改命令后，程序就会对dirty计数器进行更新，命令改了多少次数据库，dirty计数器的值就会增加多少。

##### 检查保存条件是否满足
Redis的服务器周期性操作函数serverCron默认每隔100毫秒就会执行一次，该函数用于对正在运行的服务器进行维护，它的其中一项工作就是检查save选项所设置的保存条件是否已经满足，如果满足的化，就执行BGSAVE命令。

以下伪代码展示了serverCron函数检查保存条件的过程：
```
def serverCron():
    # ...
    # 遍历所有保存条件
    for saveparam in server.saveparams:
        save_interval = unixtime_now() - server.lastsave
        if server.dirty >= savaparam.changes and save_interval > saveparam.seconds:
            BGSAVE()
```
程序会遍历并检查saveparam数组中的所有保存条件，只要有任意一个满足条件，那么服务器就会执行BGSAVE命令。

#### RDB文件结构

REDIS| db_version|databases|EOF|check_sum
---|---|---|---|---

常量用全大写字符表示，变量和数据用全小写字母表示。RDB文件最开头是REDIS部分，这个部分的长度为5字节，保存着"REDIS"五个字符。程序通过这个5个字符，
检查该文件为RDB文件。db_version长度为4字节，它的值是一个字符串表示的整数，记录RDB文件的版本号。databases包含着零个或多个数据库，以及各个数据库中的键值对数据。
+ 如果服务器状态为空，那么这个部分也为空，长度0字节
+ 如果服务器状态不为空，那么这个部分根据数据库所保存的键值对数量、类型和内容不同，长度也有所不同

EOF为常量，一个字节，标志RDB文件正文内容的结束。check_sum是一个8字节长的无符号整数，保存着一个校验和，通过前面四部分内容计算得出，在载入RDb文件时，会将载入数据的检验和和RDB文件中记录
的校验和进行比对，以此来检查RDB文件是否有出错或者损坏的情况。

##### databases部分
一个RDB文件的databases部分可以保存任意多个非空数据库。每个非空数据库在RDB文件中都可以保存为 SELECTDB、db_number、key_value_pairs 三个部分。

SELECTDB| db_number|key_value_pairs
---|---|---

SELECTDB常量的长度为1字节，当读入程序遇到这个值的时候，它知道接下来要读入的将是一个数据库号码。db_number保存着一个数据库号码，根据号码的大小不同，这个部分的长度可以是1字节、2字节或者5字节。
当程序读入db_number部分之后，服务器会调用SELECT命令，根据读入的数据库号码进行数据库切换，使得读入的键值对可以载入到正确的数据库中。key_value_pairs部分保存了数据库中所有键值对数据，如果
键值对带有过期时间，那么过期时间也会和键值对保存在一起。根据键值对的数量、类型、内容以及是否有过期时间，这个部分的长度有所不同。

##### key_value_pairs部分

TYPE|key|value
---|---|---

不带过期时间由上面所示的三部分组成，带过期时间的如下：

EXPIRETIME_MS|ms|TYPE|key|value
---|---|---|---|---

EXPIRETIME_MS 为长度1字节的常量，告知读入程序，下面是以毫秒为单位的过期时间。ms是一个8字节的带符号整数，毫秒为单位的Unix时间戳。TYPE 长度1字节，是 REDIS_RDB_TYPE_* 中的一种，*可以为
STRING、LIST、SET、ZSET、HASH、LIST_ZIPLIST、SET_INTSET、ZSET_ZIPLIST、HASH_ZIPLIST。每个type常量都代表了对象类型或者底层编码，当服务器读入RDB文件中的键值对数据时，程序会根据TYPE的值来
决定如何读入和解释value的数据。key总是字符串对象，value则是具体的值对象。

##### value 部分
具体value对应的各个对象的 RDB 文件内容结构这里不展开介绍，有兴趣的可以查看redis官方手册。其中key/value的生成的RDB文件可以通过redis配置选项：rdbcompression yes/no 控制 RDB 文件是否使用LZF算法
进行压缩，主要是对字符串对象压缩。

#### 分析RDB文件
Redis自身带有RDB文件检查攻速 redis-check-dump，网上也能找到很多处理RDB文件的工具，所以人工分析RDB文件的意义不大，工作学习中如果有这样的需求，可以直接使用工具分析。













