#### 设置键的生存时间或过期时间
通过EXPIRE命令或者PEXPIRE命令，客户端可以以秒或者毫秒精度为数据库中的某个键设置生存时间(Time To Live,TTL),在经过指定的秒数或者毫秒数之后，服务器就会自动删除生存时间为0的键。EXPIREAT命令或者PEXPIREAT命令，以秒或者毫秒精度给数据库中的某个键设置过期时间(expire time)。

TTL命令和PTTL命令接受一个带有生存时间或者过期时间的键，返回这个键的剩余生存时间。

虽然有多重不同单位和格式的设置命令，但实际上EXPIRE、PEXPIRE、EXPIREAT 三个命令都是使用PEXPIREAT命令来实现的，最终都是调用的PEXPIREAT命令对应的方法实现的。

```
def EXPIRE(key, ttl_in_sec):
    #将ttl从秒转换成毫秒
    ttl_in_ms = sec_to_ms(ttl_in_sec)
    PEXPIRE(key, ttl_in_ms)
```
PEXPIRE命令又转换成PEXPIREAT命令：
```
def PEXPIRE(key, ttl_in_ms):
    #获取以毫秒计算的当前UNIX时间戳
    now_ms = get_current_unix_timestamp_in_ms()
    #当前时间加上TTL，得出毫秒格式的键过期时间
    PEXPIREAT(key, now_ms+ttl_in_ms)
    
    
def EXPIREAT(key, expire_time_in_sec):
    #将过期时间从秒转换为毫秒
    expire_time_in_ms = sec_to_ms(expire_time_in_sec)
    PEXPIREAT(key, expire_time_in_ms)
```
##### 保存过期时间
redisDb结构的expires字典保存了数据库中所有键的过期时间，为过期字典，键就是数据库键，值是long long类型，毫秒经度的unix时间戳（过期时间）。redisDb代码如上面所示，dict * expires 就是过期字典。图示如下：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-server-1-20180605.png)

```
def PEXPIREAT(key, expire_time_in_ms):
    if key not in redisDb.dict:
        return 0
    redisDb.expires[key] = expire_time_in_ms
    #过期时间设置成功
    return 1
```
##### 移除过期时间
PERSIST 命令可以移除一个键的过期时间：
```
redis-> PEXPIREAT msg 1391234400000
(integer) 1
redis-> TTM msg
(integer) 13893281
redis-> PERSIST msg
(integer) 1
redis-> TTL msg
(integer) -1
```
代码如下：
```
def PERSIST(key):
    if key not in redisDb.expires:
        return 0
    redisDb.expires.remove(key)
    return 1
```
##### 计算并返回剩余生存时间
TTL 命令以秒为单位返回键的剩余生存是时间，而PTTL命令则以毫秒为单位返回键的剩余生存时间。伪代码如下:
```
def PTTL(key):
    if key not in redisDb.dict:
        return -2
    expire_time_in_ms = redisDb.expires.get(key)
    if expire_time_in_ms is None:
        return -1
    now_ms = get_current_unix_timestamp_in_ms()
    return (expire_time_in_ms - now_ms)

def TTL(key):
    ttl_in_ms = PTTL(key)
    if ttl_in_ms < 0:
        return ttl_in_ms
    else:
        return ms_to_sec(ttl_in_ms)
```
##### 过期键的判断
通过过期字典，程序可以用一下步骤检查一个给定键是否过期：

1. 检查给定键是否存在于过期字典，如果存在，那么取得键的过期时间
2. 检查当前unix时间戳是否大于键的过期时间，如果是的话，那么键已经过期，否则的话，键未过期

```
def is_expired(key):
    expire_time_in_ms = redisDb.expires.get(key)
    if expire_time_in_ms is None:
        retrurn False
    now_ms = get_current_unix_timestamp_in_ms()
    if now_ms > expire_time_in_ms:
        return True
    else:
        return False
```
#### 过期键删除策略

+ 定时删除：创建一个定时器， 让定时器在键的到期时间来临时，立即执行对键的删除操作
+ 惰性删除：放置键过期不管，每次从键空间获取键时都检查键是否过期，如果过期删除，没过期返回该键
+ 定期删除：每隔一段时间，程序对数据库进行一次检查，删除里面的过期键，至于删多少，以及检查多少个数据库，有算法决定

##### 定时删除
定时删除是对内存最友好的，通过定时器，可以保证过期键会尽可能快的被删除，并释放占用的内存，但是它对CPU时间是不友好的，在过期键比较多的情况下，删除过期键这一行为可能会占用相当一部分CPU时间，如果内存不紧张但是CPU时间紧张，无疑会影响服务器响应时间和吞吐量。

##### 惰性删除
惰性删除对CPU时间是最友好的，但是对内存很不友好。如果数据库中有非常多的过期键，这些键又恰巧没有被访问的话，那么他们永远也不会被删除，甚至可以看做为内存泄漏，无用的垃圾数据占用了大量的内存。

