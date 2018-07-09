#### 服务器中的数据库
Redis服务器将所有的数据库都保存在服务器状态redis.h/redisServer结构的db数组中，db数组的每个项都是一个redis.h/redisDb结构，每个redisDb结构代表一个数据库。在初始化服务器时，程序会根据服务器状态的dbnum属性来决定创建 多少个数据库。
```
struct redisServer {
    redisDb *db;
    // ...
    int dbnum;                      /* Total number of configured DBs */
}
```
#### 切换数据库
每个redis客户端都有自己的目标数据库，每当客户端执行数据库写命令或者数据库读命令的时候，目标数据库就会成为这些命令的操作对象，默认使用0号数据库。
```
redis> set msg "Hello,world"
OK
redis> get msg
"Hello,world"
redis> select 1
OK
redis[1]> get msg
(nil)

```
在服务器内部，表示客户端状态的redisClient结构中的db属性记录了客户端当前的目标数据库，这个属性是一个指向redisDb的指针。

#### 数据库键空间
redis是一个键值对数据库服务器，服务器中每个数据库都由一个redis.h/redisDb结构表示，其中，redisDb结构的dict字典保存了数据库中的所有键值对，这个字典就叫做键空间。
```
/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;
```
键空间的键也就是数据库键，都是字符串对象，值可以是字符串对象、列表对象、哈希表对象、集合对象和有序集合对象中的任意一种。键的增加、删除、更新、读取都是对键空间的操作，不同的命令会执行不通的代码，最终都是在键空间上进行一系列操作。下图就是某一状态下的键空间：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-server-0-20180601.png)

##### 其他键空间操作
除了上面说的添加、删除、更新、取值操作，还有很多针对数据库本身的redis命令，也是通过键空间进行处理完成的。比如清空整个数据库的FLUSHDB、随机返回数据库中某个键的RANDOMKEY。还有用于返回数据库键数量的DBSIZE、类似的还有EXISTS、RENAME、KEYS，这些都是对键空间进行操作实现的。

##### 读写键空间时的维护操作

+ 在读取一个键之后，服务器会根据键是否存在来更新服务器的键空间命中(hit)次数或键空间不命中(miss)次数。这两个值可以在INFO stats 命令的 keyspace_hits 属性和 keyspace_misses 属性中查看。
+ 在读取一个键后，服务器会更新键的LRU时间，这个值可以用于计算键的闲置时间，使用OBJECT idletime <key>命令查看key的闲置时间。
+ 如果服务器在读取一个键时发现该键已经过期，那么服务器会先删除这个过期键，然后才执行余下的其他操作。
+ 如果有客户端使用watch命令监视了某个键，那么服务器在对被监视的键进行修改之后，会将这个键标记为脏(dirty),从而让事务程序注意到这个键已经被修改过。
+ 服务器每次修改一个键之后，都会对脏(dirty)键计数器的值增1，这个计数器会触发服务器的持久化以及复制操作。
+ 如果服务器开启了数据库通知功能，那么在键进行修改之后，服务器将按配置发送相应的数据库通知。