##### 定期删除
定期删除是前两种策略的整合和折中：
定期删除每隔一段时间执行一次删除过期键操作，并限制删除操作执行时长和频率来减少删除操作对CPU时间的影响，从而有效减少过期键带来的内存浪费。

#### Redis的过期键删除策略
Redis实际上是定期删除和惰性删除两种策略的配合使用。

##### 惰性删除策略的实现
惰性删除由db.c/expireIfNeeded函数实现，Redis命令执行之前都会调用expireIfNeeded函数对输入键进行检查，它就像一个过滤器，它可以在命令真正执行前，过滤掉过期的输入键，从而避免命令接触到过期键。如果被删除了之后，命令按照键不存在的情况执行。
读写过程如下图所示：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-server-2-20180606.png)

```
int expireIfNeeded(redisDb *db, robj *key) {
    mstime_t when = getExpire(db,key);
    mstime_t now;

    if (when < 0) return 0; /* No expire for this key */

    /* Don't expire anything while loading. It will be done later. */
    if (server.loading) return 0;

    /* If we are in the context of a Lua script, we pretend that time is
     * blocked to when the Lua script started. This way a key can expire
     * only the first time it is accessed and not in the middle of the
     * script execution, making propagation to slaves / AOF consistent.
     * See issue #1525 on Github for more information. */
    now = server.lua_caller ? server.lua_time_start : mstime();

    /* If we are running in the context of a slave, return ASAP:
     * the slave key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys.
     *
     * Still we try to return the right information to the caller,
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time. */
    if (server.masterhost != NULL) return now > when;

    /* Return when this key has not expired */
    if (now <= when) return 0;

    /* Delete the key */
    server.stat_expiredkeys++;
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    return server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) :
                                         dbSyncDelete(db,key);
}
```
源码中判断如果执行的是lua脚本，则now是以lua开始执行的时间。如果当前服务器是slave，则不进行操作，等待master发送DEL命令。

##### 定期删除策略的实现
过期键的定期删除由redis.C/activeExpireCycle函数实现，每当redis的服务器周期性操作redis.c/serverCron函数执行时，activeExpireCycle函数就会被调用，它在规定的时间内，分多次遍历服务器中的各个数据库，从数据库的expires字典中随机检查一部分键的过期时间，并删除过期键。
源码较长，这里用伪代码描述：
```
DEFAULT_DB_NUMBERS = 16    #默认每次检查的数据库数量
DEFAULT_KEY_NUMBERS = 20   #默认每个数据库检查的键数量
current_db = 0             #全局变量，记录检查进度

def activeExpireCycle():
    if server.dbnum < DEFAULT_DB_NUMBERS：
        db_numbers = server.dbnum
    else:
        db_numbers = DEFAULT_DB_NUMBERS
    
    for i in range(db_numbers):
        #如果current_db的值等于服务器的数据库数量，重置为0
        if current_db == server.dbnum:
            current_db = 0
        redisDb = sever.db[current_db]
        current_db++
        
        for j in range(DEFAULT_KEY_NUMBERS):
            if redisDb.expires.size() == 0: break
            key_with_ttl = redisDb.expires.get_random_key()
            if is_expired(key_with_ttl):
                delete_key(key_with_ttl)
            if reach_time_limit(): return
```
伪代码过程已经很清晰，不多做赘述

#### AOF、RDB和复制功能对过期键的处理

##### RDB文件
在执行SAVE命令或者BGSAVE命令创建一个新的RDB文件时，程序会对数据库中的键进行检查，已过期的键不会被保存到新创建的RDB文件中。在启动服务器时，如果服务器开启了RDB功能，那么服务器对RDB文件进行载入，如果服务器以主服务器运行，过期键会被忽略；如果以从服务器运行，全都会被载入，在主从服务器进行数据同步的时候，从服务器的数据库就会被清空。

##### AOF文件写入
当服务器以AOF持久化模式运行时，如果数据库的某个键已经过期，但它还没有被惰性删除或者定期删除，那么AOF文件不会因为这个过期键而产生任何影响。当过期键被惰性删除或者定期删除之后，程序会向AOF文件追加一条DEL命令，来显式地记录该键已被删除。和生成RDB文件时类似，在执行AOF重写的过程中，程序会对数据库中的键进行检查，已过期的键不会被保存到重写后的AOF文件中。

##### 复制
当服务器运行在复制模式下，从服务器的过期键删除动作由主服务器控制：

+ 主服务器在删除一个过期键之后，会显式的向所有从服务器发送一个DEL命令，告知从服务器删除这个过期键。
+ 从服务器在执行客户端发送的读命令时，即使碰到过期键也不会将过期键删除，而是继续像处理未过期的键一样来处理过期键。
+ 从服务器只有在接到主服务器发来的DEL命令之后，才会删除过期键。

通过主服务器来控制从服务器统一的删除过期键，可以保证主从服务器数据的一致性，也正是因为这个原因，当一个过期键仍然存在于主服务器的数据库时，这个时期键在从服务器里的复制品也会继续存在。

#### 数据库通知
数据库通知是redis2.8加入的功能，这个功能可以让客户端通过订阅给定的频道或者模式，来获知数据库中键的变化，以及数据库中命令的执行情况。客户端获取0号数据库中针对message键执行的所有命令：
```
redis-> SUBSCRIBE __keyspace@0__:message
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "__keyspace@0__:message"
3) (integer) 1
1) "message"
2) "__keyspace@0__:message"
3) "set"
1) "message"
2) "__keyspace@0__:message"
3) "expire"
1) "message"
2) "__keyspace@0__:message"
3) "del"
```
根据返回的通知显示，先后共有set、expire、del三个操作对键message进行了操作。这一类关注某个键执行了什么命令的通知称为键空间通知（key-space notification），初此之外，还有键事件通知（key-event notification），他们关注什么命令被什么键执行了。

```
redis-> SUBSCRIBE __keyevent__@0:del
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "__keyevent@0__:del"
3) (integer) 1
1) "message"
2) "__keyevent@0__:del"
3) "hello"
1) "message"
2) "__keyevent@0__:del"
3) "msg"
1) "message"
2) "__keyevent@0__:del"
3) "redis"
```
根据发回的通知显示，hello、msg、redis三个键先后被DEL
服务器配置的notify-keyspace-events选项决定了服务器锁发送通知的类型

+ 想让服务器发送所有类型的键空间和键事件通知，可以将选项的值设置成AKE
+ 想让服务器发送所有类型的键空间知通知，可以将选项的值设置成AK
+ 想让服务器发送所有类型的键事件通知，可以将选项的值设置成AE
+ 想让服务器只发送和字符串键有关的键空间通知，可以将选项的值设置成K$
+ 想让服务器只发送和列表键有关的键事件通知，可以将选项的值设置成El

关于数据库通知的更多功能，以及配置的更多信息，请查阅官方文档。

##### 发送通知
发送数据库通知的功能是由notify.c/notifyKeyspaceEvent函数实现的：
```
void notifyKeyspaceEvent(int type, char *event, robj *key, int dbid);
```
函数的type参数是当前想要发送的通知类型，程序会根据这个值来判断通知是否就是服务器配置notify-keyspace-events选项所定的同志类型，从而决定是否发送通知。event、keys、dbid分别是事件的名称、产生事件的键、以及产生事件的数据库号码，函数会根据这几个参数构建时间通知的内容，以及接受通知的频道名。

当redis命令需要发送数据库通知时，该命令的实现函数就会调用notifyKeyspaceEvent函数并传递相关参数。

##### 发送通知的实现
```
/* The API provided to the rest of the Redis core is a simple function:
 *
 * notifyKeyspaceEvent(char *event, robj *key, int dbid);
 *
 * 'event' is a C string representing the event name.
 * 'key' is a Redis object representing the key name.
 * 'dbid' is the database ID where the key lives.  */
void notifyKeyspaceEvent(int type, char *event, robj *key, int dbid) {
    sds chan;
    robj *chanobj, *eventobj;
    int len = -1;
    char buf[24];

    /* If any modules are interested in events, notify the module system now. 
     * This bypasses the notifications configuration, but the module engine
     * will only call event subscribers if the event type matches the types
     * they are interested in. */
     moduleNotifyKeyspaceEvent(type, event, key, dbid);
    
    /* If notifications for this class of events are off, return ASAP. */
    //如果给定的通知不是服务器允许发送的通知，那么直接返回
    if (!(server.notify_keyspace_events & type)) return;

    eventobj = createStringObject(event,strlen(event));

    /* __keyspace@<db>__:<key> <event> notifications. */
    //发送键空间通知
    if (server.notify_keyspace_events & NOTIFY_KEYSPACE) {
        chan = sdsnewlen("__keyspace@",11);
        len = ll2string(buf,sizeof(buf),dbid);
        chan = sdscatlen(chan, buf, len);
        chan = sdscatlen(chan, "__:", 3);
        chan = sdscatsds(chan, key->ptr);
        //构建频道名后，构建频道对象
        chanobj = createObject(OBJ_STRING, chan);
        //发送通知
        pubsubPublishMessage(chanobj, eventobj);
        decrRefCount(chanobj);
    }

    /* __keyevent@<db>__:<event> <key> notifications. */
    //发送键事件通知
    if (server.notify_keyspace_events & NOTIFY_KEYEVENT) {
        chan = sdsnewlen("__keyevent@",11);
        if (len == -1) len = ll2string(buf,sizeof(buf),dbid);
        chan = sdscatlen(chan, buf, len);
        chan = sdscatlen(chan, "__:", 3);
        chan = sdscatsds(chan, eventobj->ptr);
        chanobj = createObject(OBJ_STRING, chan);
        pubsubPublishMessage(chanobj, key);
        decrRefCount(chanobj);
    }
    decrRefCount(eventobj);
}
```
pubsubPublishMessage就是PUBLISH命令的实现函数，执行这个函数等同于执行PUBLISh命令。